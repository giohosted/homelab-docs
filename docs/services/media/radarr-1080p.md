# Radarr 1080p

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** radarr-1080p  
**Status:** Running  
**Last Updated:** 2026-11-10

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
| Profilarr | Quality profiles, custom formats, and release scoring |

---

## Profilarr / Dumpstarr Configuration

Radarr 1080p is connected to Profilarr v2, which centrally manages its quality profiles and custom formats using the Dumpstarr configuration database.

**Profilarr URL:** `https://profilarr.giohosted.com`  
**Database:** Dumpstarr  
**Repository:** `https://github.com/Dumpstarr/Database`  
**Branch:** `stable`

### Quality Profiles

| Profile | Source |
|---------|--------|
| LQ 1080p | Dumpstarr |
| Movies 1080p | Dumpstarr |
| Movies 1080p HQ | Dumpstarr |

**Synchronization trigger:** On Pull

These three quality profiles were synchronized from Dumpstarr during the Profilarr v2 migration.

Profilarr automatically synchronizes the selected quality profiles and their referenced custom formats when updates are pulled from the Dumpstarr configuration database.

The profiles provide separate release-selection and scoring options for lower-quality, standard, and higher-quality 1080p movie preferences.

### Legacy Quality Profile Cleanup

During the migration, two obsolete Radarr quality profiles were identified:

| Legacy Profile | Resolution |
|----------------|------------|
| 1080p LQ | Replaced by Dumpstarr's `LQ 1080p` profile |
| 1080p Balanced - old | Retired |

Existing movies assigned to `1080p LQ` were reassigned to the new Dumpstarr `LQ 1080p` profile.

Radarr initially prevented deletion of the legacy profiles because collections still referenced them. The relevant collection assignments were corrected, allowing the obsolete quality profiles to be removed.

**Important:** Radarr quality profiles can be referenced by individual movies, collections, and import lists. These dependencies should be reviewed before deleting profiles in the future.

### Synchronization Behavior

- The upstream configuration source is Dumpstarr's `stable` branch.
- The selected quality profiles use Profilarr's `On Pull` synchronization trigger.
- Selected custom formats and release scores are synchronized alongside their associated quality profiles.
- Upstream changes may adjust release scoring, quality preferences, and upgrade behavior.
- Changes made directly to synchronized Radarr quality profiles may be overwritten during subsequent synchronization.
- Media Management and Delay Profile synchronization are separate from Quality Profiles; their final Radarr 1080p trigger settings were not independently recorded during this documentation update.

---

## Notes

- Both Radarr instances use port 7878 internally — they are distinguished by container name (`radarr-1080p` vs `radarr-4k`), not by port.
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Radarr because synchronized changes can overwrite local modifications.
- The active Dumpstarr quality profiles are `LQ 1080p`, `Movies 1080p`, and `Movies 1080p HQ`.
- The legacy `1080p LQ` and `1080p Balanced - old` profiles were retired during the Profilarr v2 migration.
- When retiring quality profiles, check movie assignments, collection assignments, and import lists before attempting deletion.
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts.
- Not exposed externally — internal only via `local-only@docker` Traefik middleware.
