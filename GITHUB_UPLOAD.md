# Publish this project to GitHub

1. Create a new empty repository on GitHub. Do not add GitHub's generated README, `.gitignore`, or license because this project already has its own files.
2. In PowerShell, open this project folder and run:

```powershell
git init
git add .
git commit -m "Build transactional double-entry wallet engine"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPOSITORY.git
git push -u origin main
```

3. Replace `YOUR-USERNAME` and `YOUR-REPOSITORY` with the values from the GitHub repository page.

## Before publishing

- Confirm that `.env` does not appear in `git status`.
- Keep `.env.example`; it is the safe configuration template.
- Replace local passwords and secrets before sharing any deployed environment.
- Run `docker compose up --build` once locally to confirm the stack starts.

The GitHub Actions workflow runs style checks, tests, an integration stack, and the 500-user Locust load test on every push and pull request.
