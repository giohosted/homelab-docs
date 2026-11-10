# Phase 4 — Service Migration

**Status:** Complete  
**Started:** 2026-03-15  
**Last Updated:** 2026-11-09

> **Historical documentation:** This document records the Homelab v3 service migration performed in March 2026. Deployment versions, original configuration decisions, migration issues, and deferred work are preserved as historical facts. A November 2026 operational follow-up documents subsequent changes verified during the Qbitrr audit. For current service-specific configuration, refer to the corresponding files under `services/`.

---

## Objectives

Migrate all services from v2 docker-host to v3 infrastructure. Run v2 and v3 in parallel until v3 is validated, then cut over DNS and decommission v2.

---

## Entry Criteria

- ✅ Phase 3 complete — Proxmox healthy on both nodes, AdGuard DNS running, docker-prod-01 running with NFS mounts verified
- ✅ Point-to-point DAC link configured (10.0.0.1 ↔ 10.0.0.2)
- ✅ nas-isos NFS storage added to Proxmox

---

## Migration Strategy

Migrate in waves. Infrastructure first, then media stack, then books stack. Each wave validated before the next begins. v2 services remain running on VLAN 20 (192.168.20.x) until DNS rewrites are flipped per-service to v3.

---

## Pre-Work — Data Recovery from Old ZFS Pool

Before service deployment, media/photos/books data needed to be migrated off old ZFS drives from the decommissioned v2 host.

### Recovery VM Setup

- Old ZFS drives connected to JMicron JMB58x SATA controller (PCIe) in pve-prod-01
- IOMMU enabled on pve-prod-01 (`amd_iommu=on iommu=pt`)
- TrueNAS SCALE VM attempted — hung on boot repeatedly due to ZFS pool auto-import behavior. Abandoned.
- Ubuntu 24.04 Server VM (VM ID 105, `ubuntu-recovery`) used instead — lighter, full control over pool import timing
- ZFS tools installed: `apt install zfsutils-linux`
- Old pool imported read-only: `zpool import -f -o readonly=on -d /dev/disk/by-id pool_01`
- Pool: `pool_01`, RAIDZ1, degraded (2 of 3 drives present — third drive not powered), all data readable

### Data Copied to nas-prod-01

| Source | Destination | Size |
|--------|-------------|------|
| `/pool_01/plex/media/movies` | `/mnt/user/media/movies/1080p` | 1.3TB |
| `/pool_01/plex/media/tv` | `/mnt/user/media/tv` | 2.2TB |
| `/pool_01/books/ebooks` | `/mnt/user/media/books/ebooks` | 173MB |
| `/pool_01/books/audiobooks/library` | `/mnt/user/media/books/audiobooks` | 1.9GB |
| `/pool_01/immich` | `/mnt/user/photos` | 104GB |
| `/pool_01/backups/plex` | `/mnt/user/backups/plex` | 2.1GB |
| `/pool_01/backups/docker-v2` | `/mnt/user/backups/docker-v2` | 3.0GB |

All copies verified with `du -sh` on nas-prod-01. Transfer rate ~115 MB/s (1GbE bottleneck on ubuntu-recovery VM).

### Cleanup

- ubuntu-recovery VM (ID 105) shut down and deleted after data verified
- JMicron controller removed from pve-prod-01
- Old ZFS drives disconnected

---

## Wave 1 — Infrastructure Stack ✅ Complete

### Traefik

- **Host:** docker-prod-01 (192.168.30.11)
- **Compose:** `/opt/stacks/traefik/compose.yaml`
- **Appdata:** `/opt/appdata/traefik/`
- **Version at migration:** v3.6.10

#### Configuration

