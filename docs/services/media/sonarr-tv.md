# Sonarr TV

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** sonarr-tv  
**Status:** Running  
**Last Updated:** 2026-11-10

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
| Profilarr | Quality profiles, custom formats, release scoring, and supported media settings |

---

## Profilarr / Dumpstarr Configuration

Sonarr TV is connected to Profilarr v2, which manages its quality profiles and custom formats using the Dumpstarr configuration database.

**Profilarr URL:** `https://profilarr.giohosted.com`  
**Database:** Dumpstarr  
**Repository:** `https://github.com/Dumpstarr/Database`  
**Branch:** `stable`

### Quality Profiles

| Profile | Source |
|---------|--------|
| TV 1080p | Dumpstarr |
| TV 2160p | Dumpstarr |

**Synchronization trigger:** On Pull

Profilarr automatically synchronizes selected quality profiles and their referenced custom formats when updates are pulled from Dumpstarr.

### Media Management

The following configurations were synchronized from Dumpstarr during the migration:

- Sonarr – Plex Naming
- Sonarr Quality Definitions
- Sonarr Media Settings

**Synchronization trigger:** Manual

These settings are not automatically reapplied during routine database pulls.

### Delay Profiles

**Profile:** Dumpstarr — Default Delay  
**Synchronization trigger:** Manual

The synchronized delay profile uses:

| Setting | Value |
|---------|-------|
| Preferred protocol | Usenet |
| Usenet delay | 60 minutes |
| Torrent delay | 60 minutes |

The preference for Usenet does not introduce a Usenet download client. With only qBittorrent configured, available downloads continue through torrents.

Delay profiles primarily influence RSS release selection rather than interactive or API-triggered searches.

---

## Notes

- Both Sonarr instances use port 8989 internally — they are distinguished by container name (`sonarr-tv` vs `sonarr-anime`), not by port.
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Sonarr because synchronized changes can overwrite local modifications.
- Profilarr quality profile synchronization is configured to `On Pull`; media management and delay profile synchronization remain manual.
- Changes to upstream Dumpstarr scores, cutoffs, and custom formats can affect future grabs and upgrades.
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts.
- Not exposed externally — internal only via `local-only@docker` Traefik middleware.
