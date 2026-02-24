# Homelab v3.0 — Master Architecture & Phased Build Roadmap

  

**Version:** 7.0 | **Author:** Gio | **Status:** Planning  

*Living document — update as phases complete*

  

---

  

> **DESIGN PHILOSOPHY**  

> Clean-slate rebuild with institutional knowledge from v2. Every decision is deliberate, documented, and built toward future goals (Kubernetes, GitOps, HA). Where v2 evolved organically, v3 is designed intentionally.

  

| Principle | Applied In v2? | v3 Commitment |

|---|---|---|

| Bare-metal NAS separation | No (TrueNAS as VM) | Dedicated NAS host |

| Proxmox OS redundancy | No (single NVMe) | Mirrored NVMe boot |

| Proper VLAN segmentation with rules | Partial (no FW rules) | Full inter-VLAN firewall |

| UPS protection | No | Required before power-on |

| Kubernetes-ready architecture | No | Designed for future k3s |

  

---

  

## SECTION 1 — HARDWARE ARCHITECTURE

  

### 1.1 Physical Host Overview

  

| Host | Hardware | Role | Notes |

|---|---|---|---|

| Primary Compute | Minisforum MS-A2 (Ryzen 9 7945HX) | Proxmox node — VMs & LXCs | Dual 10GbE SFP+, dual 2.5GbE RJ45 |

| NAS | Repurposed tower (i5-13400, 32GB DDR4) | 4U 12-bay Unraid NAS + Plex Docker container | LSI HBA migrates here; 10GbE via DAC to MS-A2; QuickSync iGPU for Plex transcoding |

| Secondary Compute | Dell Optiplex 3070 Micro (i5-9th gen) | Proxmox node — PBS + secondary workloads | Upgrade to 32GB RAM |

| DNS / Monitoring | Raspberry Pi 4B | Uptime Kuma + Beszel server + Proxmox QDevice | Secondary AdGuard moved to LXC on pve-prod-02 (dns-prod-02) |

  

---

  

### 1.2 Primary Compute — Minisforum MS-A2

  

#### Procurement & RAM

- Model: Minisforum MS-A2 with Ryzen 9 7945HX

- RAM: 64GB DDR5 SO-DIMM (2x 32GB) — sufficient for current workloads; max is 96GB

- Boot storage: 2x NVMe in RAID-1 mirror for Proxmox OS (no more single-drive boot risk)

- 10GbE SFP+: DAC cable direct to NAS for storage traffic (keep off LAN switch)

- 2.5GbE RJ45: uplink to UniFi switch for VM/LXC traffic

  

> **NOTE:** The Radeon 680M iGPU (integrated in 7945HX) is available for future workloads such as ML inference or additional transcoding, but is not required at launch. Plex runs on Unraid using the i5-13400 QuickSync iGPU — no passthrough needed on MS-A2.

  

---

  

### 1.3 NAS Host — Repurposed Tower

  

#### Hardware Reuse

- CPU: i5-13400 (sell the i5-13600 to offset MS-A2 cost — Unraid is IO-bound, not compute-bound)

- RAM: 32GB DDR4 (sell 2x 16GB sticks — 32GB is generous for a NAS with no heavy VM workloads)

- Mobo: Reuse existing board from current tower

- LSI SAS 9120-8i HBA: migrates from current host to NAS

- Drives: See Section 3 (Storage) for full drive layout

- No NVMe cache pool at launch — see rationale below

- 10GbE SFP+: Add a single-port 10GbE SFP+ PCIe NIC for DAC link to MS-A2

- 1GbE or 2.5GbE: uplink to UniFi switch for management, Synology replication, WAN traffic

  

#### Why No Cache Pool at Launch

- Downloads bypass cache entirely — mandatory for hardlinks (downloads and media must share one filesystem)

- Container appdata lives on docker-prod-01 local disk, not NAS NFS

- Plex transcode temp can point at a local Unraid directory

- No real workload justifies cache at launch — don't buy hardware to solve a problem you don't have

- If a use case emerges later: add 2x 512GB NVMe drives as BTRFS RAID1 mirrored cache pool

  

> **⚠ HARD REQUIREMENT:** Downloads share must NEVER use cache. Always write directly to parity array. Violations break hardlinks silently and cause file duplication.

  

---

  

### 1.4 Secondary Compute — Dell Optiplex 3070 Micro

  

- Upgrade RAM to 32GB DDR4

- Runs Proxmox VE as a second node

- Initial workload: Proxmox Backup Server (PBS) as a VM (pbs-prod-01)

- Initial workload: Secondary AdGuard Home as an LXC (dns-prod-02) — DNS resilience across both Proxmox nodes

- Future workload: Second Talos/k3s worker node (k3s-work-lab-02)

- Joins Proxmox cluster with MS-A2 for centralized management

  

---

  

### 1.5 Rack & Power

  

- Open-frame rack: 15–18U (StarTech or NavePoint open-frame)

- UPS: APC Back-UPS 1500VA — purchase BEFORE powering any new hardware. Must have USB port for NUT integration.

- Layout (top to bottom): Patch panel → UniFi switch → MS-A2 shelf → Optiplex shelf → NAS 4U → Pi (velcro/hook) → UPS at bottom

  

#### UPS Graceful Shutdown via NUT

- NUT (Network UPS Tools) server runs on nas-prod-01 — Unraid has NUT built into Community Apps

- UPS connects to nas-prod-01 via USB

- NUT clients run on pve-prod-01 and pve-prod-02 — receive low battery signal and trigger graceful Proxmox shutdown

- Proxmox shutdown sequence: VMs and LXCs shut down first, then hypervisor OS

- Unraid shuts itself down last after both Proxmox nodes are confirmed off

- Result: fully automated graceful shutdown on power loss even when unattended

  

> **⚠ HARD REQUIREMENT:** UPS must be purchased and installed before the NAS is powered on with spinning rust drives. Power loss during a write on HDDs is a data corruption risk. This is non-negotiable.

  

---

  

## SECTION 2 — NETWORKING

  

### 2.1 Network Hardware

  

| Device | Model | Notes |

|---|---|---|

| Router/Firewall | UniFi Dream Machine SE (UDM-SE) | Existing — carry forward |

| Switch | UniFi USW-Pro-Max-16 or USW-Flex-2.5G (new) | Managed 2.5GbE switch, VLAN trunking |

| Storage Link | 10GbE SFP+ DAC (MS-A2 ↔ NAS direct) | Dedicated storage traffic, not on LAN switch |

  

---

  

### 2.2 VLAN Design

  

> **NOTE:** VLAN 1 (default untagged) will NOT be used as the management VLAN. A dedicated VLAN ID is assigned to management to avoid devices landing on it accidentally.

  

| VLAN ID | Name | Subnet | Members & Purpose |

|---|---|---|---|

| 10 | Management | 192.168.10.0/24 | Proxmox hosts, NAS management, UniFi switch, UDM-SE. Strict access — management devices only. |

| 20 | Trusted | 192.168.20.0/24 | Personal laptops, phones, trusted devices. Current v2 services live here during migration — left untouched. |

| 30 | Services | 192.168.30.0/24 | All v3 VMs and LXCs. docker-prod-01, auth-prod-01, immich-prod-01, PBS, AdGuard LXCs. All new service IPs live here. |

| 40 | IoT | 192.168.40.0/24 | Smart home devices, printers, anything untrusted. Internet access only. No inter-VLAN. Migrate existing IoT devices from 192.168.30.0 during Phase 1. |

  