- HTTP → HTTPS redirect on all entrypoints
- Wildcard TLS cert for `*.giohosted.com` via Cloudflare DNS-01 challenge (Let's Encrypt)
- Docker provider: `exposedbydefault=false` — containers opt-in via `traefik.enable=true` label
- File provider: `/opt/appdata/traefik/config/` — used for cross-host static routes
- Dashboard exposed at `traefik.giohosted.com` — restricted to internal VLANs via `local-only` middleware (192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24)
- Cert stored in `/opt/appdata/traefik/acme.json` — auto-renews 30 days before expiry

#### Cross-Host Routing

Traefik routes to services on other VMs via the file provider. Static route configs live in `/opt/appdata/traefik/config/`. Routes documented at migration:

| File | Hostname | Backend |
|------|----------|---------|
| `authentik.yml` | `auth.giohosted.com` | `http://192.168.30.13:9000` |
| `immich.yml` | `photos.giohosted.com` | `http://192.168.30.14:2283` |

#### Issues Encountered

- Chrome Secure DNS (DoH) bypassed AdGuard rewrites — resolved by disabling Chrome's secure DNS setting
- Old Cloudflare wildcard A record (`*` → `69.216.122.32`) caused Chrome DoH to resolve to Cloudflare IPs — deleted all v2 A records from Cloudflare DNS
- Old Cloudflare Tunnel record for `auth` caused connection errors — deleted and recreated in cloudflared v3 setup

---

### Authentik

- **Host:** auth-prod-01 (192.168.30.13)
- **VM ID:** 104
- **OS:** Ubuntu Server 24.04 LTS
- **Resources at migration:** 2 cores, 4GB RAM, 32GB disk (local-zfs, no cache)
- **Compose:** `/opt/stacks/authentik/compose.yaml`
- **Appdata:** `/opt/appdata/authentik/`
- **Version at migration:** 2025.12.3

#### Configuration

- Restored from v2 backup — all OIDC providers, applications, users, and policies intact
- Postgres data restored from `/mnt/user/backups/docker-v2/appdata/authentik/`
- `.env` values (PG_PASS, AUTHENTIK_SECRET_KEY) carried forward from v2 — required to match existing database
- Authentik server exposed on port 9000 — Traefik routes `auth.giohosted.com` → `192.168.30.13:9000` via file provider static route
- No Docker labels on Authentik containers — cross-host routing handled entirely by Traefik file provider

#### Issues Encountered

- Authentik compose initially included Traefik Docker labels and proxy network — removed since Traefik cannot discover containers on a different host via Docker socket
- Old Cloudflare Tunnel record for `auth.giohosted.com` caused browser to receive Cloudflare error 1033 — deleted from Cloudflare DNS

---

### cloudflared

- **Host:** docker-prod-01 (192.168.30.11)
- **Compose:** `/opt/stacks/cloudflared/compose.yaml`
- **Tunnel name:** homelab-v3
- **Image:** cloudflare/cloudflared:latest

#### Exposed Services

| Hostname | Routes to | Notes |
|----------|-----------|-------|
| `auth.giohosted.com` | `https://192.168.30.11` | Authentik via Traefik |
| `audiobooks.giohosted.com` | `https://192.168.30.11` | Audiobookshelf via Traefik — CF Access enabled |
| `request.giohosted.com` | `https://192.168.30.11` | Seerr via Traefik |
| `shelf.giohosted.com` | `https://192.168.30.11` | Shelfmark via Traefik — CF Access enabled |
| `photos.giohosted.com` | `https://192.168.30.11` | Immich via Traefik — CF Tunnel deferred to Phase 5 |

#### Configuration Notes

- All routes use `https://192.168.30.11` with **No TLS Verify** enabled
- No TLS Verify was used because Traefik's certificate is hostname-based (`*.giohosted.com`) — cloudflared connecting by IP failed certificate validation. The original deployment accepted this tradeoff for the private LAN hop.
- All traffic routes through Traefik — cloudflared does not route directly to individual containers

#### Issues Encountered

- Initial setup used `http://192.168.30.11` — caused redirect loop (Traefik redirects HTTP → HTTPS, cloudflared follows, infinite loop)
- Switched to `https://192.168.30.11` with No TLS Verify — resolved

---

### adguardhome-sync

- Deployed in Phase 3 — verified still running in Phase 4
- Syncing dns-prod-01 → dns-prod-02 correctly

---

### Dockman

- **Host:** docker-prod-01 (192.168.30.11)
- **Compose:** `/opt/stacks/dockman/compose.yaml`
- **Appdata:** `/opt/appdata/dockman/`
- **URL:** `https://dockman.giohosted.com`
- **Image:** ghcr.io/ra341/dockman:latest

#### Configuration

- `DOCKMAN_COMPOSE_ROOT` set to `/opt/stacks` — Dockman manages all stacks from this path
- Basic auth enabled via env vars — credentials in `.env` (gitignored)
- Restricted to internal VLANs via `local-only` Traefik middleware — not exposed externally
- Not exposed via Cloudflare Tunnel — internal only

---

## Wave 2 — Media Stack ✅ Complete

### NFS Restructure

Before deploying the ARR stack, the NFS export structure was restructured to support hardlinks. Previously `media` and `downloads` were separate Unraid shares exported individually — this caused docker-prod-01 to receive them as separate NFS mounts, breaking hardlinks across them.

**Fix:** Created a single `data` share on Unraid containing `media/` and `downloads/` as subfolders. Exported `data` as a single NFS share. docker-prod-01 mounts `192.168.30.16:/mnt/user/data` as `/data` via NFSv3 — one filesystem, hardlinks work correctly.

Unraid's Max Server Protocol was changed from NFSv4 to NFSv3. NFSv4's pseudo-root behavior was causing sub-share automounting on top of the parent mount, which recreated the separate-filesystem problem even with a single fstab entry.

See `architecture/decisions-log.md` for full rationale.

**November 2026 verification:** The shared `/data` NFS architecture remained in use during the Qbitrr and Shelfmark audit. An isolated stale-file-handle problem affected Shelfmark, but the host mount and other containers remained accessible. No NFS architecture change was required.

### Plex

- **Host:** nas-prod-01 (Docker container, bound to 192.168.30.2)
- **DB restored from:** `/mnt/user/backups/plex/db/plex-db-2026-03-10.tar.gz`
- **Libraries:** Movies (`/data/media/movies`), TV Shows (`/data/media/tv`), Anime (`/data/media/anime`)
- **Hardware transcoding:** Intel Raptor Lake-S UHD Graphics (QuickSync) — confirmed active
- **HEVC encoding:** Set to "HEVC sources only"
- **Plex container media path updated:** `/mnt/user/data` → `/data` (updated from `/mnt/user` after NFS restructure)

#### Issues Encountered

- Permissions error on DB restore — extracted files owned by root, Plex runs as 2000:2000. Fixed with `chown -R 2000:2000` on restored files.

### ARR Stack

- **Host:** docker-prod-01 (192.168.30.11)
- **Compose:** `/opt/stacks/arr/compose.yaml`
- **All containers:** PUID=2000, PGID=2000, TZ=America/Chicago
- **Network:** proxy (external, shared with Traefik)
- **Middleware:** `local-only@docker` on all internal services

| Container | Image | Port | URL | Config Source |
|-----------|-------|------|-----|---------------|
| sonarr-tv | lscr.io/linuxserver/sonarr:latest | 8989 | sonarr-tv.giohosted.com | Restored from v2 backup |
| sonarr-anime | lscr.io/linuxserver/sonarr:latest | 8989 | sonarr-anime.giohosted.com | Fresh |
| radarr-1080p | lscr.io/linuxserver/radarr:latest | 7878 | radarr-1080p.giohosted.com | Restored from v2 backup |
| radarr-4k | lscr.io/linuxserver/radarr:latest | 7878 | radarr-4k.giohosted.com | Fresh |
| prowlarr | lscr.io/linuxserver/prowlarr:latest | 9696 | prowlarr.giohosted.com | Restored from v2 backup |
| bazarr | lscr.io/linuxserver/bazarr:latest | 6767 | bazarr.giohosted.com | Restored from v2 backup |
| profilarr | santiagosayshey/profilarr:latest | 6868 | profilarr.giohosted.com | Restored from v2 backup |
| maintainerr | ghcr.io/maintainerr/maintainerr:latest | 6246 | maintainerr.giohosted.com | Restored from v2 backup |
| seerr | ghcr.io/seerr-team/seerr:latest | 5055 | request.giohosted.com | Restored from v2 backup |
| tautulli | lscr.io/linuxserver/tautulli:latest | 8181 | tautulli.giohosted.com | Restored from v2 backup |
| flaresolverr | ghcr.io/flaresolverr/flaresolverr:latest | 8191 | Internal only | Restored from v2 backup |

#### Root Folders

| Container | Root Folder |
|-----------|-------------|
| sonarr-tv | `/data/media/tv` |
| sonarr-anime | `/data/media/anime` |
| radarr-1080p | `/data/media/movies/1080p` |
| radarr-4k | `/data/media/movies/4k` |

#### Prowlarr Applications

| App | URL |
|-----|-----|
| Sonarr TV | `http://sonarr-tv:8989` |
| Sonarr Anime | `http://sonarr-anime:8989` |
| Radarr 1080p | `http://radarr-1080p:7878` |
| Radarr 4K | `http://radarr-4k:7878` |

#### Bazarr

- Connected to Sonarr TV and Radarr 1080p only — Bazarr 1.5.6 supports only one Sonarr and one Radarr instance
- Sonarr Anime and Radarr 4K have no Bazarr coverage — acceptable, anime subtitles typically embedded
- Connected to Plex at 192.168.30.2:32400 for subtitle notifications

#### Issues Encountered

- `local-only@file` middleware reference in ARR compose — should be `local-only@docker`. Middleware is defined in Traefik container labels, not a file provider. Fixed with `sed`.
- Maintainerr crash-looping — v2.0.0+ runs as internal `node` user (UID 1000), not root. Fixed with `chown -R 1000:1000` on appdata.
- Seerr crash-looping — v3 runs internally as `node:node` (UID 1000:1000). v2 appdata was owned by `service` user (UID 2000). Fixed with `chown -R 1000:1000` on appdata.
- Seerr repeatedly falling off proxy network — caused by crash-loop. Resolved once permissions fixed and container stabilized.

---

## Wave 3 — Torrent Stack ✅ Complete

### Stack Overview

- **Host:** docker-prod-01 (192.168.30.11)
- **Compose:** `/opt/stacks/torrent/compose.yaml`
- **Containers:** Gluetun, qBittorrent, qBitrr

### Gluetun

- **Image:** qmcgaw/gluetun:latest
- **Role:** VPN killswitch container — all qBittorrent traffic routes through ProtonVPN WireGuard tunnel
- **VPN provider:** ProtonVPN — Chicago P2P servers (`SERVER_CITIES=Chicago`)
- **Port forwarding:** Enabled (`VPN_PORT_FORWARDING=on`, `PORT_FORWARD_ONLY=on`) — Gluetun automatically obtains a forwarded port from ProtonVPN and runs a script to update qBittorrent's listen port
- **Port script:** `/opt/appdata/gluetun/scripts/qbit-set-port.sh` — logs into qBittorrent WebUI via API and sets the listen port to match the VPN forwarded port
- **Network:** On `proxy` network — Traefik routes `qbittorrent.giohosted.com` via Gluetun (Traefik labels live on Gluetun, not qBittorrent)

### qBittorrent

- **Image:** lscr.io/linuxserver/qbittorrent:latest
- **Version at migration:** 5.1.4
- **Version verified November 2026:** 5.3.0
- **Network:** `network_mode: service:gluetun` — shares Gluetun's network namespace entirely. All traffic goes through VPN tunnel.
- **WebUI:** Port 8080 — accessible via `https://qbittorrent.giohosted.com` through Gluetun → Traefik
- **PUID/PGID:** 2000:2000
- **Config restored from:** `/mnt/user/backups/docker-v2/appdata/torrent-stack/qbittorrent/`
- **Network interface:** `tun0` (VPN tunnel) — bound in Settings → Advanced
- **Default save path:** `/data/downloads`

#### Download Categories

| Category | Save Path |
|----------|-----------|
| `sonarr-tv` | `/data/downloads/tv` |
| `sonarr-anime` | `/data/downloads/anime` |
| `radarr-1080` | `/data/downloads/movies/1080p` |
| `radarr-4k` | `/data/downloads/movies/4k` |
| `ebooks` | `/data/downloads/books/ebooks/downloads` |
| `audiobooks` | `/data/downloads/books/audiobooks/downloads` |

#### Download Clients (ARR instances)

| ARR Instance | Category |
|--------------|----------|
| Sonarr TV | `sonarr-tv` |
| Sonarr Anime | `sonarr-anime` |
| Radarr 1080p | `radarr-1080` |
| Radarr 4K | `radarr-4k` |

All ARR instances connect to qBittorrent via host `gluetun` port `8080` — not `qbittorrent`, because qBittorrent shares Gluetun's network namespace and the port is exposed through Gluetun.

#### Post-Migration Seeding Change

At deployment, qBittorrent had a native global 60-minute seeding stop rule.

That rule was disabled during the November 2026 audit. qBitrr now manages time-based torrent cleanup for the four ARR categories and both book categories.

This prevents the global qBittorrent rule from interfering with the intended private-tracker-specific MAM requirements.

See `services/torrent/qbittorrent.md` for current settings.

### qBitrr

- **Image:** feramance/qbitrr:latest
- **Version at migration:** 5.10.1
- **Version verified November 2026:** 5.14.9-1
- **Role:** Manages all four ARR instances — torrent health monitoring, automated search, failed-download recovery, and seeding policies
- **WebUI:** `https://qbitrr.giohosted.com/ui` — port 6969
- **Config:** `/opt/appdata/qbitrr/config.toml`
- **Managed instances:** Sonarr-TV, Sonarr-Anime, Radarr-1080, Radarr-4K
- **ManagedCategories:** `audiobooks`, `ebooks` — qBitrr manages these categories at the qBittorrent level
- **OIDC:** Configured — Authentik provider `qbitrr`, callback path `/signin-oidc`. Both `https://` and `http://` redirect URIs registered in Authentik. Local auth retained as break-glass.
- **Seeding rules at migration:** Not yet configured — MAM-specific policies were intentionally deferred to Phase 5.
- **Seeding rules by November 2026:** Implemented for all four ARR instances and both book categories, with a separate MAM tracker override. Live MAM compliance verification remains outstanding.

#### Current Seeding Policy Reference

As verified November 9, 2026:

| Managed Content | Default Seeding Policy | Status |
|-----------------|------------------------|--------|
| Sonarr-TV | 60 minutes | Configured |
| Sonarr-Anime | 60 minutes | Configured |
| Radarr-1080 | 60 minutes | Configured |
| Radarr-4K | 60 minutes | Configured |
| Public ebooks | 60 minutes | Configured |
| Public audiobooks | 60 minutes | End-to-end verified |
| MAM ebooks and audiobooks | Minimum 14 days AND ratio 1.0 | Configured; live verification pending |

The active configuration uses `RemoveTorrent = 2` and `MaxSeedingTime = 3600` for the default category and ARR policies.

The MAM override uses the `[[qBit.Trackers]]` syntax with `HitAndRunMode = "and"`.

See `services/torrent/qbitrr.md` for full configuration, operational procedures, and verification boundaries.

### Issues Encountered

| Issue | Resolution |
|-------|------------|
| qBittorrent listen port not matching Gluetun forwarded port | Port script ran before qBittorrent finished binding. Fixed by full stack restart (`docker compose down && up`) so qBittorrent starts fresh and Gluetun re-runs port script |
| Default save path reverting to `/data/torrents` on restart | Restored backup had old path hardcoded. Required `sudo sed` to edit config file directly since files owned by service user (2000) — `gio` user has read-only access |
| All torrents errored, 0 download speed | `/data/downloads` owned by `99:users` (Unraid NFS default) — service user (2000) not in `users` group. Fixed with `chown -R 2000:2000 /data/` on docker-prod-01 and `chown -R 2000:2000 /mnt/user/data/` on nas-prod-01 |
| Compose files owned by root, Dockman autosave failing | Files created with `sudo nano` are root-owned. Fixed with `sudo chown gio:service` and `sudo chmod 664` on all compose files. See `runbooks/new-stack-checklist.md` |
| AdGuard DNS rewrite for qbittorrent.giohosted.com disappeared | Browser DNS cache had stale NXDOMAIN — flushed with `ipconfig /flushdns` |

The table above records migration-time problems and corrective actions. These are not instructions to repeat those changes during future incidents without first inspecting the current state.

### DNS Rewrites Added

| Hostname | IP |
|----------|----|
| `qbittorrent.giohosted.com` | 192.168.30.11 |
| `qbitrr.giohosted.com` | 192.168.30.11 |

### Decisions Made

| Decision | Rationale |
|----------|-----------|
| Traefik labels on Gluetun, not qBittorrent | qBittorrent shares Gluetun's network namespace — it has no independent network identity. Traefik can only route to Gluetun. |
| ARR instances connect to qBit via `gluetun` hostname | Same reason — port 8080 is exposed through Gluetun's container, not qBittorrent's |
| qBitrr seeding rules deferred to Phase 5 | Seeding rules (MAM compliance, ratio limits) were deferred until book/audiobook torrents were active. This was a migration-time decision; the rules were subsequently configured. |
| Port forward script on Gluetun side | Gluetun obtains a new forwarded port on every VPN connection. Script updates qBittorrent dynamically via API so listen port always matches forwarded port. |

---

## Wave 4 — Books Stack ✅ Complete

### Stack Overview

- **Host:** docker-prod-01 (192.168.30.11)
- **Compose:** `/opt/stacks/books/compose.yaml`
- **Containers:** Audiobookshelf, Calibre-Web-Automated, Shelfmark
- **All containers:** PUID=2000, PGID=2000, TZ=America/Chicago
- **Network:** proxy (external, shared with Traefik)

### Audiobookshelf

- **Image:** ghcr.io/advplyr/audiobookshelf:latest
- **URL:** `https://audiobooks.giohosted.com`
- **External access:** Yes — Cloudflare Tunnel + Cloudflare Access
- **Appdata restored from:** `/mnt/user/backups/docker-v2/appdata/books-stack/audiobookshelf/`
- **Volumes:**
  - `/data/media/books/audiobooks` → `/audiobooks`
  - `/opt/appdata/audiobookshelf/config` → `/config`
  - `/opt/appdata/audiobookshelf/metadata` → `/metadata`
- **OIDC:** Configured — Authentik provider carried forward from v2 backup. Authentik group: `admins`.
- **Library path:** `/audiobooks` — had to repoint after restore (v2 was pointing to `/books/audiobooks`)

### Calibre-Web-Automated

- **Image:** crocodilestick/calibre-web-automated:latest
- **URL:** `https://cwa.giohosted.com`
- **External access:** No — internal only (`local-only@docker` middleware)
- **Appdata restored from:** `/mnt/user/backups/docker-v2/appdata/books-stack/cwa/`
- **Volumes:**
  - `/opt/appdata/calibre-web-automated/config` → `/config`
  - `/opt/appdata/calibre-web-automated/plugins` → `/config/.config/calibre/plugins`
  - `/data/downloads/books/ebooks/ingest` → `/cwa-book-ingest`
  - `/data/media/books/ebooks` → `/calibre-library`
- **Key env vars:** `NETWORK_SHARE_MODE=true`, `CWA_PORT_OVERRIDE=8083`, `HARDCOVER_TOKEN` in `.env`
- **OIDC:** Authentik OIDC carried forward from v2 backup — working on restore

### Shelfmark

- **Image:** ghcr.io/calibrain/shelfmark:latest
- **URL:** `https://shelf.giohosted.com`
- **External access:** Yes — Cloudflare Tunnel + Cloudflare Access
- **Appdata restored from:** `/mnt/user/backups/docker-v2/appdata/books-stack/shelfmark/`
- **Volumes verified November 2026:**
  - `/opt/appdata/shelfmark/config` → `/config`
  - `/data/downloads/books/ebooks/ingest` → `/books`
  - `/data/media/books/audiobooks` → `/audiobooks`
  - `/data/downloads/books/audiobooks/downloads` → `/data/downloads/books/audiobooks/downloads`
  - `/data/downloads/books/ebooks/downloads` → `/data/downloads/books/ebooks/downloads`
- **OIDC:** Authentik provider created fresh — Client ID `shelfmark`, callback `https://shelf.giohosted.com/api/auth/oidc/callback`. Authentik group: `admins`.
- **qBit connection:** `http://gluetun:8080/` — updated from v2 value (`http://192.168.20.10:8080/`)
- **Prowlarr connection:** `http://prowlarr:9696/` — updated from v2 value

**Documentation correction — November 9, 2026:** The original Phase 4 record incorrectly listed the audiobook download mount destination as `/books/audiobooks/downloads`. Inspection of the running Shelfmark container confirmed the actual destination is `/data/downloads/books/audiobooks/downloads`. The table above reflects that verified configuration.

#### Ebook Hardlink Workflow

Shelfmark sends ebook torrents to qBittorrent with category `ebooks`.

1. qBittorrent downloads to `/data/downloads/books/ebooks/downloads`.
2. Shelfmark hardlinks the completed ebook to `/books`, mapped to `/data/downloads/books/ebooks/ingest`.
3. CWA detects the file in the ingest directory.
4. CWA imports the ebook to `/data/media/books/ebooks`.
5. CWA may remove the ingest-side hardlink after processing.
6. qBitrr manages the original torrent's seeding period and eventual cleanup.

At the original March 2026 deployment, the torrent was intended to remain seeding indefinitely. That policy was replaced by the Qbitrr-managed one-hour default in November 2026.

> **Why the extra volume mount matters:** Shelfmark must have both the source (`/data/downloads/books/ebooks/downloads`) and destination (`/books`) mounted and visible at the expected paths for hardlinks to work. The source volume is mounted as `/data/downloads/books/ebooks/downloads:/data/downloads/books/ebooks/downloads`.

> **Why the ingest hardlink is intentional:** CWA's ingest folder is a staging location, not the final library. Removing the ingest-side hardlink does not inherently remove a separately linked original download file. The final library import must nevertheless complete successfully before torrent cleanup can be considered harmless.

**November 2026 test:** A direct EPUB download succeeded and appeared in CWA after Shelfmark's NFS issue was resolved. A full torrent-based ebook import followed by torrent cleanup remains unverified.

#### Audiobook Hardlink Workflow

Shelfmark sends audiobook torrents to qBittorrent with category `audiobooks`.

1. qBittorrent downloads to `/data/downloads/books/audiobooks/downloads`.
2. Shelfmark hardlinks the completed audiobook to `/audiobooks`, mapped to `/data/media/books/audiobooks`.
3. ABS detects the imported audiobook and makes it available for playback.
4. qBitrr manages the torrent's seeding period and eventual cleanup.
5. The imported audiobook remains available after removal of the original torrent when the library entry has been retained.

At migration, indefinite seeding was the intended policy. By November 2026, public audiobook torrents used Qbitrr-managed time-based cleanup.

**November 2026 test:** A public audiobook downloaded through Shelfmark, appeared in ABS, played successfully, and remained playable after qBitrr removed the torrent at approximately the configured one-hour seeding limit.

See `services/books/shelfmark.md` for the complete verified workflow.

### Issues Encountered

| Issue | Resolution |
|-------|------------|
| Shelfmark settings page timing out on load | v2 config had stale qBit URL (`http://192.168.20.10:8080/`) and Prowlarr URL — UI timed out trying to reach them before page could render. Fixed by editing `/opt/appdata/shelfmark/config/plugins/prowlarr_clients.json` and `prowlarr_config.json` directly. |
| ABS library visible but files not playable | Library path was still pointing to v2 path (`/books/audiobooks`). Repointed to `/audiobooks` in ABS Settings → Libraries. |
| qBitrr OIDC settings not persisting via UI | Settings UI did not reliably write to config file. Fixed by editing `/opt/appdata/qbitrr/config.toml` directly and restarting container. |
| qBitrr OIDC redirect URI mismatch | Authentik had `https://` but qBitrr was sending `http://` redirect URI. Fixed by adding `http://qbitrr.giohosted.com/signin-oidc` as a second allowed redirect URI in Authentik provider. |

### DNS Rewrites Added

| Hostname | IP |
|----------|----|
| `audiobooks.giohosted.com` | 192.168.30.11 |
| `cwa.giohosted.com` | 192.168.30.11 |
| `shelf.giohosted.com` | 192.168.30.11 |

### Decisions Made

| Decision | Rationale |
|----------|-----------|
| Shelfmark ebook destination = ingest folder, not library | CWA watches ingest and imports to library automatically. Shelfmark should not write directly to the library. |
| Hardlink toggle ON for ebooks despite CWA warning | Ingest is a staging folder — CWA can remove the ingest-side hardlink without inherently removing the original download. The final CWA import still needs to complete successfully. |
| Extra volume mount for ebook downloads path | Shelfmark requires both source and destination to be visible at expected paths. Without the correct source mount, imports and hardlinks can fail. |
| qBitrr ManagedCategories = audiobooks + ebooks only | ARR instances are managed via their own `[Sonarr-*]`/`[Radarr-*]` sections. ManagedCategories provides qBit-level management for book categories. |
| MAM seeding rules deferred to Phase 5 | Tracker-scoped seeding rules were not required to complete the initial migration. They were subsequently configured and revised during the November 2026 audit. |
| OIDC admin group = `admins` for both ABS and Shelfmark | `shelfmark-admins` group does not exist in Authentik. Reusing existing `admins` group avoids creating redundant groups. Consistent with other services. |

---

## Wave 5 — Photos Stack ✅ Complete

### Stack Overview

- **Host:** immich-prod-01 (192.168.30.14) — dedicated VM on pve-prod-02
- **Compose:** `/opt/stacks/immich/compose.yaml`
- **Containers:** immich-server, immich-machine-learning, immich-postgres, immich-redis
- **Version at migration:** v2.5.6

### immich-prod-01 VM

| Component | Detail |
|-----------|--------|
| **VM ID** | 106 |
| **Host** | pve-prod-02 |
| **OS** | Ubuntu Server 24.04 LTS |
| **CPU** | 2 cores (x86-64-v2-AES) |
| **RAM** | 6GB |
| **Disk** | 32GB (local-lvm) |
| **IP** | 192.168.30.14/24 |
| **VLAN** | 30 — Services |
| **Docker** | Installed via official script |
| **QEMU Agent** | Installed and running |

> **Node placement:** immich-prod-01 runs on pve-prod-02, not pve-prod-01 as originally planned. pve-prod-01 was already running docker-prod-01 (8GB RAM) and auth-prod-01 (4GB RAM). pve-prod-02 had ample headroom (only PBS VM + AdGuard LXC). Moving Immich to pve-prod-02 also isolates the ML worker CPU spikes from the primary media stack.

### Immich

- **Image:** ghcr.io/immich-app/immich-server:release
- **URL:** `https://photos.giohosted.com`
- **External access at migration:** Deferred to Phase 5 — Cloudflare Tunnel + CF Access to be configured
- **NFS mount:** `192.168.30.16:/mnt/user/photos` → `/data/photos` on immich-prod-01
- **Volumes:**
  - `/data/photos` → `/data` (upload location — photos, thumbs, backups)
  - `/opt/appdata/immich/postgres` → `/var/lib/postgresql/data` (local disk — NFS not supported for Postgres)
- **OIDC:** OAuth via Authentik — carried forward from v2 backup
- **Database:** Restored from Immich auto-backup (`immich-db-backup-20260310T020000-v2.5.6-pg14.19.sql.gz`) found at `/data/photos/backups/`
- **Photos:** ~10,000 assets — thumbnails regenerated after restore, all photos loading correctly

#### Traefik Routing

Immich is on a dedicated VM — Traefik on docker-prod-01 routes via file provider:

- **Config file:** `/opt/appdata/traefik/config/immich.yml`
- **Route:** `photos.giohosted.com` → `http://192.168.30.14:2283`
- Port 2283 exposed on host via ports mapping in compose (required for cross-host file provider routing)

### Issues Encountered

| Issue | Resolution |
|-------|------------|
| nas-isos NFS mount broken on pve-prod-02 | nas-isos was configured to use `10.0.0.2` (point-to-point DAC link) — only accessible from pve-prod-01. Fixed by adding a separate `nas-isos-p2` storage entry on pve-prod-02 pointing to `192.168.30.16`. Original `nas-isos` entry scoped to pve-prod-01 only. |
| docker-prod-01 RAM at 100% (1GB in swap) | Bumped from 6GB to 8GB. Verified with `free -h` before and after — swap cleared after restart. |
| Immich maintenance mode on first boot | Fresh database had no schema — Immich entered maintenance mode. Bypassed by navigating to the maintenance page using a token from logs, then cancelling restore and creating admin user. |
| Database restore via UI failed silently | Immich UI restore was interrupted mid-migration. Database left empty. Fixed by running restore manually: `gunzip -c /data/photos/backups/<backup>.sql.gz \| docker exec -i immich_postgres psql -U postgres -d immich` |
| Photos visible but "error loading image" | Database paths stored as `/data/upload/...` (v2 path) didn't match new container mount at `/data/upload/...`. After SQL path migration and compose fix, paths now correctly use `/data/upload/...` matching the official Immich compose volume mapping. |
| Immich compose volume path mismatch | Initially used non-standard `/usr/src/app/upload` internal path instead of official `/data`. Fixed compose to use official `${UPLOAD_LOCATION}:/data` mapping and ran SQL to revert asset paths back to `/data/upload/...` format. |

### DNS Rewrites Added

| Hostname | IP |
|----------|----|
| `photos.giohosted.com` | 192.168.30.11 |

### Decisions Made

| Decision | Rationale |
|----------|-----------|
| immich-prod-01 on pve-prod-02, not pve-prod-01 | pve-prod-01 RAM headroom was tight (docker-prod-01 was swapping). pve-prod-02 had ample headroom. ML worker CPU spikes also better isolated from primary media stack. |
| nas-isos dual storage entries (one per node) | nas-isos was configured to use the DAC link (10.0.0.2) which only pve-prod-01 can reach. Added separate `nas-isos-p2` entry for pve-prod-02 using 192.168.30.16. Each node's storage scoped to that node only to avoid mount failures. |
| Postgres data on local VM disk, not NFS | Immich explicitly does not support NFS for the database. Postgres data lives at `/opt/appdata/immich/postgres` on the VM's local-lvm disk. |
| DB restore via psql CLI, not Immich UI | Immich UI restore was silently failing mid-migration. Direct psql restore is reliable and bypasses the UI entirely. |
| Use official Immich compose volume mapping (`:/data`) | Non-standard internal paths cause DB asset path mismatches on future migrations. Using the official mapping ensures consistency with Immich's own tooling and makes future restores straightforward. |
| CF Tunnel + CF Access for Immich deferred to Phase 5 | Immich is accessible internally at `photos.giohosted.com`. External access and access policy configuration deferred to Phase 5 alongside other hardening tasks. |

---

## DNS Rewrites Added (AdGuard)

| Hostname | IP | Notes |
|----------|----|-------|
| `traefik.giohosted.com` | 192.168.30.11 | Traefik dashboard — internal only |
| `auth.giohosted.com` | 192.168.30.11 | Authentik via Traefik |
| `dockman.giohosted.com` | 192.168.30.11 | Dockman — internal only |
| `sonarr-tv.giohosted.com` | 192.168.30.11 | Sonarr TV |
| `sonarr-anime.giohosted.com` | 192.168.30.11 | Sonarr Anime |
| `radarr-1080p.giohosted.com` | 192.168.30.11 | Radarr 1080p |
| `radarr-4k.giohosted.com` | 192.168.30.11 | Radarr 4K |
| `prowlarr.giohosted.com` | 192.168.30.11 | Prowlarr |
| `bazarr.giohosted.com` | 192.168.30.11 | Bazarr |
| `profilarr.giohosted.com` | 192.168.30.11 | Profilarr |
| `maintainerr.giohosted.com` | 192.168.30.11 | Maintainerr |
| `request.giohosted.com` | 192.168.30.11 | Seerr |
| `tautulli.giohosted.com` | 192.168.30.11 | Tautulli |
| `qbittorrent.giohosted.com` | 192.168.30.11 | qBittorrent |
| `qbitrr.giohosted.com` | 192.168.30.11 | qBitrr |
| `audiobooks.giohosted.com` | 192.168.30.11 | Audiobookshelf |
| `cwa.giohosted.com` | 192.168.30.11 | Calibre-Web-Automated |
| `shelf.giohosted.com` | 192.168.30.11 | Shelfmark |
| `photos.giohosted.com` | 192.168.30.11 | Immich |

---

## Cloudflare DNS Changes

| Record | Action | Reason |
|--------|--------|--------|
| `*` A record → 69.216.122.32 | Deleted | v2 NPM wildcard — no longer needed |
| `giohosted.com` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `photos` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `auth` Tunnel record → homelab-v2 | Deleted | Dead v2 tunnel — recreated in homelab-v3 tunnel |

---

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| Ubuntu recovery VM instead of TrueNAS VM | TrueNAS hung on boot repeatedly due to ZFS pool auto-import. Ubuntu with manual zpool import is lighter and more controllable. |
| Authentik on dedicated VM (auth-prod-01) | Per roadmap — LXC officially removed from Proxmox community scripts May 2025. VM is the only stable option. |
| Traefik file provider for cross-host routing | Traefik Docker provider can only discover containers on the same host. File provider handles static routes to other VMs by IP. |
| Authentik secrets carried forward from v2 | PG_PASS and AUTHENTIK_SECRET_KEY must match the existing database — changing them would break the restore. |
| Chrome Secure DNS disabled on workstation | Required for AdGuard DNS rewrites to work — DoH bypasses local DNS entirely. |
| cloudflared routes to Traefik, not containers directly | Single routing layer — all traffic goes through Traefik regardless of which VM the service lives on. Simpler, more consistent. |
| No TLS Verify on cloudflared routes | Traefik cert is hostname-based — IP connections fail cert validation. This verification tradeoff was accepted for the private-LAN origin connection during migration. |
| Homarr removed from scope | No longer needed — Dockman covers compose management, individual service UIs cover operations. |
| Beszel + Uptime Kuma moved to Phase 5 | Monitoring tools belong in the hardening/operations phase, not service migration. |
| Single `data` NFS share instead of separate `media` and `downloads` | Hardlinks require files to be on the same filesystem. Separate NFS mounts — even from the same NAS — are separate filesystems. Single share with subfolders is the correct pattern per TRaSH guide. |
| NFSv3 instead of NFSv4 | NFSv4 pseudo-root behavior caused sub-share automounting on top of the parent mount, recreating the separate-filesystem problem. NFSv3 does not have this behavior. |
| Bazarr covers Sonarr TV and Radarr 1080p only | Bazarr supports one Sonarr and one Radarr instance. Anime subtitles are typically embedded — no meaningful loss from excluding Sonarr Anime and Radarr 4K. |
| Seerr appdata ownership set to 1000:1000 | Seerr v3 runs internally as node:node (UID 1000). v2 appdata was owned by service user (2000) — mismatch caused crash-loop. |

---

## November 2026 — Post-Migration Operational Follow-Up

**Review date:** November 7–9, 2026  
**Scope:** qBitrr, qBittorrent, automated media searches, torrent health and recovery, seeding policies, database reliability, and Shelfmark integration.

This review took place after completion of Phase 4. It does not change the original migration completion status.

Detailed operational findings are maintained in:

- `services/torrent/qbitrr.md`
- `services/torrent/qbittorrent.md`
- `services/books/shelfmark.md`

### qBitrr and qBittorrent Versions

| Service | March 2026 Migration | November 2026 Verified |
|---------|----------------------|------------------------|
| qBittorrent | 5.1.4 | 5.3.0 |
| qBitrr | 5.10.1 | 5.14.9-1 |

Both continue to use the existing torrent stack on `docker-prod-01`.

### Seeding Policy Optimization

The historical qBittorrent native global 60-minute seeding rule was disabled.

qBitrr became the central manager for the configured ARR and book-category seeding policies.

**Current default policy:**

```toml
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

This configuration was verified for:

- Sonarr-TV
- Sonarr-Anime
- Radarr-1080
- Radarr-4K
- `ebooks`
- `audiobooks`

The book categories are managed under:

```toml
[qBit.CategorySeeding]
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

The default public-torrent behavior is approximately one hour of seeding followed by automatic cleanup when applicable.

### MAM Tracker Override

The former MAM tracker configuration format was replaced with `[[qBit.Trackers]]`.

The current verified settings are:

```toml
[[qBit.Trackers]]
Name = "MyAnonamouse"
URI = "myanonamouse.net"
Priority = 10
MaxUploadRatio = 1.0
MaxSeedingTime = 1209600
HitAndRunMode = "and"
MinSeedRatio = 1.0
MinSeedingTimeDays = 14
TrackerUpdateBuffer = 3600
AddTrackerIfMissing = false
RemoveIfExists = false
```

The policy is intended to protect MAM torrents until both the minimum 14-day seeding duration and 1.0 upload ratio have been satisfied.

**Verification status:** Configuration parsing and Qbitrr worker startup passed. Live MAM tracker matching, continued seeding beyond the default one-hour threshold, and final HnR clearance have not yet been verified using a real MAM torrent.

Do not treat the policy as fully validated until that behavior is observed.

### Automated Search and Monitoring

All four ARR instances were verified with:

| Setting | Value |
|---------|-------|
| `SearchMissing` | `true` |
| `SearchLimit` | `5` |
| `SearchRequestsEvery` | `300` |
| `SearchAgainOnSearchCompletion` | `true` |
| `SearchByYear` | `true` |
| `SearchInReverse` | `false` |
| `Unmonitored` | `false` |
| `DoUpgradeSearch` | `false` |
| `QualityUnmetSearch` | `false` |
| `CustomFormatUnmetSearch` | `false` |
| `RefreshDownloadsTimer` | `1` |
| `ReSearch` | `true` |

Sonarr uses `SearchBySeries = "smart"` and `PrioritizeTodaysReleases = true`.

The search configuration was retained after review. No confirmed excessive indexer-search traffic was identified.

Earlier unwanted anime searches were traced to monitoring configuration, and the affected Sonarr-Anime monitoring settings were corrected.

### Torrent Health and Recovery

A stalled Radarr-1080 torrent was repeatedly detected while the old `RemoveTorrent = -1` policy prevented removal.

The removal restriction was corrected by using the intended time-based removal configuration.

Other invalid or unavailable episode torrents were removed and blocklisted during the audit.

**Verification distinction:**

- Stalled-torrent detection was observed.
- The previous removal restriction was identified and corrected.
- Removal of invalid torrents was observed.
- A new stalled-torrent event completing the full automated replacement workflow remains pending real-world verification.

### File Filtering Correction

Twelve incorrectly escaped regex expressions were corrected across the four ARR instances.

The affected patterns included numbered anime opening and ending extras such as `NCOP12` and `NCED05`.

All 24 post-correction validation tests passed.

The exclusion configuration continues to protect against unwanted samples, extras, and disallowed file extensions.

### Orphaned Database Record Cleanup

The audit identified 384 stale Sonarr-Anime episode records in Qbitrr's SQLite database.

A pre-cleanup backup was created:

```text
/opt/appdata/qbitrr/backups/qbitrr-pre-cleanup-20261108T233539Z.db
```

The targeted records were removed, and the resulting database state was verified.

Sixteen leftover Bleach media files, totaling approximately 17 GB, were also reviewed and removed with explicit authorization.

The suspected underlying search-state or reset issue was not conclusively proven. Monitor for recurrence.

### Shelfmark NFS Incident

Shelfmark experienced download and import failures while its bind-mounted directories returned `Stale file handle`.

Investigation confirmed:

- The host's `/data` NFS mount remained accessible.
- Other relevant containers could list their directories.
- Shelfmark alone returned stale file handles for the affected mounted directories.
- Restarting only Shelfmark restored access.

No NFS protocol, share, or bind-mount changes were required.

The event was treated as an isolated resolved incident. Its initiating cause remains unknown.

### Download and Import Verification

After restoring Shelfmark access:

- A direct EPUB download was successfully imported into CWA.
- A public audiobook torrent was successfully downloaded.
- Shelfmark imported the audiobook into ABS.
- ABS played the audiobook while the torrent was seeding.
- qBitrr removed the public torrent after approximately one hour of seeding.
- The audiobook remained present and playable in ABS.

**Result:** The public audiobook torrent-to-library workflow was verified end-to-end.

The MAM and torrent-based ebook workflows remain separately pending validation.

### Qbitrr Performance and Reliability

| Component | Observed Result |
|-----------|-----------------|
| CPU usage | 3.45% |
| Memory usage | 368.5 MiB |
| Process/thread count | 36 |
| Worker initialization | 13 workers started successfully |
| Restart policy | `unless-stopped` |
| OOM killed | `false` |
| Docker healthcheck | Not configured |
| Docker log driver | `json-file` |
| Explicit per-container log rotation | Not configured |
| Docker log size | Approximately 7.3 MB |
| SQLite integrity | `ok` |
| SQLite WAL checkpoint interval | 5 minutes |
| WAL checkpoint errors | None observed in supplied maintenance logs |

The measurements did not justify additional CPU, memory, polling, or database-maintenance changes.

**Optional future improvement:** Configure bounded Docker log rotation during a planned container maintenance change. No immediate action was needed.

### Outstanding Verification

- Confirm that a live MAM torrent remains protected beyond the public 60-minute seeding threshold.
- Confirm that MAM HnR clearance requires both 14 days and ratio 1.0.
- Observe a newly stalled torrent completing Qbitrr's removal and replacement workflow.
- Verify a torrent-based ebook import and cleanup end-to-end.
- Monitor for recurrence of stale Shelfmark NFS handles or orphaned Qbitrr search records.

These are operational validation items, not incomplete Phase 4 migration tasks.

---

## Exit Criteria

The following checklist reflects the original Phase 4 completion assessment. Items deferred to Phase 5 are retained in their historical state.

- ✅ All services running on v3 infrastructure
- ⬜ External access working via CF Tunnel (Immich CF Tunnel pending Phase 5)
- ⬜ PBS backing up all VMs nightly
- ⬜ Backup scripts running with Healthchecks heartbeats
- ✅ Wave 1 complete — Traefik, Authentik, cloudflared, adguardhome-sync, Dockman deployed
- ✅ Wave 2 complete — Plex restored, full ARR stack deployed, NFS restructured for hardlinks
- ✅ Wave 3 complete — Gluetun, qBittorrent, qBitrr deployed, VPN killswitch verified
- ✅ Wave 4 complete — Audiobookshelf, CWA, Shelfmark deployed, hardlink workflows configured, OIDC configured for all three + qBitrr
- ✅ Wave 5 complete — Immich deployed on immich-prod-01 (pve-prod-02), DB restored, photos loading, OAuth working
- ✅ Media/photos/books data migrated to nas-prod-01

**Historical note:** Completion of subsequent Phase 5 work should be established using `phases/phase-5-hardening.md` and the relevant current service documentation, not by retroactively changing this March 2026 migration checklist.
