# Radarr 1080p

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** radarr-1080p  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/radarr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/radarr-1080p/`
- **URL:** `https://radarr-1080p.giohosted.com`
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

Radarr 1080p manages the 1080p movie library. It handles automated downloading, importing, and monitoring of 1080p WebDL movie releases. This instance is the primary movie instance for shared users — 1080p avoids transcoding for most clients.

A separate Radarr 4K instance handles the 4K library. See `services/media/radarr-4k.md`.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/radarr-1080p` | Config and database |
| `/data` | `/data` | Media and downloads — single mount required for hardlinks |

---

## Root Folder

| Path | Purpose |
|------|---------|
| `/data/media/movies/1080p` | 1080p movie library |

---

## Download Client

| Field | Value |
|-------|-------|
| Name | qBittorrent |
| Host | `gluetun` |
| Port | `8080` |
| Category | `radarr-1080` |

---

## Connected Services

| Service | Purpose |
|---------|---------|
| Prowlarr | Indexer management — syncs all indexers automatically |
| Bazarr | Subtitle automation |
| Tautulli | Plex analytics (via Plex connection) |
| qBitrr | Torrent health monitoring and automated search |

---

## Notes

- Both Radarr instances use port 7878 internally — they are distinguished by container name (`radarr-1080p` vs `radarr-4k`) not by port
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Radarr
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts
- Not exposed externally — internal only via `local-only@docker` Traefik middleware