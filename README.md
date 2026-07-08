---

## Task 1: Pytest CI with Email Alerts

### Workflow Runs
- ✅ Passing run: `https://github.com/rishabkaushik563-hue/gha-pytest-email/actions/runs/28844939139`
- ❌ Failing run: `https://github.com/rishabkaushik563-hue/gha-pytest-email/actions/runs/28846650457`



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