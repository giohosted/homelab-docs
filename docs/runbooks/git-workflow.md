# Git Workflow — homelab-infra

**Applies to:** docker-prod-01 (`/opt/stacks`)  
**Repo:** homelab-infra (private, GitHub)  
**Last Updated:** 2026-03-13

---

## Overview

`/opt/stacks` on docker-prod-01 is a clone of the `homelab-infra` GitHub repo. Every compose file lives here. Changes are committed and pushed to GitHub to keep the repo as source of truth.

`/opt/appdata` is NOT in git — it contains container state and secrets. It is gitignored and backed up nightly to NAS via rsync (Phase 5).

---

## Daily Workflow

After creating or editing any compose file:
```bash
cd /opt/stacks
git add .
git commit -m "brief description of what changed"
git push
```

### Good commit message examples
- `add traefik compose`
- `add arr stack compose`
- `update sonarr-tv restart policy`
- `add gluetun killswitch to torrent stack`

---

## Adding a New Stack
```bash
# Create the stack directory and compose file
mkdir -p /opt/stacks/<stack-name>
nano /opt/stacks/<stack-name>/compose.yaml

# Create appdata directory (not in git)
mkdir -p /opt/appdata/<app-name>

# Commit the compose file
cd /opt/stacks
git add .
git commit -m "add <stack-name> compose"
git push
```

---

## Checking Status
```bash
cd /opt/stacks

# See what's changed since last commit
git status

# See recent commit history
git log --oneline -10
```

---

## What Is and Isn't in Git

| Path | In Git | Notes |
|------|--------|-------|
| `/opt/stacks/*/compose.yaml` | ✅ Yes | Safe to commit — no secrets |
| `/opt/stacks/*/.env` | ❌ No | Gitignored — contains secrets |
| `/opt/appdata/` | ❌ No | Gitignored — container state, backed up to NAS |

---

## Credentials

- GitHub credentials cached via `git config --global credential.helper store`
- Token stored at `~/.git-credentials` on docker-prod-01
- If token expires, generate a new PAT at GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
- Scope needed: `repo` (full control of private repositories)

---

## Notes

- Always commit before making major changes — gives you a rollback point
- `.env` files go next to their `compose.yaml` but are never committed — use `.env.example` with placeholder values if you want to document required variables
- `/opt/appdata` backup via rsync to NAS is set up in Phase 5