> **MIGRATION NOTE:** v2 services currently live on 192.168.20.0 (Trusted VLAN). They stay there untouched while v3 is built on 192.168.30.0 (Services VLAN). During Phase 4, AdGuard DNS rewrites are flipped per-service from v2 IPs to v3 IPs. No big-bang cutover — gradual per-service migration.

  

---

  

### 2.3 IP Address Plan

  

#### VLAN 10 — Management (192.168.10.0/24)

  

| Device | IP | Notes |

|---|---|---|

| UDM-SE | 192.168.10.1 | Gateway — already configured, do not change |

| Core Switch (UniFi) | 192.168.10.2 | Management interface |

| UPS (if networked later) | 192.168.10.3 | Reserved — NUT handles shutdown via USB for now |

| nas-prod-01 (Unraid) | 192.168.10.10 | Management/NFS interface |

| pve-prod-01 (MS-A2) | 192.168.10.11 | Proxmox management UI |

| pve-prod-02 (Optiplex) | 192.168.10.12 | Proxmox management UI |

| pi-prod-01 (Raspberry Pi) | 192.168.10.20 | QDevice + monitoring |

| 192.168.10.30–.49 | — | Reserved future expansion / IPMI / OOB |

| 192.168.10.100+ | — | DHCP pool if needed |

  

#### VLAN 20 — Trusted (192.168.20.0/24)

  

Unchanged from v2. Personal laptops, phones, trusted devices. v2 services (Proxmox at .2, docker host at .10) remain here during the v3 build and migration. Decommissioned v2 IPs freed up once Phase 4 cutover is complete.

  

#### VLAN 30 — Services (192.168.30.0/24)

  

All v3 VMs and LXCs. Containers inside VMs are accessed via Traefik on the VM's IP — they do not get individual VLAN IPs.

  

| Device | IP | Notes |

|---|---|---|

| dns-prod-01 (AdGuard LXC, pve-prod-01) | 192.168.30.10 | Primary DNS — AdGuard Home |

| docker-prod-01 (media/apps VM, pve-prod-01) | 192.168.30.11 | All media stack containers behind Traefik |

| pbs-prod-01 (PBS VM, pve-prod-02) | 192.168.30.12 | Proxmox Backup Server |

| auth-prod-01 (Authentik VM, pve-prod-01) | 192.168.30.13 | Authentik IdP — dedicated VM |

| immich-prod-01 (Immich VM, pve-prod-01) | 192.168.30.14 | Immich — isolated for resource tuning |

| dns-prod-02 (AdGuard LXC, pve-prod-02) | 192.168.30.15 | Secondary DNS — synced from dns-prod-01 |

| 192.168.30.100+ | — | DHCP / sandbox / test / future k3s nodes |

  

#### VLAN 40 — IoT (192.168.40.0/24)

  

7 existing IoT devices migrate from 192.168.30.0 during Phase 1. Internet access only. No inter-VLAN routing. Gateway at 192.168.40.1. DHCP pool 192.168.40.100+.

  

---

  

### 2.4 Inter-VLAN Firewall Rules

  

Rules enforced at UDM-SE. Default policy is DENY ALL inter-VLAN. Explicit ALLOW rules only.

  

| Source | Destination | Port / Protocol | Action | Reason |

|---|---|---|---|---|

| Trusted (20) | Services (30) | Any | ALLOW | Users reach internal services |

| Trusted (20) | Management (10) | TCP 8006, 22, 443 | ALLOW | Admin access to Proxmox, SSH, NAS UI |

| Services (30) | Services (30) | Any | ALLOW | Inter-service communication |

| Services (30) | Trusted (20) | Any | DENY | Services cannot initiate to user devices |

| Services (30) | Management (10) | Any | DENY | Services cannot touch infra management |

| IoT (40) | Any internal | Any | DENY | IoT fully isolated from all internal VLANs |

| Any | Internet | Any | ALLOW | All VLANs can reach WAN unless blocked |

  

---

  

### 2.5 DNS & Reverse Proxy

  

#### Split-Horizon DNS

- AdGuard Home handles DNS rewrites: `*.giohosted.com` → internal Traefik IP

- External DNS via Cloudflare — CNAMEs point to Cloudflare Tunnel

- Same FQDN works internally and externally with valid TLS

- adguardhome-sync keeps dns-prod-01 (LXC on pve-prod-01) and dns-prod-02 (LXC on pve-prod-02) in sync

  

#### Reverse Proxy — Migration from NPM to Traefik

- Traefik replaces Nginx Proxy Manager in v3.0

- Wildcard cert via Cloudflare DNS-01 challenge (`*.giohosted.com`) — one cert covers all internal services

- Docker label-based routing — no separate config files per service

- Traefik familiarization now pays dividends when k3s ingress is configured later (Traefik is k3s default ingress)

  

> **NOTE:** NPM will be kept running in parallel during migration until all services are confirmed working behind Traefik. Cutover is a flip of the AdGuard DNS rewrite target IP, making rollback instant.

  

---

  

### 2.6 Remote Access

  

- **Plex:** port forward 32400 on UDM-SE → Unraid host IP. Plex handles its own TLS and relay. Simple, proven, no overengineering.

- **All other externally exposed services:** Cloudflare Tunnel via cloudflared container on docker-prod-01. No port forwarding required.

- **WireGuard VPN:** remains on UDM-SE for secure remote LAN access — carry forward as-is

- **Externally exposed via CF Tunnel:** Audiobookshelf, Shelfmark, Seerr, Authentik

- **Pangolin:** evaluated and rejected. VPS relay adds latency for Plex streaming; CF Tunnel is free and works; no meaningful reason to add VPS cost. Revisit only if CF Tunnel becomes a real constraint.

  

> **NOTE:** Plex direct port forward intentionally kept separate from CF Tunnel. Cloudflare ToS prohibits video streaming through their tunnel. Opening one port for a well-maintained application is an acceptable risk.

  

---

  

## SECTION 3 — STORAGE ARCHITECTURE

  

### 3.1 NAS Platform Decision

  

- **Platform:** Unraid 7.2.3 (replaces TrueNAS SCALE)

- **Rationale:** Hybrid ZFS + parity array model fits the data risk tolerance perfectly. Mixed drive sizes supported. Simpler UX for the use case.

- ZFS used where data integrity is non-negotiable. Parity array used for recoverable bulk media.

  

---

  

### 3.2 Drive Inventory & Classification

  

| Drive | Count | Type | Classification | Assignment |

|---|---|---|---|---|

| WD Red Pro 12TB | 5 | CMR NAS | ✅ Production | 2x parity + 3x data (parity array) |

| WD Red Plus 4TB | 5 | CMR NAS | ✅ Production | 2x ZFS mirror pool + 2x array expansion + 1x spare |

| Seagate IronWolf 6TB | 2 | CMR NAS | ✅ Production | Hot spares / future expansion |

| Seagate SkyHawk 6TB | 4 | Surveillance CMR | ⚠️ Non-NAS | Repurpose in Synology for cold backup storage only |

| Seagate Barracuda 4TB | 1 | Desktop | ❌ Retire | Not suitable for always-on NAS duty |

  

---

  

### 3.3 Unraid Pool Layout

  

#### Parity Array — Bulk Media & Downloads

- 2x WD Red Pro 12TB as dual parity drives

