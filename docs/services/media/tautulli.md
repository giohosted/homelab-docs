# Tautulli

**Host:** docker-prod-01 (192.168.30.11)  
**Container name:** tautulli  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/tautulli:latest
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **Appdata:** `/opt/appdata/tautulli/`
- **URL:** `https://tautulli.giohosted.com`
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

Tautulli is the Plex analytics and monitoring tool. It tracks play history, user activity, and library statistics. It also handles Plex notifications — play/stop events, new media alerts, and user activity summaries.

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/config` | `/opt/appdata/tautulli` | Config, database, and logs |

---

## Connected Instances

| Service | URL | Purpose |
|---------|-----|---------|
| Plex | `http://192.168.30.2:32400` | Play history, library stats, user activity |

---

## Notes

- Not exposed externally — internal only via `local-only@docker` Traefik middleware
- Plex connection uses the VLAN 30 data interface IP (`192.168.30.2`) — not the management IP
- Play history and statistics carried forward from v2 backup