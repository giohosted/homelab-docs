# Plex Media Server

**Host:** nas-prod-01  
**Container name:** plex  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** lscr.io/linuxserver/plex (linuxserver.io)
- **Managed via:** Unraid Docker UI — not part of any compose stack
- **Network:** custom `eth1` (X710 Port 2, VLAN 30 data interface)
- **Container IP:** 192.168.30.2
- **Web UI:** `http://192.168.30.2:32400/web`

> Plex runs directly on Unraid, not on docker-prod-01. This keeps media access local — no NFS hop for transcoding. QuickSync iGPU is on the same host as the media files.

---

## Container Variables

| Variable | Value |
|----------|-------|
| PUID | 2000 |
| PGID | 2000 |
| VERSION | docker |

---

## Paths

| Container Path | Host Path | Purpose |
|----------------|-----------|---------|
| `/data` | `/mnt/user/data` | Full data share access — media and downloads |
| `/config` | `/mnt/user/appdata/plex` | Plex database and config |

---

## Hardware Transcoding

- **iGPU:** Intel UHD 730 (i5-13400, Intel Raptor Lake-S)
- **Method:** QuickSync
- **Device passthrough:** `/dev/dri` mapped into container
- **Plex transcoder settings:** Hardware acceleration enabled, hardware-accelerated encoding enabled, device set to Intel Raptor Lake-S UHD Graphics
- **HEVC encoding:** HEVC sources only

---

## Libraries

| Library | Path |
|---------|------|
| Movies (1080p) | `/data/media/movies/1080p` |
| Movies (4K) | `/data/media/movies/4k` |
| TV | `/data/media/tv` |
| Anime | `/data/media/anime` |
| Audiobooks | `/data/media/books/audiobooks` |

Libraries were configured in Phase 4 after media migration from TrueNAS completed. DB restored from `/mnt/user/backups/plex/db/plex-db-2026-03-10.tar.gz`.

---

## External Access

- Port 32400 forwarded on UDM-SE → `192.168.30.2:32400`
- Plex handles its own TLS and relay
- Not behind Traefik — direct port forward is intentional
- Not exposed via Cloudflare Tunnel — Cloudflare ToS prohibits video streaming through their tunnel

---

## Backup

Plex DB is backed up via a dedicated script (Phase 5):
- Script stops Plex, backs up `/mnt/user/appdata/plex` to `/mnt/user/backups/plex/db/`, restarts Plex
- Uses EXIT trap to ensure Plex restarts even if backup fails

---

## Issues Encountered

| Issue | Resolution |
|-------|------------|
| DB restore permissions error | Extracted files owned by root — Plex runs as 2000:2000. Fixed with `chown -R 2000:2000` on restored files |
| Plex container media path after NFS restructure | Path updated from `/mnt/user` to `/mnt/user/data` after single `data` share was created in Wave 2 |

---

## Notes

- Plex is not managed by Dockman — it lives on Unraid and is managed via the Unraid Docker UI
- The 4K library migration (existing 4K movies into Radarr 4K instance) is handled via Radarr's Movie Editor — see `services/media/radarr-4k.md`
- Future: 4K library expansion via Radarr 4K automated downloads