- 3x WD Red Pro 12TB as data drives (~36TB usable)

- Filesystem: XFS per-disk (Unraid default — do not use BTRFS for array)

- Shares: media (TV, movies, anime, books), downloads (all categories)

- Downloads share writes DIRECTLY to array — no cache involvement. Mandatory to keep downloads and media on same filesystem so hardlinks and atomic moves work.

- Expandable: 2x WD Red Plus 4TB and 2x IronWolf 6TB available as additional data drives when needed

  

#### ZFS Mirror Pool — Precious Data

- 2x WD Red Plus 4TB in ZFS mirror (~4TB usable)

- Shares: backups, photos (Immich library)

- ZFS snapshots enabled — same data integrity guarantees as TrueNAS had for these datasets

- 1x WD Red Plus 4TB kept as cold spare for this pool

  

#### Cache Pool — Not Installed at Launch

- No cache pool at initial build. Workload does not justify it.

- If added later: 2x 512GB NVMe, BTRFS RAID1 mirrored

- Would handle only small random-write workloads — never downloads, never container appdata

  

> **⚠ HARDLINK RULE:** Downloads and media shares must always be on the same Unraid pool/filesystem. Never route downloads through cache while media lives on the array — this breaks hardlinks and causes silent duplication. Downloads share = parity array direct, no exceptions.

  

---

  

### 3.4 Share & Directory Layout

  

#### Unraid NAS Shares (/mnt/user/)

  

| Share | Pool | NAS Path | Contents |

|---|---|---|---|

| media | Parity Array | /mnt/user/media | All media — movies (1080p+4K), TV, anime, books |

| downloads | Parity Array | /mnt/user/downloads | Active download staging for all categories |

| photos | ZFS Mirror | /mnt/user/photos | Immich library (irreplaceable — ZFS protected) |

| backups | ZFS Mirror | /mnt/user/backups | PBS backups, Docker appdata, Plex DB, Proxmox dumps |

| appdata | Parity Array | /mnt/user/appdata | Docker/app config from NAS perspective — container appdata actually lives on docker-prod-01 local disk |

| isos | Parity Array | /mnt/user/isos | Proxmox ISO images |

  

> **CRITICAL:** `/mnt/user` is Unraid's internal path and cannot be renamed. This path is never seen by containers — the docker-host VM mounts the NFS export as `/data`. All compose files reference `/data` paths only.

  

#### Docker Host — /data Mount Structure

  

The NAS exports `/mnt/user` via NFS. The docker-host VM mounts this as `/data`. Everything below is logical folder organization within that single mount — one filesystem, hardlinks work everywhere.

  

| Path | Purpose |

|---|---|

| /data/media/movies/1080p/ | Radarr (1080p) library |

| /data/media/movies/4k/ | Radarr (4K) library |

| /data/media/tv/ | Sonarr (TV) library |

| /data/media/anime/ | Sonarr (Anime) library |

| /data/media/books/ebooks/ | CWA Calibre library (canonical ebook state) |

| /data/media/books/audiobooks/ | ABS library — Shelfmark hardlinks here |

| /data/downloads/movies/ | qBit movie download staging |

| /data/downloads/tv/ | qBit TV download staging |

| /data/downloads/anime/ | qBit anime download staging |

| /data/downloads/books/ebooks/downloads/ | qBit ebook downloads — seeds from here |

| /data/downloads/books/ebooks/ingest/ | Shelfmark hardlinks here → CWA ingests + deletes hardlink → original stays for seeding |

| /data/downloads/books/audiobooks/downloads/ | qBit audiobook downloads — seeds from here → Shelfmark hardlinks to /data/media/books/audiobooks/ |

| /data/photos/ | Immich photo library (ZFS mirror pool) |

  

#### Ebook Hardlink Workflow (Resolved in v3)

  

v2 had a seeding breakage bug: CWA deleted the ingest file after import, killing the qBit torrent. v3 fixes this with hardlinks.

  

| Step | Actor | Action |

|---|---|---|

| 1 | qBittorrent | Downloads ebook to /data/downloads/books/ebooks/downloads/ — seeds from here permanently |

| 2 | Shelfmark | Hardlinks completed file to /data/downloads/books/ebooks/ingest/ |

| 3 | CWA | Detects file in ingest/, imports to /data/media/books/ebooks/, deletes the hardlink in ingest/ |

| 4 | qBittorrent | Original file in downloads/ is untouched — seeding continues uninterrupted |

| 5 | qBitrr | After 14 days, qBitrr stops seeding per MAM policy |

  

---

  

### 3.5 UID/GID Strategy

  

- Service UID/GID: `2000:2000` (clean break from v2 — fresh NFS exports, fresh ownership)

- Human user (gio): `1000:1000`

- All containers interacting with NFS/SMB mounts must run as `PUID=2000 PGID=2000`

- NAS share permissions set to allow `2000:2000` read/write on all service shares

- Clean separation: no services run as root, no more gio ownership of datasets

  

---

  

### 3.6 NFS vs SMB

  

- Primary protocol: NFS for Linux VM/container mounts (carry forward from v2)

- SMB available for any Windows or macOS access if needed

- Unraid exports both protocols natively — no extra config required

  

---

  

## SECTION 4 — COMPUTE & VIRTUALIZATION

  

### 4.1 Proxmox Cluster

  

- Two-node Proxmox cluster: pve-prod-01 (MS-A2) + pve-prod-02 (Optiplex)

- Cluster purpose: unified management only — one web UI to manage both nodes, all VMs and LXCs visible from one place

- **HA (High Availability) is NOT enabled** — pve-prod-02 cannot handle pve-prod-01's workload. HA without matched hardware is false security.

- QDevice on pi-prod-01 acts as a lightweight tiebreaker to prevent split-brain — 10 minute setup, well worth it

- PBS on pve-prod-02 backs up all VMs on both nodes

  

> **CLUSTER vs HA CLARIFICATION:** Clustering = one unified management UI for both nodes. HA = automatic VM migration on node failure. These are independent features. We enable clustering for convenience. We do not enable HA because it requires matched hardware and a 3+ node quorum to be meaningful. If pve-prod-01 goes down, services on it are down until it comes back up — that is acceptable for a homelab.

  

---

  

### 4.2 VM & LXC Layout — pve-prod-01 (MS-A2, Primary)

  

| Guest | Type | RAM | Notes |

|---|---|---|---|

| docker-prod-01 | VM (Ubuntu 24.04) | 16–20GB | All media/app containers. ARR stack, books, torrent, infra. Traefik at 192.168.30.11. |

| auth-prod-01 | VM (Debian) | 2GB | Authentik IdP. Dedicated VM — LXC rejected due to documented stability issues. 192.168.30.13. |

| immich-prod-01 | VM (Ubuntu 24.04) | 4–6GB | Immich photo server + ML worker. Isolated for independent resource tuning. 192.168.30.14. |

| dns-prod-01 | LXC (Debian) | 512MB | Primary AdGuard Home. 192.168.30.10. |

| [future] k3s-ctrl-lab-01 | VM | 4GB | k3s control plane — Phase 6 lab only |

| [future] k3s-work-lab-01 | VM | 8GB | k3s worker — Phase 6 lab only |

  

---

  

### 4.3 VM & LXC Layout — pve-prod-02 (Optiplex, Secondary)

  

| Guest | Type | RAM | Notes |

|---|---|---|---|

| pbs-prod-01 | VM (Debian) | 4GB | Proxmox Backup Server. Backs up all VMs on both nodes. 192.168.30.12. |

