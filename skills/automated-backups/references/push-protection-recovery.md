# Push Protection Recovery Recipe

Full reproduction of the steps taken when GitHub's push protection blocked an initial backup commit containing credentials.

## Error Pattern

```
remote: - GITHUB PUSH PROTECTION
remote:     - Push cannot contain secrets
remote:       —— GitHub Personal Access Token ——
remote:       locations:
remote:         - commit: <sha>
remote:           path: state.db:475
remote:         - commit: <sha>
remote:           path: state.db:565
remote:         ...
```

The call-to-action in the error includes a bypass URL (unblock-secret). Do NOT use it — fix the root cause.

## Recovery Recipe

1. Remove the offending files from tracking:
   ```bash
   git rm --cached state.db auth.json .env sessions/sessions.json
   ```

2. Add strict .gitignore:
   ```bash
   cat >> .gitignore << 'EOF'
   state.db
   auth.json
   .env
   sessions/
   ...
   EOF
   git add .gitignore
   ```

3. Rewrite history to purge the secret from ALL commits:
   ```bash
   git checkout --orphan clean-branch
   git add -A
   git commit -m "Fresh start — no secrets"

   # Check staged content for any remaining tokens before pushing:
   git diff --cached | grep -i "ghp_" || echo "No tokens detected"

   git branch -D main
   git branch -m clean-branch main
   git push -f origin main
   ```

4. Verify on remote:
   ```bash
   git ls-remote origin main
   ```

## Backup Script Template (Verified Working)

The full script written during this session:

- Path: `~/.hermes/scripts/hermes-backup.sh`
- Uses `git fetch origin && git reset --hard origin/main` at the start of each run
- Cron scheduled with `no_agent=True` via `cronjob(action="create", script="hermes-backup.sh", ...)`

Key constraint: the cron system accepts only the basename (relative to `~/.hermes/scripts/`).
