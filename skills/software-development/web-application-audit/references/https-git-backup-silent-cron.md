# HTTPS Git Backup with Silent Cron

Pattern: git push to GitHub over HTTPS (bypassing blocked port 22) via a
self-contained shell script running as a `no_agent=true` cron job that stays
silent on success.

## When to Use

- Port 22 (SSH) is blocked in the current environment
- You need recurring backups of config files to a GitHub repo
- You want zero-noise delivery — only hear about failures

## Recipe

### 1. Auth over HTTPS

Embed the token in the remote URL (this script approach only):
```
REPO_URL="https://<user>:<token>@github.com/<user>/<repo>.git"
```

Token must have `repo` scope. Never commit this URL — pass via env variable
or embed in the script file (which stays out of git).

### 2. Reset-based sync strategy

Since the backup script owns the repo, use hard reset so history always
matches origin/main:

```
git fetch origin
git checkout main
git reset --hard origin/main
```

This avoids non-fast-forward issues and keeps the backup in sync even if
the remote was manually modified.

### 3. .gitignore for sensitive files

```
state.db
auth.json
.env
sessions/
*.lock
.cache/
__pycache__/
logs/
```

### 4. Silent success with `no_agent=true`

In the cronjob, set:
- `no_agent=true` — runs the script directly, no LLM token cost
- `script="hermes-backup.sh"` — relative to ~/.hermes/scripts/
- If the script produces **no stdout on success**, nothing is delivered.
- Non-zero exit = error alert delivered to the user automatically.

### 5. Script pattern (minimal)

```bash
#!/bin/bash
set -e
REPO_DIR="/data/backup-tmp/hermesbackup"
REPO_URL="https://user:token@github.com/user/repo.git"

mkdir -p /data/backup-tmp
if [ ! -d "$REPO_DIR/.git" ]; then
  git clone "$REPO_URL" "$REPO_DIR"
  cd "$REPO_DIR"
else
  cd "$REPO_DIR"
  git fetch origin
  git checkout main
  git reset --hard origin/main
fi

# Update .gitignore, copy files, commit
# ...
git add -A
if ! git diff --cached --quiet; then
  git commit -m "Auto-backup $(date -u '+%Y-%m-%d %H:%M UTC')"
  git push origin main
  # No echo = silent success
fi
# No output = nothing happened or success
```

### 6. Cronjob creation

```
cronjob action=create name="hermes-backup" script="hermes-backup.sh" \
  no_agent=true schedule="0 */12 * * *"
```

## Pitfalls

- **GitHub push protection** may block commits containing secrets. If your
  backup includes `state.db` or `auth.json`, GitHub's secret scanner will
  flag them. Keep these files in `.gitignore`.
- **Fresh clone is slow** for large repos. The script clones once, then pulls.
- **Hard reset discards local changes.** This is correct for a backup repo
  where the remote is the source of truth.
- **Script must be in ~/.hermes/scripts/.** The cron tool won't accept absolute
  paths for scripts; use just the filename.