| dns-prod-02 | LXC (Debian) | 512MB | Secondary AdGuard Home. Synced from dns-prod-01 via adguardhome-sync. 192.168.30.15. |

| [future] k3s-work-lab-02 | VM | 8GB | k3s worker — Phase 6 lab only |

  

---

  

### 4.4 Proxmox Boot Redundancy

  

- **MS-A2:** 2x NVMe configured as ZFS mirror in Proxmox installer — single drive failure does not kill the hypervisor

- **Optiplex:** Single NVMe is acceptable given its secondary/non-critical role

  

---

  

### 4.5 iGPU & Hardware Transcoding

  

- Plex runs on Unraid directly — hardware transcoding via i5-13400 Intel UHD 730 iGPU using QuickSync

- QuickSync on 12th/13th gen Intel is mature, well-supported by Plex, and handles 2x simultaneous 1080p transcodes without breaking a sweat

- No iGPU passthrough required on MS-A2 — Plex is not running there

- MS-A2 Radeon 680M iGPU available for future use (ML workloads, additional transcoding) but not needed at launch

- Dedicated GPU: not required. Revisit only if transcode demand exceeds QuickSync capacity — unlikely at current user count

  

---

  

## SECTION 5 — SERVICES ARCHITECTURE

  

### 5.1 Docker Host Layout

  

#### Stack Organization

- Compose root: `/opt/stacks/<stack-name>/compose.yaml`

- Appdata root: `/opt/appdata/<app-name>/`

- All appdata lives on local VM disk (not NFS) — databases and config stay local for performance and reliability

  

#### Container Management UI

- **Dockman** — carry forward from v2

- Key advantage: restart or update individual containers without restarting the entire stack

- When k3s is introduced: use Lens or Headlamp as the dedicated Kubernetes UI

  

---

  

### 5.2 Full Service Inventory

  

| Service | Stack | v3 Status | Notes |

|---|---|---|---|

| Traefik | infra | New (replaces NPM) | Reverse proxy + wildcard TLS |

| Authentik | infra | Carry forward | IdP — OIDC for all SSO apps. Runs on auth-prod-01 VM. |

| cloudflared | infra | Carry forward | CF Tunnel — ABS, Shelfmark, Seerr, Authentik |

| AdGuard Home (LXC) | lxc | Carry forward | Primary DNS + ad-blocking |

| adguardhome-sync | infra | Carry forward | Syncs dns-prod-01 (pve-prod-01 LXC) to dns-prod-02 (pve-prod-02 LXC) |

| Homarr | infra | Carry forward | Operations dashboard |

| Beszel | monitoring | Carry forward | Host/VM metrics. Agents on all hosts. |

| Uptime Kuma (Pi) | pi-docker | Carry forward | Service uptime monitoring |

| Healthchecks.io | cloud | Carry forward | Backup job heartbeat monitoring |

| Cockpit | docker-prod-01 | New | Web-based day-to-day management of docker-prod-01 (disk, network, logs, updates) |

| Plex | unraid-docker | Move to Unraid native Docker | Runs directly on Unraid — QuickSync via i5-13400 iGPU. No NFS hop. Port 32400 forwarded on router for remote access. |

| Sonarr (TV) | arr | Carry forward | TV show automation — regular TV only |

| Sonarr (Anime) | arr | New instance | Anime automation — separate instance, different quality profiles |

| Radarr (1080p) | arr | Carry forward | Movie automation — 1080p WebDL for shared users |

| Radarr (4K) | arr | New instance | 4K WebDL automation — local viewing via Infuse/Apple TV 4K |

| Prowlarr | arr | Carry forward | Indexer management |

| Bazarr | arr | Carry forward | Subtitle automation |

| Profilarr | arr | Carry forward | Quality profile management for Sonarr/Radarr |

| Maintainerr | arr | Carry forward | Media lifecycle visibility (no deletion) |

| Seerr | arr | Carry forward | Media request frontend (formerly Overseerr) |

| Tautulli | arr | Carry forward | Plex analytics |

| qBittorrent | torrent | Carry forward | Must stay containerized with Gluetun killswitch |

| Gluetun | torrent | Carry forward | ProtonVPN WireGuard killswitch for qBit |

| qBitrr | torrent | New (replaces qbit-automation sidecar) | Single installation manages all 4 ARR instances. Seeding control, MAM compliance, torrent health monitoring. Web UI for management. |

| Immich | photos | Move to immich-prod-01 VM | Dedicated VM on pve-prod-01 — isolated for ML worker resource tuning. Storage on ZFS mirror pool. |

| Audiobookshelf | books | Carry forward | Audiobook server with OIDC |

| Calibre-Web-Automated | books | Carry forward | Ebook library manager |

| Shelfmark | books | Carry forward | Ebook/audiobook request frontend |

| Dockman | infra | Carry forward | Docker compose management — individual container control without stack restarts |

| Flaresolverr | arr | Carry forward | Cloudflare bypass for Prowlarr indexers |

  

---

  

### 5.3 Backup Architecture

  

#### Backup Tiers (v3)

  

| Tier | What | Tool | Destination |

|---|---|---|---|

| Tier 0 | VM/LXC snapshots | Proxmox Backup Server (PBS) | pbs-prod-01 VM → ZFS mirror share on NAS |

| Tier 1 | Docker appdata + stacks | Hardened rsync script + Healthchecks | NAS /backups share (ZFS mirror pool) |

| Tier 1 | Plex database | Dedicated backup script (stop/backup/start) | NAS /backups/plex/db |

| Tier 2 | NAS share snapshots | Unraid ZFS snapshot (backups + photos pool) | Local ZFS snapshots on NAS |

| Tier 3 | Off-box cold copy | Synology ABB (pull-based, read-only creds) | Synology NAS (SkyHawks) — nightly |

| Tier 4 | Cloud backup | Backblaze B2 (future) | Critical data offsite — Immich photos, backups |

  

> **NOTE:** PBS does not back up application data inside VMs — only the VM disk image. Application-level backups (Docker appdata, Plex DB) remain essential and run independently of PBS.

  

---

  

## SECTION 6 — IDENTITY & ACCESS

  

### 6.1 Identity Model (Carry Forward)

  

- Authentik remains the single IdP — no change to the model

- Group structure: admins, users, services

- Entitlement-based Cloudflare Access policies — carry forward exactly

- MFA enforced for admins

  

---

  

### 6.2 SSO-Enabled Services (v3 Target)

  

| Service | SSO Method | Notes |

|---|---|---|

| Proxmox | OIDC | Both nodes |

| Audiobookshelf | OIDC | Sub unlink fix documented |

| Calibre-Web-Automated | OAuth2 | Manual link per user |

| Beszel | OIDC | Requires email_verified: true custom scope |

| Synology DSM | OIDC | Local admin retained as break-glass |

| Homarr | OIDC | New in v3 |

| Immich | OIDC | New in v3 |

| ARR stack, Uptime Kuma | None (intentional) | LAN-only, low risk — no SSO overhead |

  

---

  

### 6.3 Externally Exposed Services

  

| Hostname | Service | Protection |

|---|---|---|

| audiobooks.giohosted.com | Audiobookshelf | Cloudflare Access + Authentik OIDC |

| books.giohosted.com | Shelfmark | Cloudflare Access + Authentik OIDC |

| request.giohosted.com | Seerr | Cloudflare Access |

