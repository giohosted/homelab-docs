# Sonarr Anime

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** sonarr-anime  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/sonarr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/sonarr-anime/`
- **URL:** `https://sonarr-anime.giohosted.com`
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

Sonarr Anime manages the anime library. It is a separate instance from Sonarr TV to allow anime-specific quality profiles, release group preferences, and indexer configuration. Anime releases have different naming conventions and quality sources than standard TV — a shared instance gets messy quickly.

A separate Sonarr TV instance handles standard TV shows. See `services/media/sonarr-tv.md`.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/sonarr-anime` | Config and database |
| `/data` | `/data` | Media and downloads — single mount required for hardlinks |

---

## Root Folder

| Path | Purpose |
|------|---------|
| `/data/media/anime` | Anime library |

---

## Download Client

| Field | Value |
|-------|-------|
| Name | qBittorrent |
| Host | `gluetun` |
| Port | `8080` |
| Category | `sonarr-anime` |

---

## Connected Services

| Service | Purpose |
|---------|---------|
| Prowlarr | Indexer management — syncs all indexers automatically |
| qBitrr | Torrent health monitoring and automated search |

> Bazarr is not connected to Sonarr Anime — Bazarr supports one Sonarr instance only (connected to Sonarr TV). Anime subtitles are typically embedded in the release.

---

## Notes

- Both Sonarr instances use port 8989 internally — they are distinguished by container name (`sonarr-tv` vs `sonarr-anime`) not by port
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Sonarr
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts
- Not exposed externally — internal only via `local-only@docker` Traefik middleware
- Anime indexers (e.g. Nyaa) are configured in Prowlarr and tagged to sync to this instance only