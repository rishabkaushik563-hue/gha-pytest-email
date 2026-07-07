# GHA Pytest Email Demo

Runs pytest on every push via GitHub Actions.
Uploads test log as artifact on every run.
Sends an email alert only when tests fail.

## Local run
```
pip install -r requirements.txt
pytest tests/ -v
```

## GitHub Secrets required
| Secret | Value |
|---|---|
| EMAIL_USERNAME | Gmail address |
| EMAIL_PASSWORD | Gmail App Password (16 chars) |
| EMAIL_TO | Recipient email address |
