# Phase 4 — Service Migration

**Status:** In Progress  
**Started:** 2026-03-15  
**Last Updated:** 2026-03-15

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

## Wave 1 — Infrastructure Stack

### Traefik

- **Host:** docker-prod-01 (192.168.30.11)
- **Compose:** `/opt/stacks/traefik/compose.yaml`
- **Appdata:** `/opt/appdata/traefik/`
- **Version:** v3.6.10

#### Configuration
- HTTP → HTTPS redirect on all entrypoints
- Wildcard TLS cert for `*.giohosted.com` via Cloudflare DNS-01 challenge (Let's Encrypt)
- Docker provider: `exposedbydefault=false` — containers opt-in via `traefik.enable=true` label
- File provider: `/opt/appdata/traefik/config/` — used for cross-host static routes
- Dashboard exposed at `traefik.giohosted.com` — restricted to internal VLANs via `local-only` middleware (192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24)
- Cert stored in `/opt/appdata/traefik/acme.json` — auto-renews 30 days before expiry

#### Cross-Host Routing
Traefik routes to services on other VMs via the file provider. Static route configs live in `/opt/appdata/traefik/config/`. Current routes:

| File | Routes to |
|------|-----------|
| `authentik.yml` | `http://192.168.30.13:9000` |

#### Issues Encountered
- Chrome Secure DNS (DoH) bypassed AdGuard rewrites — resolved by disabling Chrome's secure DNS setting
- Old Cloudflare wildcard A record (`*` → `69.216.122.32`) caused Chrome DoH to resolve to Cloudflare IPs — deleted all v2 A records from Cloudflare DNS
- Old Cloudflare Tunnel record for `auth` caused connection errors — deleted, will be recreated when cloudflared is deployed

---

### Authentik

- **Host:** auth-prod-01 (192.168.30.13)
- **VM ID:** 105
- **OS:** Ubuntu Server 24.04 LTS
- **Resources:** 2 cores, 4GB RAM, 32GB disk (local-zfs, no cache)
- **Compose:** `/opt/stacks/authentik/compose.yaml`
- **Appdata:** `/opt/appdata/authentik/`
- **Version:** 2025.12.3

#### Configuration
- Restored from v2 backup — all OIDC providers, applications, users, and policies intact
- Postgres data restored from `/mnt/user/backups/docker-v2/appdata/authentik/`
- `.env` values (PG_PASS, AUTHENTIK_SECRET_KEY) carried forward from v2 — required to match existing database
- Authentik server exposed on port 9000 — Traefik routes `auth.giohosted.com` → `192.168.30.13:9000` via file provider static route
- No Docker labels on Authentik containers — cross-host routing handled entirely by Traefik file provider

#### Network Design
Authentik runs on a dedicated VM (auth-prod-01) — not on docker-prod-01. Traefik on docker-prod-01 cannot discover Authentik containers via Docker socket (different host). Static route in Traefik file provider handles routing by IP instead.

#### Issues Encountered
- Authentik compose initially included Traefik Docker labels and proxy network — removed since Traefik cannot discover containers on a different host via Docker socket
- Old Cloudflare Tunnel record for `auth.giohosted.com` caused browser to receive Cloudflare error 1033 — deleted from Cloudflare DNS

---

## Wave 1 — Remaining (In Progress)

| Service | Status | Notes |
|---------|--------|-------|
| cloudflared | ⬜ Pending | CF Tunnel for external access — ABS, Seerr, Shelfmark, Authentik |
| adguardhome-sync | ⬜ Pending | Already deployed in Phase 3 — verify still running |
| Dockman | ⬜ Pending | Docker compose management UI |
| Homarr | ⬜ Pending | Operations dashboard |
| Beszel | ⬜ Pending | Host/VM metrics — agents on all hosts |

---

## Wave 2 — Media Stack (Pending)

| Service | Status |
|---------|--------|
| Plex (already on Unraid) | ⬜ Restore DB from backup, verify QuickSync |
| Sonarr TV | ⬜ Pending |
| Sonarr Anime | ⬜ Pending |
| Radarr 1080p | ⬜ Pending |
| Radarr 4K | ⬜ Pending |
| Prowlarr | ⬜ Pending |
| Bazarr | ⬜ Pending |
| Profilarr | ⬜ Pending |
| Maintainerr | ⬜ Pending |
| Seerr | ⬜ Pending |
| Tautulli | ⬜ Pending |

---

## Wave 3 — Torrent Stack (Pending)

| Service | Status |
|---------|--------|
| Gluetun | ⬜ Pending |
| qBittorrent | ⬜ Pending |
| qBitrr | ⬜ Pending |

---

## Wave 4 — Books Stack (Pending)

| Service | Status |
|---------|--------|
| Audiobookshelf | ⬜ Pending |
| Calibre-Web-Automated | ⬜ Pending |
| Shelfmark | ⬜ Pending |

---

## Wave 5 — Photos Stack (Pending)

| Service | Status | Notes |
|---------|--------|-------|
| Immich | ⬜ Pending | Requires immich-prod-01 VM creation |

---

## DNS Rewrites Added (AdGuard)

| Hostname | IP | Notes |
|----------|----|-------|
| traefik.giohosted.com | 192.168.30.11 | Traefik dashboard |
| auth.giohosted.com | 192.168.30.11 | Authentik — routed via Traefik file provider |

---

## Cloudflare DNS Changes

| Record | Action | Reason |
|--------|--------|--------|
| `*` A record → 69.216.122.32 | Deleted | v2 NPM wildcard — no longer needed |
| `giohosted.com` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `photos` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `auth` Tunnel record → homelab-v2 | Deleted | Dead tunnel — will recreate when cloudflared v3 is deployed |

---

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| Ubuntu recovery VM instead of TrueNAS VM | TrueNAS hung on boot repeatedly due to ZFS pool auto-import. Ubuntu with manual zpool import is lighter and more controllable. |
| Authentik on dedicated VM (auth-prod-01) | Per roadmap — LXC officially removed from Proxmox community scripts May 2025. VM is the only stable option. |
| Traefik file provider for cross-host routing | Traefik Docker provider can only discover containers on the same host. File provider handles static routes to other VMs by IP. |
| Authentik secrets carried forward from v2 | PG_PASS and AUTHENTIK_SECRET_KEY must match the existing database — changing them would break the restore. |
| Chrome Secure DNS disabled on workstation | Required for AdGuard DNS rewrites to work — DoH bypasses local DNS entirely. |

---

## Exit Criteria

- ⬜ All services running on v3 infrastructure
- ⬜ External access working via CF Tunnel
- ⬜ PBS backing up all VMs nightly
- ⬜ Backup scripts running with Healthchecks heartbeats
- ✅ Traefik deployed with wildcard cert
- ✅ Authentik deployed and restored from v2
- ✅ Media/photos/books data migrated to nas-prod-01