# qBittorrent

**Role:** Torrent client — runs inside Gluetun VPN network namespace  
**Host:** docker-prod-01 (192.168.30.11)  
**Image:** lscr.io/linuxserver/qbittorrent:latest  
**Version:** 5.1.4  
**Compose:** `/opt/stacks/torrent/compose.yaml`  
**Appdata:** `/opt/appdata/qbittorrent/`  
**URL:** `https://qbittorrent.giohosted.com`  
**Last Updated:** 2026-03-16

---

## Overview

qBittorrent is the torrent client for all media downloads. It runs inside Gluetun's network namespace — meaning it has no independent network identity and all traffic is forced through the ProtonVPN WireGuard tunnel. If the VPN drops, qBittorrent loses all connectivity.

qBittorrent does not appear on the `proxy` Docker network. Traefik routes to it via Gluetun. ARR instances connect to it using the `gluetun` hostname.

See `services/torrent/gluetun.md` for full VPN and network architecture details.

---

## Network

| Setting | Value |
|---------|-------|
| Network mode | `service:gluetun` — shares Gluetun's network namespace |
| WebUI port | 8080 |
| WebUI URL | `https://qbittorrent.giohosted.com` (via Gluetun → Traefik) |
| Network interface | `tun0` (VPN tunnel) — bound in Settings → Advanced |
| Torrenting port | Dynamic — set automatically by Gluetun port forward script |

---

## Download Categories

Categories must match exactly what ARR instances send as the download category.

| Category | Save Path |
|----------|-----------|
| `sonarr-tv` | `/data/downloads/tv` |
| `sonarr-anime` | `/data/downloads/anime` |
| `radarr-1080` | `/data/downloads/movies/1080p` |
| `radarr-4k` | `/data/downloads/movies/4k` |

---

## ARR Download Client Configuration

All ARR instances connect to qBittorrent with these settings:

| Field | Value |
|-------|-------|
| Host | `gluetun` |
| Port | `8080` |
| Username | `admin` |
| Password | see `.env` |

> **Host is `gluetun`, not `qbittorrent`** — qBittorrent shares Gluetun's network namespace. Port 8080 is exposed through Gluetun's container, not qBittorrent's.

| ARR Instance | Category |
|-------------|----------|
| Sonarr TV | `sonarr-tv` |
| Sonarr Anime | `sonarr-anime` |
| Radarr 1080p | `radarr-1080` |
| Radarr 4K | `radarr-4k` |

---

## Key Settings

| Setting | Value | Location |
|---------|-------|----------|
| Default save path | `/data/downloads` | Settings → Downloads |
| Network interface | `tun0` | Settings → Advanced |
| UPnP / NAT-PMP | Disabled | Settings → Connection |
| Port forwarding | Disabled | Settings → Connection |
| Incomplete torrents | Disabled | Settings → Downloads |
| DHT | Enabled | Settings → BitTorrent |
| Peer Exchange | Enabled | Settings → BitTorrent |
| Global upload speed limit | 350,000 KB/s | Settings → Speed |
| Max uploads per torrent | 5 | Restored from v2 |
| Global max seeding time | 60 minutes | Restored from v2 — overridden per category by qBitrr in Phase 5 |

---

## Config File

The config file lives at `/opt/appdata/qbittorrent/qBittorrent/qBittorrent.conf`.

> **Important:** qBittorrent writes its config on shutdown, not on every save. If you need to edit the config file directly while the container is stopped, use `sudo` — the file is owned by the `service` user (UID 2000) and requires elevated permissions for the `gio` user to write.
```bash
docker stop qbittorrent
sudo nano /opt/appdata/qbittorrent/qBittorrent/qBittorrent.conf
docker start qbittorrent
```

Never edit the config file while qBittorrent is running — it will overwrite your changes on shutdown.

---

## Directory Structure
```
/opt/stacks/torrent/
  compose.yaml                         ← in git (shared with Gluetun and qBitrr)
  .env                                 ← gitignored (QBIT_USER, QBIT_PASS)

/opt/appdata/qbittorrent/
  qBittorrent/
    qBittorrent.conf                   ← main config file
    categories.json                    ← download categories
    BT_backup/                         ← torrent resume data
    GeoDB/                             ← peer geolocation data
    logs/                              ← application logs
```

---

## Permissions

qBittorrent runs as PUID/PGID `2000:2000`. The `/data/downloads` NFS mount must be owned by `2000:2000` for qBittorrent to write downloaded files.

The NAS-side ownership was set during v3 build:
```bash
# On nas-prod-01
chown -R 2000:2000 /mnt/user/data/
```

If downloads ever show as errored with 0% progress, check write permissions first:
```bash
sudo -u service touch /data/downloads/tv/test.txt
```
If this fails with permission denied, re-run the chown on the NAS.

---

## Hardlinks

qBittorrent, Sonarr, and Radarr all mount `/data` from the same NFS share. This is required for hardlinks to work — when an ARR instance imports a completed download, it creates a hardlink rather than copying the file. The result is one file on disk with two directory entries — one in `/data/downloads` (for seeding) and one in `/data/media` (for Plex).

Verify hardlinks are working:
```bash
stat /data/downloads/tv/<torrent-file>.mkv
stat /data/media/tv/<show>/<imported-file>.mkv
```
Both should show the same inode number and `Links: 2`.

---

## Troubleshooting

### All torrents errored, 0 download speed
Check write permissions on `/data/downloads` — see Permissions section above.

### WebUI not reachable / Bad Gateway
qBittorrent's WebUI is exposed through Gluetun. Check Gluetun is healthy first:
```bash
docker ps | grep gluetun
docker logs gluetun --tail 20
```

### Listen port not matching forwarded port
Full stack restart forces Gluetun to re-run the port script after qBittorrent is ready:
```bash
cd /opt/stacks/torrent
docker compose down
docker compose up -d
```

Verify port match after restart:
```bash
docker logs gluetun 2>&1 | grep "port forwarded"
docker exec gluetun netstat -ulnp
```
The VPN interface (`10.2.0.x`) should show the same port as the forwarded port in the logs.

### Config changes reverting on restart
qBittorrent writes config on shutdown. Changes made via the WebUI while running are saved correctly. Changes made by editing the file directly must be done while the container is stopped — otherwise qBittorrent overwrites them on exit.

---

## Notes

- qBittorrent is restored from the v2 backup at `/mnt/user/backups/docker-v2/appdata/torrent-stack/qbittorrent/`
- The v2 backup had `Session\DefaultSavePath=/data/torrents` — corrected to `/data/downloads` during v3 setup
- Seeding behavior is managed by qBitrr — do not configure global seeding limits that would interfere with MAM compliance rules (Phase 5)
- Do not change the network interface from `tun0` — this is what binds qBittorrent to the VPN tunnel