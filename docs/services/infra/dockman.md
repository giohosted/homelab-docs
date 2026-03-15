# Dockman

**Role:** Docker Compose management UI — start, stop, restart individual containers without stack restarts  
**Host:** docker-prod-01 (192.168.30.11)  
**Compose:** `/opt/stacks/dockman/compose.yaml`  
**Appdata:** `/opt/appdata/dockman/`  
**URL:** https://dockman.giohosted.com  
**Last Updated:** 2026-03-15

---

## Overview

Dockman provides a web UI for managing Docker Compose stacks on docker-prod-01. Its key advantage over other Docker UIs is the ability to restart or update individual containers without restarting the entire stack — critical for the ARR stack where restarting one service shouldn't affect others.

Dockman is internal only — not exposed via Cloudflare Tunnel.

---

## Configuration

### Environment Variables (.env)

| Variable | Notes |
|----------|-------|
| `DOCKMAN_AUTH_USERNAME` | Login username — stored in `.env`, gitignored |
| `DOCKMAN_AUTH_PASSWORD` | Login password — stored in `.env`, gitignored |

### Key Settings

| Setting | Value | Notes |
|---------|-------|-------|
| `DOCKMAN_COMPOSE_ROOT` | `/opt/stacks` | Root directory Dockman manages — matches homelab-infra repo clone location |
| `DOCKMAN_AUTH_ENABLE` | `true` | Basic auth enabled |
| `DOCKMAN_LOG_AUTH_WARNING` | `false` | Suppresses auth warning since auth is enabled |

### Volumes

| Host Path | Container Path | Notes |
|-----------|---------------|-------|
| `/opt/stacks` | `/opt/stacks` | Compose files — Dockman reads and manages these |
| `/opt/appdata/dockman/config` | `/config` | Dockman config and state |
| `/var/run/docker.sock` | `/var/run/docker.sock` | Docker socket — required for container management |

---

## Access

- **URL:** `https://dockman.giohosted.com`
- **Restricted to:** Internal VLANs only via `local-only` Traefik middleware (192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24)
- **Not exposed** via Cloudflare Tunnel

---

## Notes

- Dockman manages stacks at `/opt/stacks` — this is the homelab-infra git repo clone. Any compose changes made via Dockman UI should also be committed to git manually afterward.
- Port 8866 is not exposed directly — all access goes through Traefik
- Fresh install in v3 — appdata from v2 Synology backup not restored (config is minimal, not worth restoring)