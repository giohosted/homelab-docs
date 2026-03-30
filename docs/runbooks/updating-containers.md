# Updating a Docker Container

**Last Updated:** 2026-03-19

---

## Overview

All containers in homelab-v3 are managed via Docker Compose stacks under `/opt/stacks`. Updating a container means pulling the latest image and recreating the container. Dockman can do this via the UI, or you can do it via CLI.

---

## Method 1 — Dockman (UI)

1. Go to `https://dockman.giohosted.com`
2. Find the stack containing the container you want to update
3. Click the stack → **Update** (pulls latest images for all containers in the stack)
4. Click **Restart** to recreate the containers with the new image

> Dockman operates at the stack level — it pulls and restarts all containers in a stack together.

---

## Method 2 — CLI

SSH into the host where the container runs, then:
```bash
# 1. Navigate to the stack directory
cd /opt/stacks/<stack-name>

# 2. Pull the latest image
docker compose pull

# 3. Recreate the container with the new image
docker compose up -d

# 4. Verify the container is running
docker ps | grep <container-name>

# 5. Check logs for any errors
docker logs <container-name> --tail 50
```

### Example — updating Authentik on auth-prod-01
```bash
ssh gio@192.168.30.13
cd /opt/stacks/authentik
docker compose pull
docker compose up -d
docker ps | grep authentik
docker logs authentik-server --tail 50
```

### Example — updating the media stack on docker-prod-01
```bash
ssh gio@192.168.30.11
cd /opt/stacks/media
docker compose pull
docker compose up -d
docker ps
```

---

## Cleaning Up Old Images

After updating, old images linger on disk. Clean them up periodically:
```bash
docker image prune -f
```

This removes all dangling images (old versions no longer used by any container). Safe to run at any time.

---

## Stack Locations by Host

| Host | IP | Stacks path |
|------|----|-------------|
| docker-prod-01 | 192.168.30.11 | `/opt/stacks` |
| auth-prod-01 | 192.168.30.13 | `/opt/stacks` |
| immich-prod-01 | 192.168.30.14 | `/opt/stacks` |

---

## Notes

- Always `git pull` in `/opt/stacks` before making any changes — someone (you) may have edited the compose file from another host
- `docker compose up -d` only recreates containers whose image or config has changed — it does not restart unchanged containers
- Authentik updates may include database migrations that run automatically on startup — check logs after updating
- Immich updates can be disruptive if they include breaking DB migrations — always check the Immich release notes before updating
- After updating, commit nothing to git unless you intentionally changed a compose file or `.env`