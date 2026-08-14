# Transactional Double-Entry Wallet Engine

A reference implementation of a transactional wallet service built around PostgreSQL, SQLAlchemy, FastAPI, Redis, Celery, and RabbitMQ.

## Design

- `ledger_entries` is append-only. A transfer creates exactly one debit and one credit row with the same `transaction_id`.
- Wallet balances are calculated as `SUM(credits) - SUM(debits)`; there is no mutable balance column.
- Transfers run in one PostgreSQL transaction and lock both wallets with `SELECT ... FOR UPDATE` in ascending UUID order.
- `Idempotency-Key` is required for `POST /transfers`. Redis uses `SET NX EX` for the claim and stores the completed HTTP response.
- Inserting a committed `wallet_transfers` row fires a PostgreSQL trigger that creates one pending outbox event in the same transaction.
- Celery polls the outbox with `FOR UPDATE SKIP LOCKED`, dispatches the event, and marks it dispatched or failed.

## Run locally with Docker

```powershell
docker compose up --build
```

The API is available at `http://localhost:8000`; OpenAPI docs are at `http://localhost:8000/docs`.
Prometheus is at `http://localhost:9090`, and Jaeger tracing is at `http://localhost:16686`.

The local bootstrap user is `wallet-demo` / `wallet-demo-password`. Change these values before sharing the environment.

Authenticate through Swagger's `Authorize` button, or request a token:

```powershell
Invoke-RestMethod -Method Post http://localhost:8000/auth/token `
  -ContentType 'application/x-www-form-urlencoded' `
  -Body 'username=wallet-demo&password=wallet-demo-password'
```

Create two wallets and transfer funds. For PowerShell API calls, first obtain a bearer token and add it to every protected request:

```powershell
$token = (Invoke-RestMethod -Method Post http://localhost:8000/auth/token `
  -ContentType 'application/x-www-form-urlencoded' `
  -Body 'username=wallet-demo&password=wallet-demo-password').access_token
$auth = @{ Authorization = "Bearer $token" }
$a = Invoke-RestMethod -Method Post http://localhost:8000/wallets -Headers $auth -ContentType 'application/json' -Body '{"currency":"USD"}'
$b = Invoke-RestMethod -Method Post http://localhost:8000/wallets -Headers $auth -ContentType 'application/json' -Body '{"currency":"USD"}'
Invoke-RestMethod -Method Post "http://localhost:8000/admin/wallets/$($a.id)/credit?amount=100" -Headers $auth
$transferHeaders = @{ Authorization = "Bearer $token"; "Idempotency-Key" = "demo-1" }
Invoke-RestMethod -Method Post http://localhost:8000/transfers -Headers $transferHeaders -ContentType 'application/json' -Body (ConvertTo-Json @{source_wallet_id=$a.id; destination_wallet_id=$b.id; amount="10.00"; currency="USD"})
```

The local stack already includes demo JWT authentication, wallet ownership checks, Redis rate limiting, signed webhook delivery, Prometheus metrics, and Jaeger tracing. For a production deployment, replace local defaults with managed services, TLS/HTTPS, secret management, backups, alerting, a real identity model, and a hardened external webhook/notification provider.

## Tests and load testing

Metrics are exposed at `http://localhost:8000/metrics`. Set `OTEL_EXPORTER_OTLP_ENDPOINT` to export OpenTelemetry traces to Jaeger, Grafana Tempo, or another OTLP collector.

Install the development dependencies and run unit/integration tests against a PostgreSQL and Redis instance:

```powershell
pip install -r requirements.txt
pytest -q
```

The included Locust scenario can create two wallets, fund the source through the admin seed endpoint, and fire transfers at one source wallet:

```powershell
locust -f loadtest/locustfile.py --host http://localhost:8000 -u 500 -r 500 --run-time 30s --headless
```

To save timestamped CSV and JSON results locally on Windows:

```powershell
.\loadtest\run.ps1
```

CI runs the same 500-user test and uploads the recorded results as a workflow artifact.

It reports failed requests and checks the source balance after the run. The test intentionally uses unique idempotency keys per request; retry behavior is covered by `tests/test_idempotency.py`.

## Environment

See `.env.example`. The Docker defaults are suitable only for local development.

## GitHub repository hygiene

The included `.gitignore` excludes local secrets, generated PDFs, temporary render files, test reports, and Python caches. Commit `.env.example` as a safe configuration template, but never commit your local `.env` file.
