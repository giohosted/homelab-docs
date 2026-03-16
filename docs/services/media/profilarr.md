# Profilarr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** profilarr  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** santiagosayshey/profilarr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/profilarr/`
- **URL:** `https://profilarr.giohosted.com`
- **Network:** proxy
- **Config source:** Restored from v2 backup

## Container Variables

| Variable | Value |
|----------|-------|
| PUID | 2000 |
| PGID | 2000 |
| TZ | America/Chicago |

---

## Overview

Profilarr manages quality profiles and custom formats across all ARR instances. It imports TRaSH Guides configurations and syncs them to Sonarr and Radarr — keeping quality profiles, custom formats, and scoring consistent without manual configuration in each instance.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/profilarr` | Config and database |

---

## Connected Instances

| App | URL |
|-----|-----|
| Sonarr TV | `http://sonarr-tv:8989` |
| Sonarr Anime | `http://sonarr-anime:8989` |
| Radarr 1080p | `http://radarr-1080p:7878` |
| Radarr 4K | `http://radarr-4k:7878` |

---

## Notes

- Not exposed externally — internal only via `local-only@docker` Traefik middleware
- Quality profiles and custom formats should be managed through Profilarr, not directly in individual ARR instances — direct edits will be overwritten on next sync