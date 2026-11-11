# Radarr 4K

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** radarr-4k  
**Status:** Running  
**Last Updated:** 2026-11-10

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
| Profilarr | Quality profiles, custom formats, and release scoring |

> Bazarr is not connected to Radarr 4K — Bazarr supports one Radarr instance only (connected to Radarr 1080p). Infuse handles subtitles natively for 4K local viewing.

---

## Profilarr / Dumpstarr Configuration

Radarr 4K is connected to Profilarr v2, which centrally manages its quality profiles and custom formats using the Dumpstarr configuration database.

**Profilarr URL:** `https://profilarr.giohosted.com`  
**Database:** Dumpstarr  
**Repository:** `https://github.com/Dumpstarr/Database`  
**Branch:** `stable`

### Quality Profiles

Radarr 4K uses quality profiles synchronized from Dumpstarr.

**Synchronization trigger:** On Pull

The selected Dumpstarr quality profiles were synchronized during the Profilarr v2 migration. Their exact names were not independently recorded during this documentation update.

The active selections can be reviewed under:

`Profilarr → Arrs → Radarr 4K → Sync → Quality Profiles`

Profilarr automatically synchronizes the selected quality profiles and their referenced custom formats when updates are pulled from the Dumpstarr configuration database.

This keeps supported release scoring, quality preferences, and custom format recommendations synchronized without requiring manual adjustments in Radarr.

### Synchronization Behavior

- The upstream configuration source is Dumpstarr's `stable` branch.
- Profilarr is configured to pull upstream database updates automatically, hourly.
- The selected quality profiles use the `On Pull` synchronization trigger.
- Selected custom formats and release scores are synchronized alongside their associated quality profiles.
- Upstream changes can affect release selection, scoring, and upgrade behavior.
- Direct modifications to synchronized quality profiles may be overwritten during subsequent synchronization.
- Media Management and Delay Profile synchronization are controlled separately from Quality Profiles; their final Radarr 4K trigger settings were not independently recorded during this documentation update.

---

## Existing 4K Library Migration

Existing 4K movies from v2 were migrated into this instance using Radarr's Movie Editor:

1. In Radarr 4K → Movie Editor — bulk select all existing 4K titles
2. Set root folder to `/data/media/movies/4k`
3. Trigger library rescan
4. Files were not re-downloaded — existing files were re-pathed and assigned to this instance for monitoring

---

## Notes

- Both Radarr instances use port 7878 internally — they are distinguished by container name (`radarr-1080p` vs `radarr-4k`), not by port.
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Radarr because synchronized changes can overwrite local modifications.
- Profilarr Quality Profiles synchronization is configured to `On Pull` using the Dumpstarr database.
- Changes to upstream Dumpstarr scores, cutoffs, and custom formats can affect future grabs and upgrades.
- Quality profile dependencies may include movies, collections, and import lists. Review these before removing obsolete profiles.
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts.
- Not exposed externally — internal only via `local-only@docker` Traefik middleware.
- 4K files are for local viewing only via Infuse/Apple TV 4K — never transcode 4K, always direct play.
