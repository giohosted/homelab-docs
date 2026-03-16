# Bazarr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** bazarr  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/bazarr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/bazarr/`
- **URL:** `https://bazarr.giohosted.com`
- **Network:** proxy
- **Config source:** Restored from v2 backup

## Container Variables

| Variable | Value |
|----------|-------|
| PUID | 2000 |
| PGID | 2000 |
| TZ | America/Chicago |

---

## Connected Instances

Bazarr 1.5.6 supports one Sonarr instance and one Radarr instance only.

| App | URL | Coverage |
|-----|-----|----------|
| Sonarr TV | `http://sonarr-tv:8989` | TV subtitles |
| Radarr 1080p | `http://radarr-1080p:7878` | Movie subtitles |
| Plex | `http://192.168.30.2:32400` | Subtitle notifications |

> Sonarr Anime and Radarr 4K are not connected — Bazarr only supports one instance of each. Anime subtitles are typically embedded in the release. Radarr 4K is local viewing only via Infuse which handles subtitles natively.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/bazarr` | Config and database |
| `/data` | `/data` | Media access for subtitle matching |

---

## Notes

- Subtitles are downloaded and matched against files in `/data/media/tv` and `/data/media/movies/1080p`
- Plex connection is for subtitle event notifications only — Bazarr tells Plex to refresh metadata after a subtitle is downloaded