| auth.giohosted.com | Authentik | Tunnel only (must NOT be behind CF Access) |

  

---

  

## SECTION 7 — PHASED BUILD ROADMAP

  

Phases are sequential. Each phase has clear entry criteria (what must be true before starting) and exit criteria (what must be true to consider the phase done). Do not skip phases.

  

---

  

### PHASE 0 — Procurement & Physical Build

*Pre-requisite*

  

**Objective:** Acquire all hardware, build the rack, validate physical layer before any software is installed.

  

**Tasks:**

- Purchase UPS (APC 1500VA) — FIRST purchase, before anything else

- Purchase Minisforum MS-A2 (Ryzen 9 7945HX, 64GB DDR5)

- Purchase 2x NVMe for MS-A2 Proxmox boot mirror (1TB each)

- Purchase 10GbE SFP+ PCIe NIC for NAS host

- Purchase 2x DAC cables (MS-A2 ↔ NAS 10GbE direct link)

- Purchase UniFi switch (USW-Pro-Max-16 or USW-Flex-2.5G)

- Purchase 4U 12-bay chassis (IStarUSA D-400-12 or Rosewill RSV-L4412U)

- Purchase open-frame rack (15–18U)

- Sell i5-13600 — use i5-13400 for NAS host

- Sell 2x 16GB DDR4 sticks — keep 32GB for NAS host

- Upgrade Optiplex 3070 Micro RAM to 32GB

- Assemble NAS in new chassis — install i5-13400, 32GB DDR4, LSI HBA, drives, 10GbE NIC

- Mount all equipment in rack

- Connect UPS and verify power to all devices

- Cable management — label all cables

  

**Exit Criteria:**

- All hardware powered on and POST-ing

- UPS protecting full stack

- Network cables labeled and connected to switch

  

---

  

### PHASE 1 — Network Foundation

*Core Infrastructure*

  

**Objective:** Get the new switch in place, VLANs properly configured with firewall rules, and verify connectivity before any services move.

  

**Entry Criteria:** Phase 0 complete. All hardware racked and powered.

  

**Tasks:**

- Adopt new UniFi switch into UDM-SE

- Create VLANs: 10 (Management, 192.168.10.0/24), 20 (Trusted, 192.168.20.0/24), 30 (Services, 192.168.30.0/24), 40 (IoT, 192.168.40.0/24)

- Verify Management VLAN is NOT VLAN 1 — test and confirm in UniFi

- Assign static IPs per Section 2.3 IP Address Plan — UDM-SE at 192.168.10.1, switch at 192.168.10.2

- Migrate 7 existing IoT devices from 192.168.30.0 → VLAN 40 (192.168.40.0/24)

- Configure VLAN trunk ports to Proxmox hosts

- Configure inter-VLAN firewall rules per Section 2.4

- Verify IoT VLAN has no inter-VLAN reach

- Verify Trusted VLAN can reach Services VLAN

- Verify Management VLAN only reachable from Trusted on specific ports (8006, 22, 443)

- Test: laptop on Trusted VLAN can reach Proxmox UI — check

- Test: IoT device cannot ping anything on Trusted or Services — check

  

**Exit Criteria:**

- All VLANs created and tagged correctly

- All firewall rules in place and validated

- No unintended inter-VLAN routing

  

---

  

### PHASE 2 — NAS Build & Storage Commissioning

*Core Infrastructure*

  

**Objective:** Build Unraid from scratch, create all pools, configure shares, validate NFS exports and permissions before any services depend on it.

  

**Entry Criteria:** Phase 1 complete. NAS physically assembled and on Services VLAN.

  

**Tasks:**

- Install Unraid 7.2.3 (USB flash boot)

- Configure array: 2x 12TB WD Red Pro parity, 3x 12TB WD Red Pro data drives

- Configure ZFS mirror pool: 2x 4TB WD Red Plus

- No cache pool — skip entirely at launch

- Create shares: media, downloads, photos, backups, appdata, isos

- **CRITICAL:** Set downloads share — Use cache: No. Writes go direct to parity array. This is mandatory for hardlinks.

- **CRITICAL:** Set media share — Use cache: No. Same filesystem as downloads.

- Assign photos and backups shares to ZFS mirror pool

- Configure NFS exports for all shares

- Set share permissions: UID/GID `2000:2000` read/write on all service shares

- Enable ZFS snapshots on ZFS mirror pool (backups + photos)

- Verify 10GbE DAC link to MS-A2 is up and passing traffic

- Test NFS mount from a Linux VM — verify read/write and correct `2000:2000` ownership

- Enable Plex Docker container on Unraid — configure QuickSync iGPU passthrough to container

- Install and configure NUT server on Unraid via Community Apps

- Connect UPS USB to nas-prod-01 — verify NUT detects UPS and shows battery/status

- Configure NUT shutdown thresholds (e.g. trigger shutdown at 20% battery or after 5 min on battery)

- Configure Unraid notifications → Discord

- Begin media migration from old TrueNAS pool (copy → verify → do not delete until confirmed)

  

**Exit Criteria:**

- All pools created and healthy

- NFS mounts tested and working from a test VM

- Media data migrated and verified

- Snapshots running on ZFS pool

  

---

  

### PHASE 3 — Proxmox Rebuild & Core VM Infrastructure

*Compute Layer*

  

**Objective:** Fresh Proxmox install on MS-A2 and Optiplex. Core infrastructure VMs and LXCs standing up. Services are NOT migrated yet.

  

**Entry Criteria:** Phase 2 complete. NAS healthy, NFS exports verified.

  

**Tasks — MS-A2 (pve-prod-01):**

- Install Proxmox VE on 2x NVMe RAID-1 mirror (select ZFS RAID-1 during installer)

- Set hostname: `pve-prod-01`

- Configure Proxmox management interface on Management VLAN 10

- Add NAS NFS shares as Proxmox storage (media, backups, isos)

- Create AdGuard LXC on Services VLAN 30 — hostname `dns-prod-01`

- Configure AdGuard: DNS rewrites, upstream resolvers, carry forward v2 config

- Point DHCP (UDM-SE) to dns-prod-01 IP (192.168.30.10) as primary DNS

- Create docker-host Ubuntu 24.04 VM on Services VLAN 30 — hostname `docker-prod-01`

  - Allocate 16–20GB RAM, 4–6 vCPU

  - Mount NFS at `/data` → `/mnt/user` on nas-prod-01

  - Install Docker + Docker Compose

  - Create `/opt/stacks` and `/opt/appdata` directory structure

  - Install Cockpit + cockpit-files plugin for web-based day-to-day management

  

**Tasks — Optiplex (pve-prod-02):**

- Install Proxmox VE

- Set hostname: `pve-prod-02`

- Configure management interface on Management VLAN 10

- Create PBS VM — hostname `pbs-prod-01`

- Add PBS NFS storage target → NAS /backups share

- Configure PBS to back up VMs on pve-prod-01

  

**Tasks — NUT Clients (Graceful Shutdown on Power Loss):**

- Install NUT client on pve-prod-01: `apt install nut-client`

- Install NUT client on pve-prod-02: `apt install nut-client`

- Configure `/etc/nut/upsmon.conf` on each node to point at nas-prod-01 NUT server

- Configure shutdown action: on UPS critical signal, gracefully shut down all VMs then Proxmox host

- Test graceful shutdown: simulate power event, verify VMs stop cleanly before hypervisor

  

