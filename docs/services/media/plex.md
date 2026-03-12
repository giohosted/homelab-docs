# Plex Media Server

**Host:** nas-prod-01  
**Container name:** plex  
**Status:** Running  
**Last Updated:** 2026-03-11

---

## Deployment

- Runs as a Docker container directly on Unraid (not on docker-prod-01)
- Managed via Unraid Docker UI — not part of any compose stack
- Template: lnxserver/plex (linuxserver.io)
- Network: custom eth1 (VLAN 30 data interface)
- Container IP: 192.168.30.2
- Web UI: http://192.168.30.2:32400/web

---

## Hardware Transcoding

- iGPU: Intel UHD 730 (i5-13400, Intel Raptor Lake-S)
- Method: QuickSync
- Device passthrough: /dev/dri mapped into container
- Plex transcoder settings: hardware acceleration enabled, hardware-accelerated encoding enabled, device set to Intel Raptor Lake-S UHD Graphics

---

## Paths

| Container Path | Host Path | Purpose |
|---|---|---|
| /data | /mnt/user | Full NAS share access |
| /config | /mnt/user/appdata/plex | Plex database and config |

---

## Container Variables

| Variable | Value |
|---|---|
| PUID | 2000 |
| PGID | 2000 |
| VERSION | docker |

---

## External Access

- Port 32400 forwarded on UDM-SE → 192.168.30.2:32400
- Plex handles its own TLS and relay
- Not behind Traefik — direct port forward intentional (Cloudflare ToS prohibits video streaming via tunnel)

> **Note:** Port forward not yet configured — pending Phase 4 when Plex libraries are populated and external access is needed.

---

## Libraries

Configured in Phase 4 after media migration from TrueNAS completes.

| Library | Path |
|---|---|
| Movies (1080p) | /data/media/movies/1080p |
| Movies (4K) | /data/media/movies/4k |
| TV | /data/media/tv |
| Anime | /data/media/anime |
| Audiobooks | /data/media/books/audiobooks |