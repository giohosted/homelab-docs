# Maintainerr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** maintainerr  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** ghcr.io/maintainerr/maintainerr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/maintainerr/`
- **URL:** `https://maintainerr.giohosted.com`
- **Network:** proxy
- **Config source:** Restored from v2 backup

## Container Variables

| Variable | Value |
|----------|-------|
| TZ | America/Chicago |

> Maintainerr v2.0.0+ runs internally as the `node` user (UID 1000) — PUID/PGID are not used. Appdata must be owned by `1000:1000`.
```bash
sudo chown -R 1000:1000 /opt/appdata/maintainerr/
```

---

## Overview

Maintainerr provides visibility into media lifecycle — it shows what is eligible for removal based on configurable rules (last watched date, number of plays, request age, etc.). 

**Deletion is not enabled** — Maintainerr is used for visibility only. No media is automatically removed.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/opt/app/data` | `/opt/appdata/maintainerr` | Config and database |

---

## Connected Instances

| App | URL |
|-----|-----|
| Plex | `http://192.168.30.2:32400` |
| Sonarr TV | `http://sonarr-tv:8989` |
| Radarr 1080p | `http://radarr-1080p:7878` |

---

## Notes

- Appdata ownership must be `1000:1000` — container crashes on startup if owned by 2000:2000
- Deletion is intentionally disabled — Maintainerr is for monitoring media lifecycle only
- Not exposed externally — internal only via `local-only@docker` Traefik middleware