**Tasks — Proxmox Cluster + QDevice:**

- Create Proxmox cluster on pve-prod-01 first (Datacenter → Cluster → Create)

- Install QDevice support on pi-prod-01: `apt install corosync-qdevice`

- Add QDevice from pve-prod-01: `pvecm qdevice setup <pi-prod-01-ip>`

- Join pve-prod-02 to cluster (Datacenter → Cluster → Join)

- Verify both nodes visible under single Proxmox UI

- Confirm HA is NOT enabled — clustering for management convenience only

  

**Exit Criteria:**

- Proxmox healthy on both nodes with mirrored boot on MS-A2

- AdGuard LXC running — DNS working for all devices

- docker-host VM running — NFS mounts verified

- PBS running and taking test backups

  

---

  

### PHASE 4 — Service Migration

*Application Layer*

  

**Objective:** Migrate all services from v2 docker-host to v3 docker-host. Run v2 and v3 in parallel until v3 is validated, then cut over DNS and decommission v2.

  

**Entry Criteria:** Phase 3 complete. docker-prod-01 VM running with NFS mounts verified.

  

**Migration Strategy:** Migrate in waves. Infra first, then media stack, then books stack. Each wave is validated before the next begins.

  

#### Wave 1 — Infrastructure Stack

- Deploy Traefik with Cloudflare DNS-01 wildcard cert (`*.giohosted.com`)

- Verify Traefik is serving HTTPS internally before migrating any other service

- Deploy Authentik on auth-prod-01 VM (restore from backup)

- Deploy cloudflared (CF Tunnel) — keep pointed at v2 until all services migrated

- Deploy adguardhome-sync

- Create dns-prod-02 LXC on pve-prod-02 — verify adguardhome-sync propagating correctly

- Deploy Dockman

- Deploy Homarr (restore config from backup)

- Deploy Beszel + agents on all hosts

- Update AdGuard DNS rewrite: `*.giohosted.com` → Traefik IP (not NPM)

- Validate: internal HTTPS access to all infra services via Traefik

  

#### Wave 2 — Media Stack (ARR)

- Plex is already running on Unraid (configured in Phase 2) — restore database from backup, verify library, update paths to match /data structure

- Verify QuickSync hardware transcoding is working in Plex → Settings → Transcoder

- Configure Plex port forward on UDM-SE: 32400 → Unraid host IP

- Verify remote access working for external users

- Deploy Sonarr (TV) — restore config, verify `/data/media/tv` and `/data/downloads/tv` paths

- Deploy Sonarr (Anime) — fresh config with anime-specific quality profiles (TRaSH Guides), `/data/media/anime` and `/data/downloads/anime` paths

- Deploy Radarr (1080p) — restore config, verify `/data/media/movies/1080p` and `/data/downloads/movies` paths

- Deploy Radarr (4K) — fresh config with 4K WebDL quality profile, `/data/media/movies/4k` path

- **Migrate existing 4K movies into Radarr (4K)** — use Radarr's Movie Editor to bulk-select existing 4K titles, assign correct root folder (`/data/media/movies/4k`), and trigger library rescan. Do not re-download — existing files just need re-pathing and monitoring assigned to the correct instance.

- Deploy Prowlarr — restore config, sync indexers to all four ARR instances

- Deploy Bazarr, Profilarr, Maintainerr, Seerr, Tautulli — restore configs

- Verify hardlinks working: `stat` a file in /data/media — confirm inode matches file in /data/downloads

- Test end-to-end: request → grab → import → Plex for each category

  

#### Wave 3 — Torrent Stack

- Deploy torrent-stack: Gluetun + qBittorrent + qBitrr

- Verify VPN killswitch — confirm public IP is ProtonVPN Chicago

- Verify automatic port forwarding — MAM shows Fully Connectable

- Configure qBitrr — connect all four ARR instances (Sonarr-TV, Sonarr-Anime, Radarr-1080p, Radarr-4K)

- Configure qBitrr MAM seeding rules — 14 day minimum seed time on ebooks and audiobooks categories

- Verify qBitrr web UI accessible and showing all ARR instances correctly

- Repoint Sonarr and Radarr instances to new qBit endpoint

  

#### Wave 4 — Books Stack

- Deploy Audiobookshelf, Calibre-Web-Automated, Shelfmark

- Restore configs from v2 backups

- Verify NFS paths: /books/ebooks/library, /books/audiobooks/library, etc.

- Verify OIDC for Audiobookshelf

- Test audiobook hardlink workflow end-to-end

  

#### Wave 5 — Photos Stack (immich-prod-01)

- Deploy Immich stack on immich-prod-01 VM: server, ml worker, redis, postgresql

- Mount NFS `/data/photos` → immich-prod-01 from nas-prod-01

- Restore Immich library — photo data already migrated in Phase 2

- Verify face detection and CLIP indexing working (ML worker)

- Configure Authentik OIDC for Immich

- Verify Immich accessible via Cloudflare Tunnel

  

#### Cutover

- Update Cloudflare Tunnel to point at v3 services (docker-prod-01 for media stack, auth-prod-01 for Authentik, immich-prod-01 for Immich)

- Update Plex port forward to point at nas-prod-01

- Verify all externally exposed services working

- Monitor for 48 hours

- Decommission v2 Proxmox host — power down, keep for 2 weeks before wiping

  

**Exit Criteria:**

- All services running on v3 infrastructure

- External access working via CF Tunnel

- PBS backing up all VMs nightly

- Backup scripts running with Healthchecks heartbeats

  

---

  

### PHASE 5 — Operational Hardening

*Stabilization*

  

**Objective:** Tighten operations, validate all backup tiers, implement monitoring alerting, and document the final v3 state.

  

**Tasks:**

- Validate all PBS backup jobs — test restore of docker-prod-01 VM

- Validate rsync backup script with mountpoint safety check and lockfile

- Validate Plex DB backup script with EXIT trap

- Verify all Healthchecks.io heartbeats are firing

- Verify Synology ABB pulling from NAS correctly with read-only credentials

- Evaluate Backblaze B2 for Immich photos + backups (Tier 4 cloud backup)

- Review and tighten Traefik middleware: rate limiting, security headers

- Audit Authentik: verify all OIDC integrations, session policies, MFA enforcement

- Document v3 final state — write master runbook Phase 0 equivalent

- Verify VLAN firewall rules with penetration test from IoT VLAN

  

**Exit Criteria:**

- All backup tiers validated with test restores

- All monitoring and alerting confirmed end-to-end

- v3 master runbook written and committed

  

---

  

### PHASE 6 — Kubernetes (k3s/Talos) — Sandbox First, Then Production

*Future Phase*

  

**Objective:** Introduce Kubernetes as an isolated learning cluster. No production services migrate until the cluster is fully understood and stable. Build skills deliberately — sandbox phase is complete before any production workload is considered.

  

> **LOCKED APPROACH:** Sandbox first. Production services stay on Docker until k3s is proven stable and operator is comfortable. The sandbox cluster runs in parallel — failure there has zero impact on running services. Only after sandbox phase is complete will selective production migration begin.

  

> **SCOPE NOTE:** Phase 6 is a dedicated sub-project with its own runbook. It is listed here for architectural awareness — v3 hardware and network design consciously leaves headroom for it. Do not start Phase 6 until Phase 5 is fully stable.

  

**Planned Architecture:**

- Talos Linux VMs on both Proxmox nodes

