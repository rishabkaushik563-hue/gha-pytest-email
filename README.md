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


---

## Task 2: Docker Build + Push to Amazon ECR (branch: task2)

### Overview
On every push to `task2`, GitHub Actions builds a Docker image of the pytest app,
tags it with the Git commit SHA, and pushes it to a private Amazon ECR repository.
Authentication uses GitHub OIDC — no AWS access keys are stored anywhere.

### Files
- `Dockerfile` — builds a `python:3.12-slim` image, installs dependencies, copies app + tests
- `.dockerignore` — excludes `.git/`, `.github/`, caches, and docs from the build context
- `.github/workflows/docker-ecr.yml` — build, cache, push workflow

### Caching Approach
Uses Docker Buildx with the `type=gha` cache backend (`cache-from`/`cache-to: type=gha`),
which stores layer cache in GitHub's own Actions cache service (scoped per repo, 10GB limit).

### Cached Layers
Per the Dockerfile layer order:
1. Base image (`python:3.12-slim`) — rarely invalidated
2. `WORKDIR /app` — static, always cached
3. `COPY requirements.txt .` — cached unless dependencies change
4. `RUN pip install ...` — cached unless `requirements.txt` changes (most expensive layer, biggest time saving)
5. `COPY app/`, `COPY tests/` — invalidated on every code change (expected, cheap to rebuild)

### How Cache Was Verified
1. Ran the workflow twice on `task2` with no code changes (second run via empty commit).
2. Compared build step duration: Run 1 = `15s`, Run 2 = `18s`.
3. Confirmed `CACHED` markers in the second run's build log for the `pip install` and `WORKDIR` layers.
4. Confirmed two distinct SHA-tagged images in ECR (`pytest-app` repository → Image tags).

### Workflow Runs
- Run 1 (initial build): `https://github.com/rishabkaushik563-hue/gha-pytest-email/actions/runs/28884425900`
- Run 2 (cached build): `https://github.com/rishabkaushik563-hue/gha-pytest-email/actions/runs/28884643382`

### Security
- No AWS access keys committed or stored as secrets.
- GitHub Actions authenticates via OIDC → assumes `github-actions-ecr-push` IAM role, scoped to this repo + `task2` branch only.
- IAM policy is least-privilege: scoped to `ecr:*` push actions on the single `pytest-app` repository ARN.