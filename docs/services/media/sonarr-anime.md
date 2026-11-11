# Sonarr Anime

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** sonarr-anime  
**Status:** Running  
**Last Updated:** 2026-11-10

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
| Prowlarr | Indexer management — syncs anime-specific indexers |
| qBitrr | Torrent health monitoring and automated search |
| Profilarr | Anime quality profiles, custom formats, release scoring, and supported media settings |

> Bazarr is not connected to Sonarr Anime — Bazarr supports one Sonarr instance only (connected to Sonarr TV). Anime subtitles are typically embedded in the release.

---

## Profilarr / Dumpstarr Configuration

Sonarr Anime is connected to Profilarr v2, which manages its anime-specific quality profile and custom formats using the Dumpstarr configuration database.

**Profilarr URL:** `https://profilarr.giohosted.com`  
**Database:** Dumpstarr  
**Repository:** `https://github.com/Dumpstarr/Database`  
**Branch:** `stable`

### Quality Profiles

| Profile | Source |
|---------|--------|
| Anime 1080p | Dumpstarr |

**Synchronization trigger:** On Pull

The existing `Anime 1080p` quality profile was synchronized through Profilarr during migration.

Profilarr automatically synchronizes the selected quality profile and its referenced custom formats when updates are pulled from Dumpstarr.

This allows anime-specific release scoring and custom format recommendations to remain current without manually editing the profile in Sonarr.

### Media Management

The following configurations were synchronized from Dumpstarr during migration:

- Sonarr – Plex Naming
- Sonarr Quality Definitions
- Sonarr Media Settings

**Synchronization trigger:** Manual

These settings are not automatically reapplied during routine database pulls.

### Delay Profiles

**Profile:** Dumpstarr — Default Delay  
**Synchronization trigger:** Manual

The Dumpstarr default delay profile specifies:

| Setting | Value |
|---------|-------|
| Preferred protocol | Usenet |
| Usenet delay | 60 minutes |
| Torrent delay | 60 minutes |

The delay profile was synchronized during migration. These are the upstream Dumpstarr values; the resulting settings were not separately recorded from Sonarr Anime after synchronization.

Since qBittorrent is the configured download client and no Usenet client is configured, downloads continue through torrents.

Delay profiles primarily influence RSS release selection rather than interactive or API-triggered searches.

---

## Notes

- Both Sonarr instances use port 8989 internally — they are distinguished by container name (`sonarr-tv` vs `sonarr-anime`), not by port.
- Quality profiles and custom formats are managed by Profilarr — do not edit them directly in Sonarr because synchronized changes can overwrite local modifications.
- Profilarr quality profile synchronization is configured to `On Pull`; media management and delay profile synchronization remain manual.
- Changes to upstream Dumpstarr scores, cutoffs, and custom formats can affect future grabs and upgrades.
- Hardlinks require `/data` to be a single NFS mount covering both `/data/media` and `/data/downloads` — do not split these into separate mounts.
- Not exposed externally — internal only via `local-only@docker` Traefik middleware.
- Anime indexers (e.g., Nyaa) are configured in Prowlarr and tagged to sync to this instance only.
