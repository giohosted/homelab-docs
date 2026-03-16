# Sonarr TV

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** sonarr-tv  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/sonarr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/sonarr-tv/`
- **URL:** `https://sonarr-tv.giohosted.com`
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

Sonarr TV manages the standard TV library. It handles automated downloading, importing, and monitoring of TV show releases. This instance covers all non-anime TV content.

A separate Sonarr Anime instance handles the anime library. See `services/media/sonarr-anime.md`.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/sonarr-tv` | Config and database |
| `/data` | `/data` | Media and downloads — single mount required for hardlinks |

---

## Root Folder

| Path | Purpose |
|------|---------|
| `/data/media/tv` | TV library |

---

## Download Client

| Field | Value |
|-------|-------|
| Name | qBittorrent |
| Host | `gluetun` |
| Port | `8080` |
| Category | `sonarr-tv` |

---

## Connected Services

| Service | Purpose |
|---------|---------|
| Prowlarr | Indexer management — syncs all indexers automatically |
| Bazarr | Subtitle automation |
| Seerr | Media request frontend |
| qBitrr | Torrent health monitoring and automated search |

---

## Notes

- Both Sonarr instances use port 8989 internally — they are distinguished by container name (`sonarr-tv` vs `sonarr-anime`) not by port
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Sonarr
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts
- Not exposed externally — internal only via `local-only@docker` Traefik middleware