- MS-A2: Talos control plane VM + Talos worker 1

- Optiplex: Talos worker 2

- Storage: Longhorn for PVCs inside the cluster

- Ingress: Traefik ingress controller (familiar from Docker context — same mental model)

- GitOps: Flux or ArgoCD

- Tooling to learn: Helm, kubectl, Longhorn, cert-manager, Flux/ArgoCD, Terraform, Talos

  

**Sandbox Phase (No Production Traffic):**

- Stand up cluster, learn core primitives: pods, deployments, services, namespaces

- Deploy test workloads — nothing that matters if it breaks

- Learn ingress controllers, PVC binding, Helm charts, operators

- Deliberately break things and recover — that is the point of the sandbox

  

**Production Migration Candidates (Post-Sandbox Only):**

- Immich — benefits from HA and operator-managed upgrades

- Authentik — IdP should be highly available

- Beszel / Uptime Kuma — monitoring infrastructure

- Traefik — already the ingress controller in k3s, natural fit

  

**Intentionally Staying in Docker:**

- ARR stack (Sonarr, Radarr, Prowlarr, Bazarr) — hardlinks and atomic moves make k8s messy

- qBittorrent + Gluetun — VPN killswitch model does not translate cleanly to k8s networking

- Books stack (CWA, ABS, Shelfmark) — ingest/hardlink workflows are filesystem-dependent

  

> **⚠ NOTE:** Do not rush Phase 6. A broken k3s cluster on top of an unstable foundation helps nobody. Phase 5 fully stable — reliable backups, clean monitoring, solid documentation — is the only acceptable entry point for Phase 6.

  

---

  

## SECTION 8 — KEY DECISIONS LOG

  

Decisions made during architecture planning, with rationale. Reference this before second-guessing a choice.

  

| Decision | Choice Made | Rationale |

|---|---|---|

| NAS platform | Unraid 7.2.3 | Hybrid ZFS+parity fits risk tolerance; mixed drive sizes; ZFS where it counts |

| TrueNAS | Retired | Moved to dedicated NAS host; Unraid replaces SCALE |

| Reverse proxy | Traefik (replaces NPM) | k3s ingress alignment; wildcard cert; Docker labels; more scalable |

| Container management | Dockman (carries forward from v2) | Individual container restart without stack disruption; Lens/Headlamp for k8s later |

| UID/GID for services | 2000:2000 | Clean break from v2; fresh NFS exports; no migration of old ownership mess |

| Dual Radarr instances | Radarr 1080p + Radarr 4K | 4K WebDL for local Infuse/Apple TV 4K viewing; 1080p for shared users to avoid transcoding |

| Dual Sonarr instances | Sonarr TV + Sonarr Anime | Anime needs different quality profiles and release group logic; single instance gets messy |

| /data unified mount | Single NFS mount at /data | All media + downloads on one filesystem — required for hardlinks to work across ARR and torrent stack |

| Ebook seeding fix | Hardlink via Shelfmark → ingest | CWA deleting ingest broke qBit seeding in v2; hardlink means CWA deletes copy, original seeds on |

| Plex deployment | Unraid native Docker container | QuickSync via i5-13400 iGPU — better than Radeon 680M for transcoding; no NFS hop; no iGPU passthrough complexity on MS-A2 |

| NAS CPU | i5-13400 (sell i5-13600) | Unraid is IO-bound not compute-bound; recoup cost toward MS-A2 |

| NAS RAM | 32GB DDR4 (sell 2x 16GB) | No heavy VM workloads on Unraid; 32GB is generous for pure NAS duty |

| Cache pool | Not installed at launch | Downloads bypass cache (hardlink rule); appdata on Docker VM local disk; no real workload justifies cache at launch. Add 2x 512GB BTRFS RAID1 later if needed. |

| Downloads share | Parity array direct — no cache | Downloads and media must be on same filesystem. Cache involvement breaks hardlinks. Non-negotiable. |

| Plex external access | Port forward 32400 direct | Plex handles TLS natively; adding Traefik proxy adds complexity for zero security gain; simple wins |

| Pangolin | Rejected | VPS relay adds latency for Plex streaming; CF Tunnel is free and works for non-streaming services |

| Docker VM count | Single docker-prod-01 VM | One VM for all media/app containers. Multi-VM Docker solves HA poorly and gets torn down when k3s arrives anyway. |

| k3s approach | Sandbox first, then selective production | No production services migrate until cluster is proven stable. Sandbox runs in parallel with zero production impact. |

| DNS | AdGuard stays | Works; ad-blocking is a bonus; no reason to migrate |

| Cache pool drives | Mirrored NVMe (never single) | Single cache SSD failure before mover = data loss; always mirror |

| Proxmox boot | ZFS RAID-1 mirror on MS-A2 | Single NVMe was the main fragility in v2; fixed |

| VLAN 1 for management | Rejected — use dedicated VLAN ID | VLAN 1 is default untagged; management should not be accidental landing zone |

| Dual parity on array | Yes (2 parity drives) | 5+ data drives warrants dual parity; single parity is a risk |

| SkyHawk drives in NAS | Rejected — Synology cold storage only | Surveillance firmware; not appropriate for NAS parity array |

| Barracuda in NAS | Rejected — retired | Desktop drive; not rated for always-on NAS duty |

| qBitrr replaces qbit-automation | qBitrr (single install, web UI) | Manages all 4 ARR instances from one place. Web UI simplifies troubleshooting. Custom Python sidecar + cron was over-engineered. |

| Cockpit on docker-prod-01 | Installed alongside Docker | Day-to-day management: disk usage, network, updates, logs, file browser. Low overhead, high operational value. |

| Proxmox cluster approach | Cluster without HA + QDevice on Pi | Cluster = unified UI. HA rejected — Optiplex cannot handle MS-A2 workload. QDevice prevents split-brain. |

| UPS graceful shutdown | NUT server on Unraid, clients on Proxmox nodes | Fully automated shutdown on power loss. Proxmox nodes shut down first, Unraid last. Works unattended. |

| IP addressing scheme | 192.168.x.x (carry forward) | UDM-SE already at 192.168.10.1 — migrating to 10.0.x.x breaks existing config unnecessarily |

| VLAN 30 name | Services (was Servers) | Services more accurately describes the workload — VMs and containers, not physical servers |

| Authentik on LXC vs VM | Dedicated VM (auth-prod-01) | LXC officially removed from Proxmox community scripts May 2025 — frequent breakage, 14GB RAM build requirement. VM is the only stable option. |

| Immich on shared VM vs dedicated | Dedicated VM (immich-prod-01) | ML worker causes CPU spikes during indexing. Isolated VM allows resource caps without affecting media stack. |

| Secondary AdGuard location | LXC on pve-prod-02 (dns-prod-02) | Two LXCs on different Proxmox nodes = DNS survives either node going down. Pi freed for QDevice + monitoring only. |

| Container VLAN IPs | No — only VMs/LXCs get IPs | Docker containers share the host VM IP. Traefik routes by hostname. Individual container IPs were a planning mistake caught early. |

  

---

  

## SECTION 9 — NAMING CONVENTIONS, REPOS & FOLDER STRUCTURE

  

### 9.1 Host Naming Convention

  

**Format:** `{role}-{env}-{index}` — lowercase, hyphens only, DNS-safe everywhere.

  

| Host | Name | Notes |

|---|---|---|

| Minisforum MS-A2 (Proxmox primary) | pve-prod-01 | pve = standard Proxmox abbreviation |

