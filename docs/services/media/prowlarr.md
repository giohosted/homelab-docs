# Prowlarr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** prowlarr  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/prowlarr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/prowlarr/`
- **URL:** `https://prowlarr.giohosted.com`
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

Prowlarr is the indexer manager for the ARR stack. It manages all torrent indexers in one place and syncs them to all connected ARR instances automatically. Adding or updating an indexer in Prowlarr propagates to all instances — no need to configure indexers in each ARR instance individually.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/prowlarr` | Config and database |

---

## Connected Applications

| App | URL |
|-----|-----|
| Sonarr TV | `http://sonarr-tv:8989` |
| Sonarr Anime | `http://sonarr-anime:8989` |
| Radarr 1080p | `http://radarr-1080p:7878` |
| Radarr 4K | `http://radarr-4k:7878` |

---

## FlareSolverr

FlareSolverr is configured as a proxy in Prowlarr for indexers that require Cloudflare bypass:

| Field | Value |
|-------|-------|
| Name | FlareSolverr |
| Type | FlareSolverr |
| Host | `http://flaresolverr:8191` |
| Tag | `flaresolverr` |

Indexers that require FlareSolverr are tagged with `flaresolverr` in Prowlarr. Only tagged indexers route through it.

---

## Notes

- Indexers are managed here only — do not add indexers directly in individual ARR instances
- Not exposed externally — internal only via `local-only@docker` Traefik middleware
- If an indexer shows as failing, check FlareSolverr is running before assuming the indexer is down