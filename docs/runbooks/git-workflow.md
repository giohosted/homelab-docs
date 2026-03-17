# Git Workflow — homelab-infra

**Applies to:** docker-prod-01, auth-prod-01, immich-prod-01 (`/opt/stacks`)
**Repo:** homelab-infra (private, GitHub)
**Last Updated:** 2026-03-16

---

## Overview

`/opt/stacks` on each host is a clone of the `homelab-infra` GitHub repo. Every compose file lives here. Changes are committed and pushed to GitHub to keep the repo as source of truth.

`/opt/appdata` is NOT in git — it contains container state and secrets. It is gitignored and backed up nightly to NAS via rsync (Phase 5).

The repo is cloned on multiple hosts:

| Host | Path | What lives there | Status |
|------|------|-----------------|--------|
| docker-prod-01 | /opt/stacks | traefik, arr, books, torrent, dockman, cloudflared, adguardhome-sync | Active |
| auth-prod-01 | /opt/stacks | authentik | Active |
| immich-prod-01 | /opt/stacks | immich | Active |

**GitHub is the source of truth — not any individual host.**

---

## The Golden Rule

**Always pull before you commit.** If you make changes on docker-prod-01 and forget to pull on auth-prod-01 before committing there, you risk pushing a stale file and overwriting the latest version on GitHub.

---

## Standard Workflow

Use this every time you make a change on any host:
```bash
# 1. Make sure you're in the repo root
cd /opt/stacks

# 2. Sync with GitHub first — always do this before anything else
git pull

# 3. Check exactly what has changed
git status

# 4. Stage your changes
git add .               # stages everything changed/new
# OR
git add <folder/file>   # stages only specific files if you have
                        # multiple changes and only want some

# 5. Commit with a clear message describing what changed
git commit -m "brief description of what changed"

# 6. Push to GitHub
git push
```

---

## When to use `git add .` vs `git add <path>`

| Situation | Command |
|-----------|---------|
| You only changed one thing and want to commit all of it | `git add .` |
| You changed multiple things but only want to commit some | `git add <specific folder or file>` |
| You're unsure what changed | Run `git status` first, then decide |

**When in doubt, always run `git status` before `git add` so you know exactly what you're about to commit.**

---

## Daily Workflow

After creating or editing any compose file:
```bash
cd /opt/stacks
git pull
git add .
git commit -m "brief description of what changed"
git push
```

### Good commit message examples
- `add traefik compose`
- `add authentik compose`
- `add arr stack compose`
- `add immich compose`
- `update sonarr-tv restart policy`
- `add gluetun killswitch to torrent stack`
- `fix immich compose - use official /data mount path`

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
git pull
git add .
git commit -m "add <stack-name> compose"
git push
```

---

## Checking What Changed
```bash
cd /opt/stacks

# See what files are new, modified, or deleted
git status

# See recent commit history
git log --oneline -10

# See exactly what changed in a specific file
git diff <file>
```

---

## Multi-Host Gotcha — Stale Files

Because the repo lives on multiple hosts, this scenario can happen:

1. You update `traefik/compose.yaml` on docker-prod-01 and push it
2. You SSH into auth-prod-01 and run `git add .` without pulling first
3. auth-prod-01 has the OLD `traefik/compose.yaml`
4. You just overwrote the updated version on GitHub

**How to avoid it:** Always run `git pull` first on whichever host you're working on before making or committing any changes.

---

## Per-Host Git Identity

Each host requires git identity to be configured before committing. Run once per host:
```bash
git config --global user.email "delgadogiovanny@gmail.com"
git config --global user.name "giohosted"
git config --global credential.helper store
```

This was required on immich-prod-01 during Phase 4 Wave 5 — new VMs will not have git identity pre-configured.

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
- Token stored at `~/.git-credentials` on each host
- If token expires: GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token
- Scope needed: `repo` (full control of private repositories)

---

## Notes

- Always commit before making major changes — gives you a rollback point
- `.env` files go next to their `compose.yaml` but are never committed — use `.env.example` with placeholder values if you want to document required variables
- `/opt/appdata` backup via rsync to NAS is set up in Phase 5
- Any new VM added to the homelab that will host compose files needs git installed, the repo cloned, and git identity configured before first commit