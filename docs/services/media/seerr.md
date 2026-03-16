# Seerr

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** seerr  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** ghcr.io/seerr-team/seerr:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/seerr/`
- **URL:** `https://request.giohosted.com`
- **Network:** proxy
- **Config source:** Restored from v2 backup (formerly Overseerr)

## Container Variables

| Variable | Value |
|----------|-------|
| TZ | America/Chicago |

> Seerr v3 runs internally as `node:node` (UID 1000:1000) — PUID/PGID are not used. Appdata must be owned by `1000:1000`.
```bash
sudo chown -R 1000:1000 /opt/appdata/seerr/
```

---

## Overview

Seerr is the media request frontend. Users submit requests for movies and TV shows through Seerr, which passes them to the appropriate Radarr or Sonarr instance. Requests are tracked and users are notified when their media becomes available.

Seerr is exposed externally via Cloudflare Tunnel at `https://request.giohosted.com`.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/app/config` | `/opt/appdata/seerr` | Config and database |

---

## Connected Instances

| Service | URL | Purpose |
|---------|-----|---------|
| Plex | `http://192.168.30.2:32400` | User authentication and library sync |
| Radarr 1080p | `http://radarr-1080p:7878` | Movie requests |
| Radarr 4K | `http://radarr-4k:7878` | 4K movie requests |
| Sonarr TV | `http://sonarr-tv:8989` | TV requests |
| Sonarr Anime | `http://sonarr-anime:8989` | Anime requests |

---

## External Access

Exposed via Cloudflare Tunnel:

| Hostname | Routes to | Notes |
|----------|-----------|-------|
| `request.giohosted.com` | `https://192.168.30.11` | Via Traefik, No TLS Verify enabled |

---

## Issues Encountered

| Issue | Resolution |
|-------|------------|
| Crash-looping on startup | Seerr v3 runs as `node:node` (UID 1000) — v2 appdata was owned by `service` (UID 2000). Fixed with `chown -R 1000:1000` on appdata |
| Repeatedly falling off proxy network | Caused by crash-loop — resolved once permissions were fixed and container stabilized |

---

## Notes

- URL is `request.giohosted.com` not `seerr.giohosted.com` — user-facing name is more intuitive
- Seerr is a fork of Overseerr — v2 Overseerr config and database restored successfully
- Authentik OIDC integration carried forward from v2