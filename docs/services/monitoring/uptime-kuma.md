# Uptime Kuma

**Host:** pi-prod-01 (192.168.10.20)
**Status:** Running
**Last Updated:** 2026-03-17

---

## Overview

Uptime Kuma is a self-hosted uptime monitoring tool. It monitors all homelab services and hosts, alerting when anything goes down.

---

## Deployment

- **Install method:** Docker Compose
- **Image:** `louislam/uptime-kuma:2`
- **Web UI:** `http://192.168.10.20:3001`
- **Access:** From VLAN 20 (Trusted) — port 3001 is included in `MGMT_ADMIN_PORTS` firewall group
- **Database:** Embedded MariaDB
- **Data directory:** `/opt/appdata/uptime-kuma`

### compose.yaml
```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    network_mode: host
    dns:
      - 192.168.30.10
      - 192.168.30.15
    volumes:
      - /opt/appdata/uptime-kuma:/app/data
```

> **DNS note:** The container DNS is explicitly set to AdGuard (dns-prod-01 + dns-prod-02) so that internal `*.giohosted.com` hostnames resolve correctly inside the container. Without this, Docker uses its own resolver which has no knowledge of internal rewrites.

---

## Monitors

### Hosts (Ping)

| Name | IP |
|------|----|
| pve-prod-01 | 192.168.10.11 |
| pve-prod-02 | 192.168.10.12 |
| nas-prod-01 | 192.168.10.10 |
| pi-prod-01 | 192.168.10.20 |
| docker-prod-01 | 192.168.30.11 |
| auth-prod-01 | 192.168.30.13 |
| immich-prod-01 | 192.168.30.14 |
| dns-prod-01 | 192.168.30.10 |
| dns-prod-02 | 192.168.30.15 |
| pbs-prod-01 | 192.168.30.12 |

### Infrastructure (HTTP)

| Name | URL |
|------|-----|
| Traefik | https://traefik.giohosted.com/dashboard/ |
| Authentik | https://auth.giohosted.com |
| Dockman | https://dockman.giohosted.com |
| AdGuard Primary | http://192.168.30.10:3000 |
| AdGuard Secondary | http://192.168.30.15:3000 |
| Beszel | http://192.168.10.20:8090 |

### Media (HTTP)

| Name | URL |
|------|-----|
| Plex | http://192.168.30.2:32400/web |
| Sonarr TV | https://sonarr-tv.giohosted.com |
| Sonarr Anime | https://sonarr-anime.giohosted.com |
| Radarr 1080p | https://radarr-1080p.giohosted.com |
| Radarr 4K | https://radarr-4k.giohosted.com |
| Prowlarr | https://prowlarr.giohosted.com |
| Bazarr | https://bazarr.giohosted.com |
| Profilarr | https://profilarr.giohosted.com |
| Maintainerr | https://maintainerr.giohosted.com |
| Overseerr | https://request.giohosted.com |
| Tautulli | https://tautulli.giohosted.com |

### Torrent (HTTP)

| Name | URL |
|------|-----|
| qBittorrent | https://qbittorrent.giohosted.com |

### Books (HTTP)

| Name | URL |
|------|-----|
| Audiobookshelf | https://audiobooks.giohosted.com |
| Calibre-Web | https://cwa.giohosted.com |
| Shelfmark | https://shelf.giohosted.com |

### Photos (HTTP)

| Name | URL |
|------|-----|
| Immich | https://photos.giohosted.com |

---

## Useful Commands
```bash
# Restart Uptime Kuma
cd /opt/stacks/uptime-kuma
docker compose restart

# View logs
docker logs uptime-kuma

# Update
docker compose pull
docker compose up -d