| Optiplex 3070 Micro (Proxmox secondary) | pve-prod-02 | |

| Unraid NAS | nas-prod-01 | 192.168.10.10 |

| Raspberry Pi 4B | pi-prod-01 | QDevice + Uptime Kuma + Beszel server. 192.168.10.20 |

| Docker host VM (pve-prod-01) | docker-prod-01 | All media/app containers. 192.168.30.11 |

| Authentik VM (pve-prod-01) | auth-prod-01 | Dedicated VM. 192.168.30.13 |

| Immich VM (pve-prod-01) | immich-prod-01 | Isolated for resource tuning. 192.168.30.14 |

| PBS VM (pve-prod-02) | pbs-prod-01 | 192.168.30.12 |

| Primary AdGuard LXC (pve-prod-01) | dns-prod-01 | 192.168.30.10 |

| Secondary AdGuard LXC (pve-prod-02) | dns-prod-02 | 192.168.30.15 |

| Future k3s control plane VM | k3s-ctrl-lab-01 | lab env — not prod |

| Future k3s worker VMs | k3s-work-lab-01, k3s-work-lab-02 | one per Proxmox node |

  

> **CONTAINER NAMING:** Containers use app name only — no env suffix needed. Where multiple instances of the same app exist, use a descriptor not a number: `sonarr-tv` and `sonarr-anime`, `radarr-1080p` and `radarr-4k`. Numbers are for identical instances; descriptors are for distinct-purpose instances.

  

---

  

### 9.2 Git Repositories

  

Two repositories. Keep them strictly separate — docs and IaC have different audiences, different change cadences, and different sensitivity levels.

  

#### Repo 1: homelab-docs (Obsidian → GitHub, private)

  

Knowledge hub. Architecture decisions, service documentation, runbooks, phase notes. Written in Markdown, managed in Obsidian, pushed to GitHub.

  

| Path | Contents |

|---|---|

| README.md | Entry point — what this repo is, how to navigate it |

| architecture/overview.md | High-level topology diagram and narrative |

| architecture/network.md | VLANs, firewall rules, DNS design |

| architecture/storage.md | Unraid pools, share layout, /data structure |

| architecture/compute.md | Proxmox layout, VM/LXC inventory |

| architecture/decisions-log.md | ADR-style record of every architectural decision and rationale |

| hardware/inventory.md | Every device, specs, role, purchase date |

| hardware/rack-layout.md | Physical rack diagram and cable notes |

| services/media/ | plex.md, sonarr.md, radarr.md, arr-stack.md, etc. |

| services/books/ | cwa.md, audiobookshelf.md, shelfmark.md |

| services/infra/ | traefik.md, authentik.md, adguard.md, cloudflare-tunnel.md |

| services/monitoring/ | beszel.md, uptime-kuma.md, healthchecks.md |

| services/photos/ | immich.md |

| networking/vlan-firewall-rules.md | Full firewall rule set documentation |

| networking/dns-rewrites.md | All AdGuard DNS rewrite entries |

| networking/wireguard-vpn.md | WireGuard config on UDM-SE |

| runbooks/backup-restore.md | Step-by-step restore procedures for each tier |

| runbooks/drive-replacement.md | Unraid drive swap procedure |

| runbooks/new-service-checklist.md | Standard checklist for adding any new service |

| phases/ | One .md per phase — build notes, decisions made, issues encountered |

| kubernetes/ | Placeholder — populate during Phase 6 |

  

#### Repo 2: homelab-infra (IaC/configs, cloned onto docker-prod-01 at /opt/stacks)

  

Source of truth for all compose files, scripts, and non-secret configs. Never commit `.env` files with real secrets — use `.env.example` with placeholder values and add `.env` to `.gitignore`.

  

| Path | Contents |

|---|---|

| README.md | Repo overview, setup instructions, .env guidance |

| stacks/arr/compose.yaml | sonarr-tv, sonarr-anime, radarr-1080p, radarr-4k, prowlarr, bazarr, profilarr, maintainerr, seerr, tautulli, flaresolverr |

| stacks/books/compose.yaml | cwa, audiobookshelf, shelfmark |

| stacks/torrent/compose.yaml | gluetun, qbittorrent, qbitrr |

| stacks/traefik/compose.yaml | traefik only |

| stacks/cloudflared/compose.yaml | cloudflared only |

| stacks/auth/compose.yaml | authentik server, worker, postgresql, redis |

| stacks/photos/compose.yaml | immich server, ml, redis, postgresql |

| stacks/monitoring/compose.yaml | beszel |

| stacks/dns-sync/compose.yaml | adguardhome-sync |

| stacks/dashboard/compose.yaml | homarr |

| stacks/dockman/compose.yaml | dockman |

| scripts/backup-docker.sh | Hardened rsync — covers /opt/appdata AND /opt/stacks to NAS. Mountpoint check, lockfile, Healthchecks ping. |

| scripts/backup-plex-db.sh | Plex DB stop/backup/start with EXIT trap |

| configs/traefik/ | Static config, dynamic config, middlewares |

| configs/adguard/ | DNS rewrites export for version control |

| .env.example | All required env vars with placeholder values only — never real secrets |

| .gitignore | .env files, /opt/appdata, any file matching *secret* or *password* |

  

> **SECRET MANAGEMENT:** Each stack directory gets its own `.env` file alongside `compose.yaml`. Real `.env` files are gitignored — never committed. `.env.example` in the repo root shows all required variables with placeholder values. This is the correct pattern until a dedicated secrets manager is warranted.

  

> **STACK GROUPING RULE:** Containers share a compose file only if they are functionally coupled — they need each other's networks, depend on each other, or are always started and stopped together. ARR stack is one compose. Traefik, cloudflared, and dockman are each their own compose. When in doubt, separate.

  

---

  

### 9.3 Docker Host Folder Structure

  

`/opt` on docker-prod-01. `/opt/stacks` is the homelab-infra git repo clone. `/opt/appdata` is gitignored and covered by the backup script.

  

| Path | Purpose |

|---|---|

| /opt/stacks/ | homelab-infra git repo clone — all compose files live here |

| /opt/stacks/{stack}/compose.yaml | One compose per stack |

| /opt/stacks/{stack}/.env | Stack secrets — gitignored, never committed |

| /opt/appdata/ | All container persistent data — NOT in git, rsync-backed to NAS nightly |

| /opt/appdata/sonarr-tv/ | Sonarr TV instance appdata |

| /opt/appdata/sonarr-anime/ | Sonarr Anime instance appdata |

| /opt/appdata/radarr-1080p/ | Radarr 1080p instance appdata |

| /opt/appdata/radarr-4k/ | Radarr 4K instance appdata |

| /opt/appdata/{app}/ | One directory per container, named to match container name |

| /opt/scripts/ | Backup scripts, utilities — mirrored in homelab-infra repo |

| /data/ | NFS mount point — maps to /mnt/user on nas-prod-01 |

  

> **⚠ BACKUP SCRIPT SCOPE:** `backup-docker.sh` must cover BOTH `/opt/stacks` AND `/opt/appdata`. Stacks are in git but belt-and-suspenders backup is worth it. Appdata is not in git and is the only copy of container state — it is the more critical of the two. Both paths rsync nightly to NAS /backups share on the ZFS mirror pool.

  

---

  

*Homelab v3.0 Master Roadmap | v7.0 | Living Document — Update as phases complete*