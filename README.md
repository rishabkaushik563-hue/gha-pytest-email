---

## Task 1: Pytest CI with Email Alerts

### Workflow Runs
- ✅ Passing run: `https://github.com/rishabkaushik563-hue/gha-pytest-email/actions/runs/28844939139`
- ❌ Failing run: `<https://github.com/rishabkaushik563-hue/gha-pytest-email/actions/runs/28844939139`



### How the Alert Works
1. On every push, GitHub Actions runs `pytest tests/ -v` and pipes output to `test-log.txt`.
2. The log file is uploaded as an artifact on every run (pass or fail), using `if: always()`.
3. A separate step (`Check test result`) greps `test-log.txt` for `FAILED` or `ERROR` and sets an environment variable `tests_failed=true/false` via `$GITHUB_ENV`.
4. The email step runs conditionally: `if: env.tests_failed == 'true'` — using the `dawidd6/action-send-mail` action over Gmail SMTP (port 465, App Password auth).
5. On success, the email step is skipped; on failure, an email is sent with the log file attached.

### Problems Hit and Fixes

| Problem | Cause | Fix |
|---|---|---|
| `fatal: unable to auto-detect email address` on commit | Git had no configured identity on this machine | Ran `git config --global user.email` and `user.name` once |
| `fatal: ... returned error: 400` on push | Remote URL literally included `< >` placeholder brackets from copied command | Removed and re-added remote with the actual username, no brackets |
| `remote: Invalid username or token. Password authentication is not supported` | GitHub removed password auth for git operations | Switched to a GitHub Personal Access Token used in place of the password, then later to `gh auth login` for persistent auth |
| `gh : term not recognized` right after installing GitHub CLI | PATH not refreshed in the already-open PowerShell session | Closed and reopened PowerShell so it picked up the updated PATH |
| `remote: Repository not found` (404) on push | Remote URL pointed to a repo name/account that didn't exactly match what existed on GitHub | Verified exact repo name and username on github.com, then corrected with `git remote set-url origin ...` |
| `nothing to commit, working tree clean` after editing a test to break it | File edit wasn't saved in the editor before running `git status` | Reopened the file, made the edit, explicitly saved with `Ctrl+S`, confirmed via `git status` before committing |

### Verification Steps
- Successful run: all 5 tests `PASSED`, artifact `pytest-log` uploaded, "Send failure email" step shown as **skipped**, no email received.
- Failed run: 1 test `FAILED`, artifact still uploaded, "Send failure email" step executed, email received in inbox with log attached.
- Reverted the intentional break via `git revert HEAD` to restore a clean passing state for final submission.

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