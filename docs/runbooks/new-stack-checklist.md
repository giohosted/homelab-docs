# New Stack Checklist

**Applies to:** docker-prod-01 (`/opt/stacks`)  
**Last Updated:** 2026-03-16

---

## Overview

Every new Docker Compose stack deployed on docker-prod-01 follows this checklist. Steps are mandatory — skipping them causes Dockman autosave failures and potential permission errors on container file writes.

---

## Step 1 — Create directories
```bash
mkdir -p /opt/stacks/<stack-name>
mkdir -p /opt/appdata/<app-name>
```

For stacks with multiple containers, create one appdata directory per container:
```bash
mkdir -p /opt/appdata/<app-1>
mkdir -p /opt/appdata/<app-2>
```

---

## Step 2 — Create compose.yaml and .env
```bash
sudo nano /opt/stacks/<stack-name>/compose.yaml
sudo nano /opt/stacks/<stack-name>/.env
```

---

## Step 3 — Fix ownership and permissions

**Always run this after creating files with sudo nano** — files created with sudo are owned by root and cannot be edited by Dockman.
```bash
sudo chown gio:service /opt/stacks/<stack-name>/compose.yaml
sudo chmod 664 /opt/stacks/<stack-name>/compose.yaml
```

If the stack has a `.env` file:
```bash
sudo chown gio:service /opt/stacks/<stack-name>/.env
sudo chmod 660 /opt/stacks/<stack-name>/.env
```

### Why these values
| Path | Owner | Group | Mode | Reason |
|------|-------|-------|------|--------|
| `compose.yaml` | `gio` | `service` | `664` | Dockman runs as service user and needs write access |
| `.env` | `gio` | `service` | `660` | Secrets must not be world-readable |
| `/opt/appdata/<app>/` | `2000:2000` (service) | — | `755` | Containers run as PUID/PGID 2000 |

---

## Step 4 — Fix appdata ownership

Appdata directories must be owned by `2000:2000` so containers can write to them:
```bash
sudo chown -R 2000:2000 /opt/appdata/<app-name>/
```

---

## Step 5 — Commit to homelab-infra
```bash
cd /opt/stacks
git add .
git commit -m "add <stack-name> compose"
git push
```

---

## Step 6 — Add DNS rewrite in AdGuard

For any service exposed via Traefik, add a DNS rewrite in AdGuard at `192.168.30.10`:

| Domain | Answer |
|--------|--------|
| `<service>.giohosted.com` | `192.168.30.11` |

---

## Notes

- `/opt/appdata` is **not** in git — it contains container state and secrets. It is rsync-backed to NAS nightly (Phase 5).
- `.env` files are gitignored — never commit real secrets. Use `.env.example` with placeholder values if you want to document required variables.
- The `service` user on docker-prod-01 is UID/GID 2000 — same as the PUID/PGID used by all containers.
- This checklist will be superseded when a proper umask or ACL solution is implemented.