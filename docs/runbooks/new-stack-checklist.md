# New Stack Checklist

**Applies to:** docker-prod-01 (`/opt/stacks`)  
**Last Updated:** 2026-11-16

---

## Overview

Use this checklist for new Docker Compose stacks on docker-prod-01. Komodo is the current management UI; Dockman was removed in November 2026. Check permissions for the actual editor, runtime user and service rather than applying legacy Dockman-specific ownership rules indiscriminately.

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

## Step 3 — Review ownership and permissions

Files created with `sudo nano` can be owned by `root:root`. Ensure the account expected to maintain the repository can edit tracked Compose files, while keeping `.env` files restricted. The following values are the **previously established docker-prod-01 conventions**, not a new requirement imposed by Komodo.
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
| `compose.yaml` | `gio` | `service` | `664` | Previously used for shared group editing; reevaluate group write access for each stack |
| `.env` | `gio` | `service` | `660` | Secrets must not be world-readable |
| `/opt/appdata/<app>/` | `2000:2000` (service) | — | `755` | Containers run as PUID/PGID 2000 |

---

## Step 4 — Fix appdata ownership

For containers explicitly configured to run as UID/GID `2000:2000`, the established appdata ownership may be appropriate:
```bash
sudo chown -R 2000:2000 /opt/appdata/<app-name>/
```

**Do not run this command against all appdata indiscriminately.** Databases and other containers can have different ownership requirements. Confirm the image user and volume paths before changing permissions.

---

## Step 5 — Commit to homelab-infra
```bash
cd /opt/stacks
git status --short
git add -- <stack-name>/compose.yaml
git diff --cached
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
- The documented `service` account/group on docker-prod-01 uses UID/GID 2000 for applicable services; do not assume every container uses that identity.
- Existing `gio:service` permissions may still be present from the Dockman era, but Komodo does not by itself justify granting group write access. Reassess least privilege when changing a stack.
- This checklist can be refined if a consistent host-specific umask or ACL policy is implemented.
