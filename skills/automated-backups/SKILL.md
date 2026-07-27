---
name: automated-backups
description: "Schedule git-based backups to GitHub via Hermes cron."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [git, backup, cron, github, automation, scripting]
    related_skills: [github-repo-management, github-auth, hermes-agent]
---

# Automated Backups to GitHub

Schedule recurring data-persistence backups to a GitHub repository using git HTTPS auth and Hermes cron jobs. Covers the full setup: script authoring, cron scheduling, and recovering from push-protection blocks.

## When to Use This Skill

- User asks to "backup everything here every N hours/days" to GitHub
- Need a script-only cron job (zero token consumption) that pushes file snapshots
- Environment has port 22 blocked (SSH unavailable) — must use HTTPS auth

## Step-by-Step

### 1. Clone (or create) the backup repo

```bash
git clone https://github.com/<user>/<repo>.git /data/backup-tmp/<repo>
cd /data/backup-tmp/<repo>
git config user.email "backup@example.com"
git config user.name "Automated Backup"
```

**Port-22 workaround:** Use HTTPS remote URL with the PAT embedded in the URL (never SSH):

```bash
git remote set-url origin https://<user>:<token>@github.com/<user>/<repo>.git
```

### 2. Write the backup script

Place scripts at `~/.hermes/scripts/<name>.sh`.

Key pattern for a push-only script that owns the repo:

```bash
#!/bin/bash
set -e

REPO_DIR="/data/backup-tmp/myrepo"
REPO_URL="https://<user>:<token>@github.com/<user>/<repo>.git"

# Clone or reset to match remote
if [ ! -d "$REPO_DIR/.git" ]; then
  git clone "$REPO_URL" "$REPO_DIR"
else
  cd "$REPO_DIR"
  git fetch origin
  git checkout main
  git reset --hard origin/main   # safe because we own this repo
fi

# Copy files (credentials excluded via .gitignore)
cp -f /data/.hermes/config.yaml .
cp -rf /data/.hermes/skills .
# ... etc

# Commit and push
git add -A
if ! git diff --cached --quiet; then
  git commit -m "Auto-backup $(date -u '+%Y-%m-%d %H:%M UTC')"
  git push origin main
fi
```

### 3. Exclude sensitive files

Strict `.gitignore` — must be in the repo before the first commit:

```
state.db  auth.json  .env  sessions/
*.lock  .cache/  __pycache__/  *.pyc
logs/  audio_cache/  image_cache/  .pairing/
provider_models_cache.json  models_dev_cache.json
ollama_cloud_models_cache.json
gateway_state.json  channel_directory.json
```

### 4. Schedule via Hermes cron

Use `no_agent=true` so the script runs directly (zero LLM tokens):

```python
cronjob(
  action="create",
  name="my-backup",
  schedule="0 */12 * * *",       # every 12 hours
  script="my-backup-script.sh",  # relative to ~/.hermes/scripts/
  no_agent=True
)
```

## Handling Push Protection Blocks

GitHub's secret scanning may reject a push if a credential file (state.db, auth.json) was accidentally committed. **Do NOT bypass the block URL** — rewrite history:

```bash
git rm --cached state.db auth.json
cat >> .gitignore # add the excludes above
git add -A
git checkout --orphan clean-branch
git add -A
git commit -m "Fresh start — no secrets"
git branch -D main
git branch -m clean-branch main
git push -f origin main
```

## Silent-on-Success / Error-Only Delivery

When the user prefers not to be notified on every backup, configure the script to produce **zero stdout on success** and **exit 1 on failure**. With `no_agent=True`, empty stdout is never delivered, and a non-zero exit triggers an error alert.

```bash
# Silent on success:
git add -A
if ! git diff --cached --quiet; then
  git commit -m "Auto-backup $(date)"
  git push origin main 2>&1 || { echo "PUSH FAILED"; exit 1; }
fi
# No stdout = no delivery (success or no-changes both produce nothing)
```

Use `echo "ERROR MSG"; exit 1` rather than bare `set -e` for failure reporting. A bare `set -e` with push failure may emit only stderr, which the cron scheduler doesn't capture as a delivery payload.

Ask the user about notification preference early (every-backup pings vs. error-only alerts). The silent-on-success pattern is a one-line shift once the preference is known.

## Pitfalls

- **Sensitive files can sneak into the first commit** — use `git diff --cached` to check for `ghp_` tokens before pushing. Add strict .gitignore before the initial `git add -A`.
- **Non-fast-forward rejections** — when history diverges, use `git reset --hard origin/main` at the START of each script run to keep local tracking aligned.
- **Scripts use relative path** — the cronjob system requires script paths relative to `~/.hermes/scripts/`. The script body itself uses absolute paths.
- **Git identity needed** — `git config user.name` and `user.email` must be set per-repo or commits fail.
- **Large initial push** — Hermes skills can be ~30MB. Create the empty GitHub repo first.
