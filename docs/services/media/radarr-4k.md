# Radarr 4K

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** radarr-4k  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/radarr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/radarr-4k/`
- **URL:** `https://radarr-4k.giohosted.com`
- **Network:** proxy
- **Config source:** Fresh — no v2 equivalent

## Container Variables

| Variable | Value |
|----------|-------|
| PUID | 2000 |
| PGID | 2000 |
| TZ | America/Chicago |

---

## Overview

Radarr 4K manages the 4K movie library. It handles automated downloading, importing, and monitoring of 4K WebDL movie releases for local viewing via Infuse on Apple TV 4K. This instance is separate from Radarr 1080p to allow independent quality profiles and avoid serving 4K files to shared users who would trigger transcoding.

A separate Radarr 1080p instance handles the 1080p library. See `services/media/radarr-1080p.md`.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/radarr-4k` | Config and database |
| `/data` | `/data` | Media and downloads — single mount required for hardlinks |

---

## Root Folder

| Path | Purpose |
|------|---------|
| `/data/media/movies/4k` | 4K movie library |

---

## Download Client

| Field | Value |
|-------|-------|
| Name | qBittorrent |
| Host | `gluetun` |
| Port | `8080` |
| Category | `radarr-4k` |

---

## Connected Services

| Service | Purpose |
|---------|---------|
| Prowlarr | Indexer management — syncs all indexers automatically |
| qBitrr | Torrent health monitoring and automated search |

> Bazarr is not connected to Radarr 4K — Bazarr supports one Radarr instance only (connected to Radarr 1080p). Infuse handles subtitles natively for 4K local viewing.

---

## Existing 4K Library Migration

Existing 4K movies from v2 were migrated into this instance using Radarr's Movie Editor:

1. In Radarr 4K → Movie Editor — bulk select all existing 4K titles
2. Set root folder to `/data/media/movies/4k`
3. Trigger library rescan
4. Files were not re-downloaded — existing files were re-pathed and assigned to this instance for monitoring

---

## Notes

- Both Radarr instances use port 7878 internally — they are distinguished by container name (`radarr-1080p` vs `radarr-4k`) not by port
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Radarr
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts
- Not exposed externally — internal only via `local-only@docker` Traefik middleware
- 4K files are for local viewing only via Infuse/Apple TV 4K — never transcode 4K, always direct play