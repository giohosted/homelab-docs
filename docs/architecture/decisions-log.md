# Architecture Decisions Log

**Last Updated:** 2026-11-13

---

## Hardware

### NAS Motherboard — ASUS TUF Gaming Z690-Plus WiFi D4

**Decision:** Replace original Gigabyte B760M DS3H DDR4 with ASUS TUF Gaming Z690-Plus WiFi D4.

**What was considered:**

- Gigabyte B760M DS3H DDR4 (original plan — already owned)
    
- MSI PRO B660M-A DDR4 (found in IT closet, spare)
    
- ASUS TUF Z690-Plus WiFi D4 (purchased used ~$100 on Reddit r/hardwareswap)
    

**Why:** The Gigabyte board has only 1x usable PCIe slot (one x16 physical slot, two x1 slots). The NAS needs to run both the LSI SAS 9120-8i HBA (x8 card, requires x8 or x16 slot) and the 10GbE SFP+ NIC (requires x4 minimum) simultaneously — physically impossible on the Gigabyte board. The MSI board was ruled out due to known Realtek NIC driver issues in Linux. The ASUS TUF Z690 has a second x16 physical slot wired at x4 from the chipset, which fits both cards with no conflict. ATX form factor, solid VRM, Intel 2.5GbE onboard, and BIOS already updated for 13th gen.

---

### NAS 10GbE NIC — Dell Intel X710-DA2

**Decision:** Dell Intel X710-DA2 dual-port SFP+ NIC (~$25 eBay).

**What was considered:**

- Intel X520 (older, very common in homelabs)
    
- Dell Intel X710-DA2 (newer chipset, dual port)
    

**Why:** X710 has a newer chipset than the X520 with better long-term driver support in Linux. The initial decision valued a second port for expansion. **Current implementation:** Port 1 is a dedicated DAC to pve-prod-01; Port 2 is a DAC to USW-Pro-Max-24 for the NAS Services VLAN (`192.168.30.16`). The older single-port-in-use description is superseded.

---

### NAS CPU — i5-13400 over i5-13600

**Decision:** Keep i5-13400 for the NAS, sell the i5-13600.

**Why:** Unraid is IO-bound, not compute-bound. The performance delta between the two chips is irrelevant for a NAS workload. Selling the i5-13600 partially offsets the MS-A2 cost. The i5-13400 also has Intel UHD 730 iGPU for Plex QuickSync hardware transcoding — which is the only compute-heavy task the NAS does.

---

### NAS RAM — 32GB, sold extra sticks

**Decision:** Keep 32GB DDR4 for the NAS, sell the extra 2x 16GB sticks.

**Why (original sizing):** No heavy VM workloads originally ran on Unraid, and Plex used QuickSync. 32 GB was considered sufficient for the NAS role; selling the extra sticks recouped cost. **Subsequent change:** PBS also moved to an Unraid Docker container in November 2026. Actual live memory sizing after that change is a separate operational check.

---

### NAS Chassis — Rosewill RSV-L4412U

**Decision:** Rosewill RSV-L4412U 4U 12-bay chassis (~$345).

**Why:** 12 drive bays accommodates current drive inventory (5x 12TB + 5x 4TB + 2x 6TB) with room to expand. 4U rackmount fits the open-frame rack plan. Includes sliding rails which is important for accessing rear cabling in a rack.

---

### Primary Compute — Minisforum MS-A2

**Decision (original plan):** Minisforum MS-A2 (Ryzen 9 7945HX, planned 64 GB DDR5) as primary Proxmox node. **Current installed RAM is 32 GB**, as reported by current inventory and live Proxmox; the 64 GB figure below was an original sizing decision, not a verified installed configuration.

**Why:** 16-core/32-thread CPU with strong single-thread performance handles multiple concurrent VMs and LXCs without breaking a sweat. Dual 10GbE SFP+ built-in eliminates need for a separate NIC for storage link. Compact form factor fits on a 2U shelf in the rack. 64GB DDR5 is sufficient for all planned workloads with headroom; max is 96GB.

---

### MS-A2 Boot — ZFS RAID-1 Mirror (Samsung + Sabrent NVMe)

**Decision:** Two NVMe drives in ZFS RAID-1 mirror configured in the Proxmox installer.

**Why:** Single NVMe boot was the primary fragility in v2 — one drive failure killed the entire hypervisor. Mirrored boot means a single drive can fail and Proxmox keeps running. Selected Proxmox installer ZFS RAID-1 option at install time; no additional configuration needed.

---

### Secondary Compute — Dell OptiPlex 3070 Micro (historical; replaced November 13, 2026)

**Historical decision (superseded):** Use existing OptiPlex 3070 Micro as pve-prod-02 rather than building a second full Proxmox host from spare parts. This was the original deployment, not the current node.

**What was considered:**

- Building a second full node using spare i5-13600, spare mobo, and extra 32GB RAM
    
- Using the Optiplex as-is (free from work)
    

**Why:** The Optiplex's workload is intentionally light — PBS VM and one AdGuard LXC. HA is explicitly not enabled in this design (Optiplex cannot absorb MS-A2's workload anyway), so a matched second node provides no real benefit. Building a second powerful node means higher idle power draw and cost for no meaningful gain. Selling the i5-13600 and extra RAM recoups ~$250-300. The Optiplex is free and perfectly adequate for its role.

---

### Original OptiPlex RAM — Left at 16GB (historical)

**Historical decision (superseded):** Leave the original OptiPlex 3070 Micro at 16 GB DDR4 (2×8 GB Micron). The current replacement `pve-prod-02` has 48 GB.

**Why at the time:** The then-current workload (PBS VM + AdGuard LXC + Immich VM) fit within 16 GB. PBS was later moved to Unraid Docker, CT 102 moved to the new pve-prod-03, and Immich moved to the replacement Dell.

---

### Replace pve-prod-02 with Dell OptiPlex 5000 SFF — November 13, 2026

**Decision (implemented):** Retire the original OptiPlex 3070 Micro and replace `pve-prod-02` with a Dell OptiPlex 5000 SFF (i7-12700, 48 GB DDR4, two 512 GB NVMe drives in ZFS RAID-1). Reuse hostname `pve-prod-02.local` and management IP `192.168.10.12/24` only after removing the old hardware from cluster membership.

**Why:** More CPU/RAM headroom for Immich's containerized ML work, improved boot-disk redundancy, and preservation of established node names and integrations without duplicate host identities.

**Implementation:** Evacuated all guests and destroyed retired PBS VM 103 on the old node; powered it off; removed it with `pvecm delnode pve-prod-02`, deleted the verified-empty stale node config directory, and removed obsolete `local-lvm`. Installed Proxmox VE 9.2 (later updated to 9.2.26/kernel 7.0.14-28-pve) on a ZFS mirror; joined new hardware to `homelab-v3`. The replacement has a VLAN-aware management bridge, two synchronized EFI boot partitions, and active local-zfs. Verified healthy three-node quorum, ZFS mirror, SMART health, updated packages, and host monitoring.

**Limits:** Physical one-drive boot failover has not been tested. The former OptiPlex 3070 Micro must remain isolated or be wiped before reuse because it retains its old hostname/cluster identity on disk.

---

### Third Compute — Lenovo ThinkCentre M90q (`pve-prod-03`) — November 2026

**Decision (implemented):** Add a Lenovo ThinkCentre M90q Tiny (i7-10700, 24 GB RAM, mirrored 2×512 GB NVMe) as the third Proxmox voting node at `192.168.10.13/24`.

**Why:** Increase compute flexibility and establish a healthy 3-node quorum without retaining the earlier Raspberry Pi QDevice. Spread primary and secondary AdGuard onto separate hypervisors.

**Implementation:** Removed QDevice before adding Lenovo; configured ZFS RAID-1, VLAN-aware `vmbr0`, and joined `homelab-v3`. Moved secondary AdGuard CT 102 to this node; temporarily hosted Immich VM 105 while `pve-prod-02` was replaced. By November 13, VM 105 was moved to the new Dell, and CT 102 is the only current guest on `pve-prod-03`.

**Limits:** Proxmox HA remains disabled. Independent boot from each Lenovo SSD and the later BIOS update remain unverified.

---

### UPS — Tripp-Lite SMART1500LCDXL

**Decision:** Tripp-Lite SMART1500LCDXL 1500VA/900W (~$145). First hardware purchased.

**Why:** UPS was a non-negotiable requirement before powering any spinning-rust drives — power loss during a write on HDDs is a data corruption risk. USB interface enables NUT integration for fully automated graceful shutdown. 1500VA handles the full stack (5 devices) with capacity to spare. Purchased before any other hardware as a hard rule.

---

### Rack — StarTech 4POSTRACK18U

**Decision:** StarTech 4POSTRACK18U 18U open-frame rack (bundled with switch, $560 total).

**Why:** 18U provides enough space for all current hardware (U1-U11 occupied) plus 6 empty U's for future expansion. Open-frame is fine for a basement/closet install and simplifies cable access vs. enclosed cabinet.

---

### Switch — UniFi USW-Pro-Max-24

**Decision:** UniFi USW-Pro-Max-24 24-port 2.5GbE managed switch (bundled with rack).

**Why:** 2.5GbE on all ports future-proofs LAN connections — all current hosts support 2.5GbE or better. Stays in the UniFi ecosystem with the UDM-SE for unified management. SFP+ uplink ports used for DAC connections to UDM-SE and MS-A2.

---

## Networking

### VLAN Design — 4 VLANs

**Decision:** VLAN 10 (Management), VLAN 20 (Trusted), VLAN 30 (Services), VLAN 40 (IoT).

**Why:** Clean separation of concerns. Management VLAN isolates infrastructure devices (Proxmox hosts, NAS, switches) from everything else. Trusted is personal devices. Services is all v3 VMs and LXCs. IoT is fully isolated with internet-only access. Default VLAN 1 intentionally not used as management — devices should never accidentally land on the management VLAN.

**Historical note (2026-10-29):** This records the original four-VLAN design. VLAN 50 (Game Servers) was subsequently introduced for dedicated game-server workloads. Proxmox VM 106 (`dragonwilds-server`) has been verified with VLAN tag 50. Its complete addressing and UniFi firewall configuration remain to be verified. See `architecture/network/vlan-design.md`.

---

### No HA (High Availability)

**Decision:** Proxmox cluster without HA enabled.

**Why (current position):** Although the cluster now has three voting nodes, guest HA has **not** been designed or enabled: host capacities differ, guest disks are on local ZFS, and automatic failover behavior and recovery capacity have not been validated. A quorum majority provides cluster control-plane resilience but does not move or restart guests automatically. Clustering primarily supports centralized administration and planned migrations. If a host fails, its local-disk workloads require a deliberate recovery process.

---

### QDevice on Pi (historical; removed November 2026)

**Historical decision (superseded):** Run Proxmox QDevice on pi-prod-01 as a two-node cluster tiebreaker. **QDevice was removed before adding pve-prod-03.**

**Why:** A 2-node Proxmox cluster without a quorum device can suffer split-brain — both nodes think they're the primary and fence each other. QDevice on the Pi costs nothing (10-minute setup) and prevents this. The Pi is lightweight, always-on, and perfectly suited for this role.

---

### Three-Node Quorum Without QDevice — November 2026

**Decision (implemented):** The cluster now has voting nodes `pve-prod-01`, `pve-prod-02`, and `pve-prod-03`, one vote each; the Pi QDevice is no longer used. Two of the three nodes are required for quorum. HA remains disabled, so this decision affects cluster coordination, not automatic workload failover.

**Why:** Three dedicated Proxmox nodes provide majority quorum without a fourth external vote. This improves operational independence from the monitoring Raspberry Pi while maintaining a straightforward homelab setup.

---

### DNS — AdGuard Home (carry forward)

**Decision (current placement):** Keep AdGuard Home as the DNS solution. Primary CT 100 on `pve-prod-01` (`dns-prod-01`); secondary CT 102 on **`pve-prod-03`** (`dns-prod-02`); synchronization via adguardhome-sync.

**Why:** Works well in v2, ad-blocking is a bonus, no reason to migrate. Two instances on separate Proxmox nodes provide DNS continuity if one host is lost. Secondary AdGuard moved off the Pi in v3, originally ran on the old pve-prod-02, and was moved to pve-prod-03 during the November replacement. Pi QDevice was removed; Uptime Kuma monitoring remains on the Pi.

---

### Reverse Proxy — Traefik (replaces NPM)

**Decision:** Migrate from Nginx Proxy Manager to Traefik in v3.

**Why:** Traefik is the default ingress controller in k3s — learning it now in a Docker context pays dividends when Kubernetes is introduced in Phase 6. Docker label-based routing eliminates per-service config files. Wildcard cert via Cloudflare DNS-01 challenge covers all internal services with one cert. NPM kept running in parallel during migration until all services are confirmed on Traefik — rollback is an AdGuard DNS rewrite flip.

---

### External Access — Cloudflare Tunnel + Plex Port Forward

**Decision:** CF Tunnel for all externally exposed services except Plex. Plex gets a direct port forward on 32400.

**Why:** Cloudflare ToS prohibits video streaming through their tunnel — so Plex cannot use it. Plex handles its own TLS and relay natively, so a direct port forward is simple and proven. Everything else (Authentik, ABS, Seerr, Shelfmark) goes through CF Tunnel with no port forwarding required.

**Pangolin evaluated and rejected:** VPS relay adds latency for Plex streaming, CF Tunnel is free and works, no meaningful reason to add VPS cost.

---

## Storage

### NAS Platform — Unraid 7.2.3 (replaces TrueNAS SCALE)

**Decision:** Unraid as the NAS OS.

**Why:** Hybrid ZFS + parity array model fits the data risk tolerance perfectly. ZFS used where data integrity is non-negotiable (photos, backups). Parity array used for recoverable bulk media where mixed drive sizes and expandability matter more than ZFS guarantees. TrueNAS ran as a VM in v2 — v3 gives it a dedicated host as it should have always had.

---

### No Cache Pool at Launch

**Decision:** No NVMe cache pool installed at launch.

**Why:** Downloads must bypass cache entirely — downloads and media must live on the same filesystem for hardlinks and atomic moves to work. If downloads go through cache and media lives on the array, hardlinks break silently and cause file duplication. Container appdata lives on the docker-prod-01 local disk, not NAS. Plex transcode temp points at a local Unraid directory. No real workload justifies cache at launch. If a use case emerges: add 2x 512GB NVMe as BTRFS RAID-1 mirrored cache pool. Cache drives must always be mirrored — a single cache SSD failure before the mover runs means data loss.

---

### Downloads Share — Parity Array Direct, No Cache

**Decision:** Downloads share set to "Use cache: No" in Unraid. Writes go directly to parity array.

**Why:** Hard requirement for hardlinks. Downloads and media must be on the same filesystem. Cache involvement breaks this. Non-negotiable — violations cause silent file duplication and broken seeding.

---

### Dual Parity on Array

**Decision:** 2x WD Red Pro 12TB as parity drives (dual parity).

**Why:** With 3+ data drives, dual parity is worth the cost. Single parity means two simultaneous drive failures (one during a rebuild) kills the array. Dual parity tolerates two simultaneous failures.

---

### SkyHawk and Barracuda Drives — Not in Unraid Array

**Decision:** 4x Seagate SkyHawk 6TB go to Synology for cold backup storage only. 1x Seagate Barracuda 4TB retired.

**Why:** SkyHawk drives use surveillance firmware — not appropriate for a NAS parity array. Barracuda is a desktop drive not rated for always-on NAS duty. Neither belongs in a production array.

---

### UID/GID — 2000:2000 for All Services

**Decision:** All containers run as PUID=2000, PGID=2000. Clean break from v2.

**Why:** v2 had messy ownership from organic growth — some services ran as root, some as gio (1000:1000), inconsistent across NFS mounts. v3 starts clean with a dedicated service UID/GID that is separate from both root and the human user account. NAS share permissions set to allow 2000:2000 read/write on all service shares.

---

### Single `data` NFS Share for Media and Downloads

**Decision:** Consolidate `media` and `downloads` Unraid shares into a single `data` share with `media/` and `downloads/` as subfolders. Export only `data` via NFS to docker-prod-01.

**What was considered:**

- Separate `media` and `downloads` shares exported individually (original v3 setup)
    
- Single `data` share containing both as subfolders (chosen)
    

**Why:** Hardlinks require files to be on the same filesystem. Separate NFS mounts — even from the same NAS — are separate filesystems from the client's perspective. When Radarr imports a file from `/data/downloads` to `/data/media`, hardlinks only work if both paths resolve to the same underlying mount. With separate mounts they silently fall back to copy+delete, breaking seeding and wasting IO. A single `data` share with subfolders presents one filesystem to docker-prod-01. This is the correct pattern per the TRaSH guide for Unraid setups.

**Migration:** Existing `media` and `downloads` share contents moved into `/mnt/user/data/media/` and `/mnt/user/data/downloads/` respectively. Old shares deleted. docker-prod-01 fstab updated to mount `192.168.30.16:/mnt/user/data` → `/data`. All container paths unchanged (`/data/media/...` and `/data/downloads/...`) — no ARR reconfiguration required.

---

### NFSv3 Instead of NFSv4

**Decision:** Set Unraid's Max Server Protocol to NFSv3.

**What was considered:**

- NFSv4 (Unraid default)
    
- NFSv3 (chosen)
    

**Why:** NFSv4 uses a pseudo-root filesystem that automatically automounts sub-shares on top of a parent mount as the client traverses into them. Even after switching to a single `data` share fstab entry, NFSv4 was causing `media` and `downloads` to be automounted as separate NFS mounts on docker-prod-01 — recreating the separate-filesystem problem. NFSv3 does not have this pseudo-root behavior. Mounts are explicit and static. The `data` share mounts once at `/data` and stays that way.

---

## Services & Compute

### Plex — Unraid Native Docker Container

**Decision:** Plex runs directly on Unraid as a Docker container, not on docker-prod-01.

**Why:** i5-13400 QuickSync iGPU (Intel UHD 730) is mature, well-supported, and handles multiple simultaneous 1080p transcodes easily. Running Plex on Unraid eliminates an NFS hop — media files are local to the host doing the transcoding. No iGPU passthrough complexity on MS-A2 needed. Radeon 680M on the MS-A2 is available for future use but QuickSync is the better transcoding choice for this workload.

---

### Dual Radarr Instances — 1080p and 4K

**Decision:** Two separate Radarr instances: radarr-1080p and radarr-4k.

**Why:** 4K WebDL for local viewing via Infuse/Apple TV 4K. 1080p for shared users to avoid transcoding overhead. Single Radarr instance managing both quality tiers gets messy with quality profiles and root folders. Separate instances are clean and independently configurable.

---

### Dual Sonarr Instances — TV and Anime

**Decision:** Two separate Sonarr instances: sonarr-tv and sonarr-anime.

**Why:** Anime requires different quality profiles, release group logic, and indexer configuration vs. regular TV. A single instance handling both gets messy. Separate instances are independently configurable.

---

### qBitrr Replaces Custom qbit-automation Sidecar

**Decision:** qBitrr replaces the custom Python sidecar + cron job used in v2.

**Why:** qBitrr manages all 4 ARR instances from a single installation with a proper web UI. The v2 custom sidecar was over-engineered and fragile. qBitrr handles seeding control, MAM compliance (14-day minimum seed time), and torrent health monitoring out of the box.

---

### Authentik — Dedicated VM, Not LXC

**Decision:** Authentik runs on a dedicated VM (auth-prod-01), not an LXC.

**Why:** The Proxmox community officially removed the Authentik LXC helper script in May 2025 due to frequent breakage and a 14GB RAM requirement during build. VM is the only stable deployment option. Dedicated VM also isolates the IdP from other workloads — appropriate given Authentik is the SSO gateway for everything.

---

### Immich — Dedicated VM

**Decision:** Immich runs on a dedicated VM (immich-prod-01), not on docker-prod-01.

**Why:** The Immich ML worker causes significant CPU spikes during face detection and CLIP indexing. Isolating it on a dedicated VM allows resource caps (CPU limits, RAM limits) without affecting the media stack running on docker-prod-01.

---

### immich-prod-01 — Original Placement on Old pve-prod-02 (historical)

**Historical decision (superseded hardware):** Deploy immich-prod-01 on the original pve-prod-02 (OptiPlex 3070 Micro) instead of pve-prod-01 (MS-A2). This explains the original placement, not the final host model.

**What was considered:**

- pve-prod-01 (original plan — more powerful CPU)
    
- pve-prod-02 (chosen — more RAM headroom)
    

**Why:** At Phase 4 Wave 5 start, pve-prod-01 was already running docker-prod-01 (8GB RAM) and auth-prod-01 (4GB RAM), with docker-prod-01 confirmed to have been swapping (1GB swap usage before RAM bump). Adding a 6GB Immich VM would have pushed pve-prod-01 RAM utilization uncomfortably high. pve-prod-02 was only running PBS (2GB) and one AdGuard LXC (512MB) — ample headroom for the 6GB Immich VM. The CPU performance difference between the i5-9500T and Ryzen 9 7945HX is meaningful for ML workloads but acceptable for a homelab Immich instance where indexing runs in the background.

---

### Immich — Offline Migration to Replacement pve-prod-02 — November 13, 2026

**Decision (implemented):** Keep `immich-prod-01` as VM 105 with its existing IP `192.168.30.14`, VLAN 30, guest CPU/RAM allocation (2 vCPU/6 GiB), and Unraid photo NFS mount. Temporarily move it to `pve-prod-03` while retiring the old Dell, then perform a controlled **offline** migration to the new `pve-prod-02` with `--with-local-disks --targetstorage local-zfs`.

**Why:** Offline migration limits complexity during hardware replacement; using the same VM ID, virtual NIC, guest configuration, and IP preserves Traefik routing and application integrations. The new i7-12700 node offers stronger physical CPU capacity for future Immich ML tuning.

**Verified:** Migration task `OK` on November 13; containers running, operator-tested Immich web UI, NFS photo directories present, and a fresh VM 105 PBS snapshot backup `TASK OK` at 17:16 CST. No CPU/RAM tuning was performed during the move; full restore and independent storage-failure recovery remain untested.

---

### Single Docker Host VM

**Decision:** One docker-prod-01 VM for all media and application containers.

**Why:** Multi-VM Docker arrangements solve HA poorly and add operational complexity. All these containers will eventually migrate to k3s anyway — over-investing in a multi-VM Docker architecture that gets torn down in Phase 6 is wasteful. One clean VM, one NFS mount, one place to look when something breaks.

---

### Kubernetes — Sandbox First

**Decision:** k3s introduced as an isolated sandbox cluster in Phase 6. No production services migrate until the cluster is fully understood and stable.

**Why:** A broken k3s cluster on top of an unstable foundation helps nobody. The sandbox runs in parallel with zero impact on running services. Only after the sandbox phase is complete will selective production migration begin. ARR stack, qBittorrent/Gluetun, and books stack intentionally stay in Docker permanently — hardlink and VPN killswitch workflows do not translate cleanly to Kubernetes.

---

### Cockpit on docker-prod-01

**Decision:** Install Cockpit alongside Docker on docker-prod-01.

**Why:** Provides web-based day-to-day management of the VM — disk usage, network, logs, updates, file browser — without needing to SSH for routine tasks. Low overhead, high operational value.

---

### Bazarr — Covers Sonarr TV and Radarr 1080p Only

**Decision:** Bazarr connected to Sonarr TV and Radarr 1080p only. Sonarr Anime and Radarr 4K have no Bazarr coverage.

**What was considered:**

- Two Bazarr instances (one per Sonarr/Radarr pair)
    
- Single Bazarr covering TV + Movies only (chosen)
    

**Why:** Bazarr 1.5.6 supports only one Sonarr and one Radarr instance. Running two Bazarr instances adds operational overhead with minimal benefit — anime subtitles are typically embedded in the release or not needed at all. 4K movies are for local viewing via Infuse which handles embedded subtitles natively. The value of automatic subtitle hunting is highest for regular TV and 1080p movies.

---

## Backup

### PBS for VM/LXC Snapshots (original VM-hosted design; superseded)

**Historical decision (superseded):** Proxmox Backup Server originally ran as VM 103 on the old `pve-prod-02`, backing up guests to the Unraid NAS ZFS mirror. **VM 103 is no longer present.**

**Why:** PBS is the right tool for VM-level backups in a Proxmox environment. Running it on pve-prod-02 means the backup server is on a different node than the primary workloads — a backup server on the same node it's backing up is a single point of failure.

---

### PBS on Unraid Docker — Implemented November 3–5, 2026

**Decision (implemented):** Move PBS off VM 103 to the Docker `proxmox-backup-server` container running on `nas-prod-01`, accessible at `192.168.30.16:8007`, while retaining the Proxmox storage ID `pbs-prod-01` and the existing datastore `nas-backups`.

**Why:** Recurrent PBS failures involved an `ESTALE` stale NFS lock descriptor over the old NFSv3-to-Unraid-FUSE datastore path. The replacement Docker deployment binds directly to NAS ZFS storage at `/mnt/zfs-mirror/backups/pbs`, eliminating that client access path. This does not prove the original upstream invalidation trigger.

**Status:** November 13 scheduled backups succeeded on nodes 01 and 03; VM 105 completed a successful manual snapshot backup from the new node 02. VM 103 was destroyed before physical node replacement; its exclusion remains in the cluster's 02:00 backup job as a harmless stale rule. A full isolated VM/CT restore is still outstanding. See `runbooks/backup-schedule.md`.

---

### Appdata Backups — rsync, Separate from PBS

**Decision:** Docker appdata and compose stacks backed up independently via hardened rsync script, separate from PBS.

**Why:** PBS backs up VM disk images — it does not back up application data inside VMs. Container appdata (databases, configs) requires its own backup strategy. rsync to NAS /backups share (ZFS mirror pool) nightly, with Healthchecks.io heartbeat monitoring for silent failure detection.

---

### NAS Network Interface Design — Dual Interface, Management/Data Separation

**Decision:** nas-prod-01 uses two separate physical network interfaces for two distinct purposes. Onboard 2.5GbE → VLAN 10 (management only, Unraid UI at 192.168.10.10). X710 Port 2 SFP+ → VLAN 30 (data/NFS, at 192.168.30.16).

**What was considered:**

- Single interface on VLAN 30 for everything (management + NFS)
    
- Single interface on VLAN 10 for everything (would require firewall hole for services to reach NFS)
    
- Dual interface separating management and data planes (chosen)
    

**Why:** The NAS holds backups, photos, and all irreplaceable data. Putting the Unraid management UI on VLAN 30 alongside services means a compromised container could reach it. Putting everything on VLAN 10 requires poking holes in the management VLAN for NFS traffic, undermining its isolation. Dual interface solves both problems cleanly — services on VLAN 30 reach NFS via 192.168.30.16, the firewall hard-blocks VLAN 30 → VLAN 10, so no compromised container can ever reach the Unraid UI at 192.168.10.10. Management UI is only reachable from Trusted (VLAN 20) on TCP 443.

---

### NAS X710 Port 2 — DAC to Switch (not spare 2.5GbE PCIe NIC)

**Decision:** NAS VLAN 30 data interface uses X710 Port 2 SFP+ via DAC to USW-Pro-Max-24 SFP+ port, giving 10GbE on the NFS/Plex traffic path. Spare 2.5GbE PCIe NIC not used.

**What was considered:**

- Spare 2.5GbE PCIe NIC installed in NAS for VLAN 30 interface
    
- X710 Port 2 DAC to switch SFP+ port (chosen)
    

**Why:** The NAS data interface is the busiest path in the entire setup — NFS mounts for the ARR stack, Plex streaming, PBS backup traffic all flow through it simultaneously. 10GbE on this path is the highest-value use of the switch's remaining SFP+ port. The MS-A2 VM/LXC traffic (Traefik, ARR API calls, DNS) never approaches 2.5GbE saturation, so the MS-A2 uses its onboard 2.5GbE RJ45 for LAN instead.

---

### MS-A2 LAN Interface — Onboard 2.5GbE RJ45 (not DAC to switch)

**Decision:** MS-A2 uses its onboard 2.5GbE RJ45 port for VM/LXC LAN traffic to the switch. The 1M DAC cable originally purchased for MS-A2 → switch is repurposed for NAS X710 Port 2 → switch.

**What was considered:**

- 1M DAC from MS-A2 SFP+ to switch SFP+ (original plan)
    
- Onboard 2.5GbE RJ45 to switch RJ45 (chosen)
    

**Why:** VM/LXC workloads on pve-prod-01 (Traefik, ARR stack, Authentik, Immich, DNS) do not come close to saturating 2.5GbE. Using the DAC on MS-A2 would consume the switch's last SFP+ port for no meaningful performance gain, while the NAS data interface benefits significantly from 10GbE. Repurposing the 1M DAC to the NAS eliminates the need to purchase an additional cable.

---

### pi-prod-01 — VLAN 10 (Management), Not VLAN 30

**Decision:** pi-prod-01 remains on VLAN 10 at 192.168.10.20.

**What was considered:**

- VLAN 30 (Services) — would co-locate Pi with services it monitors, but requires explicit firewall rule for QDevice traffic to reach Proxmox nodes on VLAN 10
    
- VLAN 10 (Management) — chosen
    

**Why (original decision):** Keeping infrastructure monitoring and the then-active QDevice on VLAN 10 avoided punching additional VLAN 30 → VLAN 10 firewall exceptions. Since the QDevice was removed in November 2026, the Pi remains on VLAN 10 for monitoring and management reachability. Any current Beszel deployment should be checked separately; Uptime Kuma is confirmed operational.

---

### IP Addressing Scheme — 192.168.x.x Retained

**Decision:** Keep 192.168.x.x addressing. VLAN 10 = 192.168.10.0/24, VLAN 20 = 192.168.20.0/24, VLAN 30 = 192.168.30.0/24, VLAN 40 = 192.168.40.0/24.

**What was considered:**

- 10.0.x.x (e.g. 10.0.10.0/24 per VLAN)
    
- 10.10.x.x (e.g. 10.10.10.0/24 per VLAN — aesthetically cleaner, NAS would be 10.10.10.10)
    
- 192.168.x.x (chosen — carry forward)
    

**Why:** No practical benefit to switching. Both 10.x.x.x and 192.168.x.x are RFC 1918 private ranges with identical routing behavior. 192.168.x.x gives 254 hosts per VLAN which is more than sufficient for a homelab. The UDM-SE is already configured, v2 devices are already on 192.168.20.x, and all documentation is already written around 192.168.x.x. Switching creates documentation churn, reconfiguration work, and risk to the live v2 setup for zero functional gain.

---

### TrueNAS Media Migration — VM on pve-prod-01 (Option 2)

**Decision:** Migrate 3.32TiB of media from TrueNAS RAIDZ1 to Unraid by spinning up a temporary TrueNAS VM on pve-prod-01 with LSI HBA passthrough, importing the existing pool, and copying directly to Unraid over the network.

**What was considered:**

- Boot existing v2 Proxmox SSD on MS-A2 hardware — rejected due to Secure Boot and MAC address/hardware mismatch issues
    
- Copy TrueNAS → Synology → Unraid (double transfer, ~28-30 hours total) — rejected as unnecessarily slow
    
- TrueNAS VM on pve-prod-01 → Unraid direct copy (~14-15 hours, single transfer) — chosen
    

**Why:** ZFS RAIDZ1 cannot be natively read by Unraid. A temporary TrueNAS VM with HBA passthrough is the cleanest single-transfer path — import the existing pool, copy to Unraid over LAN, decommission the VM. Avoids double transfer time and keeps the LSI HBA in use without extra steps. v2 Proxmox host Gigabyte mobo was sold before this decision was finalized, making Option 1 (booting old SSD on new hardware) the only alternative, which was rejected due to hardware compatibility issues.

---

### Books Stack — Shelfmark Ebook Hardlink Volume Mount

**Decision:** Mount the ebook qBittorrent download folder into Shelfmark at its exact client-side path (`/data/downloads/books/ebooks/downloads:/data/downloads/books/ebooks/downloads`), not a remapped path.

**What was considered:**

- Mounting at a simplified path (e.g. `/downloads/ebooks`) inside the container
    
- Mounting at the exact client path (chosen)
    

**Why:** Per Shelfmark documentation, the torrent client directory must be mounted at exactly the same path inside the container as it appears to qBittorrent. If paths differ, hardlink creation fails silently — Shelfmark falls back to copy, the original file in downloads gets deleted after import, and qBit's seeding torrent is broken. Matching the path exactly is the only reliable way to ensure hardlinks work.

---

### Books Stack — Ebook Hardlink to Ingest Despite CWA Warning

**Decision:** Enable Shelfmark's "Hardlink Book Torrents" toggle for ebooks, with destination set to the CWA ingest folder.

**What was considered:**

- Copy mode (no hardlink) — destination is ingest folder, CWA deletes after import, original in downloads also gone, seeding broken
    
- Hardlink mode with ingest as destination (chosen)
    

**Why:** Shelfmark warns against hardlinking when the destination is a library ingest folder, because CWA deletes files from ingest after import. The warning assumes the ingest copy IS the only copy. In this setup it is not — the original lives in qBit's download folder, and Shelfmark hardlinks a second inode to ingest. CWA deletes the ingest hardlink; the original in downloads is untouched. Seeding continues. The warning does not apply to this pattern.

---

### Books Stack — qBitrr ManagedCategories vs ARR Section Management

**Decision:** Add `audiobooks` and `ebooks` to qBitrr's `ManagedCategories`. Do not add the ARR categories (`sonarr-tv`, `sonarr-anime`, `radarr-1080`, `radarr-4k`) to `ManagedCategories`.

**Why:** These are two different mechanisms in qBitrr. ARR instances are managed via their own `[Sonarr-*]` and `[Radarr-*]` config sections with `Managed = true` — this handles search, import, and torrent health for those categories. `ManagedCategories` is for qBit-level seeding rule management applied directly to torrent categories, independent of ARR. Only `audiobooks` and `ebooks` need this — they have MAM-specific 14-day seeding rules to be configured in Phase 5. The ARR categories are already fully managed through their own config sections.

---

### Books Stack — MAM Seeding Rules Deferred to Phase 5

**Decision:** Configure MAM-specific seeding rules in qBitrr during Phase 5, not during books stack deployment.

**Why:** The seeding rules are tracker-scoped — they apply only to torrents from MAM, not from AudiobookBay or other sources. No active book torrents exist at deployment time. Configuring tracker-specific logic before any torrents are being managed adds complexity with no immediate benefit. Phase 5 operational hardening is the correct time to set these up alongside the full seeding policy review.

---

### OIDC Admin Group — `admins` for Shelfmark and ABS

**Decision:** Use the existing `admins` Authentik group for both Shelfmark and ABS admin authorization, rather than creating service-specific groups.

**Why:** A `shelfmark-admins` or `abs-admins` group would need to be created and populated from scratch. The existing `admins` group already contains the correct users and is used consistently across other services. Creating per-service admin groups adds Authentik management overhead with no meaningful security benefit for a single-admin homelab.

---

### qBitrr OIDC — HTTP Redirect URI Required Alongside HTTPS

**Decision:** Register both `https://qbitrr.giohosted.com/signin-oidc` and `http://qbitrr.giohosted.com/signin-oidc` as allowed redirect URIs in the Authentik provider.

**Why:** qBitrr sends an HTTP redirect URI in its OIDC authorization request despite the app being served over HTTPS via Traefik. Authentik rejects the request if the redirect URI doesn't exactly match a registered value. Adding both HTTP and HTTPS variants satisfies Authentik's validation without modifying qBitrr's behavior. The HTTP redirect URI is only used during the OIDC handshake on the private LAN — it does not expose the app over plain HTTP externally.

---

### NAS ISO Storage — Separate DAC and Services-VLAN Paths

**Decision (current scope after November 13, 2026):** Keep `nas-isos` restricted to `pve-prod-01` via `10.0.0.2` (direct DAC), and use `nas-isos-p2` via `192.168.30.16` on **both** `pve-prod-02` and `pve-prod-03`. Both IDs expose the same `/mnt/user/isos` export.

**What was considered:**

- Editing the cluster-wide `nas-isos` entry to use `192.168.30.16` for all nodes — rejected, would break pve-prod-01's DAC link NFS mount
    
- Adding a duplicate storage entry scoped per-node (chosen)
    

**Why:** The `10.0.0.2` point-to-point DAC address is reachable only from pve-prod-01. Nodes 02 and 03 instead reach Unraid's NFS export at `192.168.30.16` over the Services-VLAN path. Preserving distinct storage definitions keeps the primary node's DAC path intact without duplicating any ISO data. The ID `nas-isos-p2` is retained for compatibility even though it now serves both nodes 02 and 03.

---

### Immich — Postgres on Local VM Disk, Not NFS

**Decision (current storage):** Keep Immich Postgres at `/opt/appdata/immich/postgres` inside VM 105 on its **local ZFS-backed system disk** (`local-zfs` on new pve-prod-02), never on the NFS-mounted photos share. Its original location used `local-lvm` on the retired Dell; the VM disk was migrated to ZFS during the node replacement.

**Why:** Immich explicitly does not support NFS for the Postgres database. NFS latency and locking behavior cause Postgres stability issues. The photo files themselves (upload location) are correctly stored on NFS — only the database is kept local.

---

### Immich — DB Restore via psql CLI, Not Immich UI

**Decision:** Restore Immich database backup directly via `psql` command rather than using the Immich maintenance mode UI restore.

**What was considered:**

- Immich maintenance mode UI restore (attempted first — failed silently mid-migration)
    
- Direct psql restore via `gunzip | docker exec psql` (chosen)
    

**Why:** The Immich UI restore was interrupted mid-migration, leaving the database in an inconsistent state with empty tables. The direct psql restore is reliable, provides clear output, and bypasses the UI entirely. For future restores, always use the CLI method.

---

### Immich — Official Compose Volume Mapping (`:/data`)

**Decision:** Use the official Immich compose volume mapping (`${UPLOAD_LOCATION}:/data`) rather than any non-standard internal path.

**What was considered:**

- Non-standard internal path `/usr/src/app/upload` (used initially — caused DB path mismatch)
    
- Official mapping `:/data` (chosen)
    

**Why:** Immich stores asset paths in its database using the internal container path. Using a non-standard internal path creates a permanent divergence from Immich's own tooling and documentation — any future migration, restore, or upgrade that references the official compose will result in broken asset paths. Using the official mapping ensures the database paths match what Immich and its backup/restore tooling expect, making future migrations straightforward.

---

### docker-prod-01 RAM — Bumped from 6GB to 8GB

**Decision:** Increase docker-prod-01 RAM allocation from 6GB to 8GB during Phase 4 Wave 5.

**Why:** Verified via `free -h` that docker-prod-01 had 1GB of active swap usage — evidence of real memory pressure during ARR stack operation (search bursts, simultaneous container startup). 8GB provides comfortable headroom for the full stack (Traefik, ARR stack, books stack, torrent stack) with room to grow. auth-prod-01 was also checked and found to have only 268KB swap — no bump needed there.

---

## AI Observer & MCP

### AI Observer — Dedicated VM, Not Shared Docker Host

**Decision:** Deploy AI Observer on a dedicated Ubuntu Server VM (`observer-prod-01`, VM 107) on pve-prod-01 instead of placing AI integration services on docker-prod-01.

**What was considered:**

- Deploying the MCP server alongside existing production application containers on docker-prod-01.
    
- Deploying AI Observer on its own lightweight VM (chosen).
    

**Why:** AI integrations introduce another path into infrastructure management APIs. Separating them from the main application Docker host reduces shared administrative and application exposure, creates an independent operating-system boundary, and allows credentials, networking, updates, and service management to be handled separately.

The dedicated VM also provides a controlled location for adding future infrastructure integrations without modifying the primary production Docker environment.

**Implemented configuration:** 2 vCPU, 2 GiB RAM, 20 GiB disk, VLAN 30, IP `192.168.30.17`.

The VM and its Proxmox configuration were verified through the live Homelab Proxmox integration.

---

### Proxmox MCP — Read-Only First

**Decision:** Deploy Proxmox MCP with read-only capabilities and defer all write, delete, and execution functionality.

**What was considered:**

- Enabling management operations during the initial integration.
    
- Restricting the integration to observation, diagnostics, and auditing (chosen).
    

**Why:** The immediate goal is to allow ChatGPT to inspect infrastructure, diagnose problems, and provide recommendations without introducing the risk of accidental or unauthorized state changes.

Read-only access also provides an opportunity to evaluate MCP reliability, tool behavior, information exposure, and day-to-day usefulness before considering expanded permissions.

**Implemented controls:**

- `PROXMOX_ALLOW_WRITES=false`
    
- `PROXMOX_ALLOW_DELETES=false`
    
- `PROXMOX_ALLOW_EXEC=false`
    
- Dedicated Proxmox account with `PVEAuditor` permissions.
    

The read-only integration was successfully tested through authenticated MCP tool calls and through the ChatGPT plugin.

**Future consideration:** Write permissions may be evaluated after the read-only deployment has demonstrated reliable behavior. Any expansion requires a separate permissions review, narrowly scoped capabilities, explicit user authorization for consequential changes, and verification of the actual available operations.

No write access is currently authorized or implemented.

---

### Proxmox Access — Dedicated API Identity and Targeted Firewall Exception

**Decision:** Use a dedicated Proxmox API identity with audit-only privileges, combined with a narrowly scoped UniFi firewall exception.

**What was considered:**

- Using an existing Proxmox administrative account.
    
- Granting the observer broad access to VLAN 10.
    
- Creating a dedicated read-only account and permitting only the required API connection (chosen).
    

**Why:** A dedicated identity prevents the AI integration from inheriting human administrator privileges. Audit-only permissions limit the impact of a compromised token or unintended MCP request.

Network access is restricted independently of API permissions, so the observer cannot rely on broad Services-to-Management connectivity.

**Implemented access:**

- Account: `observer@pve`
    
- API token: `observer@pve!proxmox-mcp`
    
- Role: `PVEAuditor`
    
- Source: `192.168.30.17`
    
- Destination: `192.168.10.11`
    
- Protocol: TCP
    
- Port: `8006`
    

The MCP server also validates the Proxmox HTTPS endpoint using a pinned TLS certificate fingerprint.

**Verification:** Authenticated Proxmox API requests succeeded after the targeted firewall exception was implemented. A subsequent end-to-end ChatGPT MCP query successfully returned live cluster data.

The precise current UniFi rule configuration remains subject to periodic firewall audits.

---

### OpenAI Secure MCP Tunnel — Outbound-Only Access

**Decision:** Use OpenAI Secure MCP Tunnel to connect ChatGPT to the privately hosted Proxmox MCP server.

**What was considered:**

- Exposing the MCP server through a public HTTPS endpoint.
    
- Routing the MCP server through the existing Traefik and Cloudflare Tunnel infrastructure.
    
- Using OpenAI Secure MCP Tunnel with the MCP server bound to localhost (chosen).
    

**Why:** An outbound tunnel avoids introducing a publicly reachable management-related service and eliminates the need for inbound internet port forwarding.

It also keeps the AI Observer independent of the existing application reverse-proxy and Cloudflare Tunnel configuration.

The Proxmox MCP endpoint remains accessible only on `127.0.0.1:8080` within observer-prod-01.

The OpenAI tunnel client forwards authenticated requests to this local endpoint.

**Implemented connection:**

- Tunnel name: `homelab-proxmox`
    
- Tunnel client: OpenAI `tunnel-client`
    
- MCP endpoint: `http://127.0.0.1:8080/mcp`
    
- OpenAI control-plane connection: Outbound HTTPS
    
- ChatGPT plugin: `Homelab Proxmox`
    

**Verification:** The tunnel service passed its readiness checks, and ChatGPT successfully invoked `proxmox_get_cluster_status` through the integration.

This OpenAI tunnel is distinct from the Cloudflare Tunnel used to expose existing homelab applications.

---

### AI Observer — Docker Compose for MCP, systemd for Tunnel Client

**Decision:** Run Proxmox MCP as a Docker Compose container while running the OpenAI tunnel client as a native systemd service.

**What was considered:**

- Managing all AI Observer components as Docker containers.
    
- Deploying the tunnel client separately as a host-level service (chosen).
    

**Why:** Proxmox MCP is a containerized application with a straightforward Compose deployment and an existing configuration workflow in `homelab-infra`.

The OpenAI tunnel client has a dedicated native Linux binary and systemd deployment model. Running it separately provides explicit service-account isolation, protected host-level credentials, independent restart behavior, and straightforward journal-based diagnostics.

The two components can be maintained and updated independently without requiring access to the Docker socket from the tunnel service.

**Implemented locations:**

- Proxmox MCP Compose: `/opt/stacks/proxmox-mcp/compose.yaml`
    
- Proxmox MCP environment: `/opt/stacks/proxmox-mcp/.env`
    
- Tunnel binary: `/opt/tunnel-client/tunnel-client`
    
- Tunnel configuration: `/etc/tunnel-client/config.yaml`
    
- Tunnel systemd unit: `/etc/systemd/system/tunnel-client.service`
    

The tunnel daemon runs as the dedicated unprivileged `tunnel-client` account, without Docker group membership.

**Operational choice:** Manage the observer through SSH and CLI rather than extending the existing Dockman deployment to this host.

---

### AI Observer — Dedicated Credentials and Git Exclusions

**Decision:** Keep all three MCP stacks and their OpenAI tunnel credentials outside version-controlled configuration, using protected local files and Bitwarden for credential retention.

**What was considered:**

- Embedding tokens directly into Compose, systemd, or tunnel configuration.
    
- Persisting GitHub credentials through plaintext Git credential storage.
    
- Separating credentials from committed configuration and restricting their filesystem permissions (chosen).
    

**Why:** The observer handles credentials that can access infrastructure monitoring APIs and OpenAI tunnel services. These credentials should not be copied into Git history, exposed in command arguments unnecessarily, or made readable by unrelated services.

The November 1 three-integration state used protected `.env` files for Proxmox, Unraid and Komodo, with three separate tunnel runtime credentials. Phase 4 (November 5) added a protected root-only UniFi `.env` and a fourth restricted OpenAI runtime key. Proxmox and Unraid tunnel clients have protected MCP bearer-header files; Komodo and UniFi intentionally have none. Do not assume the November 1 or November 5 files are included in the historical October 30 observer backup.

The `homelab-infra` repository contains the MCP Compose configuration, while secrets remain excluded from Git.

The initial Git workflow avoided persistently storing a GitHub personal access token through Git's plaintext credential helper.

**Recovery consideration:** The configuration under `/etc/tunnel-client`, the MCP `.env`, and the tunnel systemd service are not automatically covered merely because the Compose repository exists in Git.

Backup and credential-recovery procedures must explicitly account for these files.

A verified observer backup and restoration procedure remains outstanding.

---

### AI Observer — Three Independent Read-Only Integrations and Komodo Information Boundary

**Decision (verified November 1, 2026):** Extend the existing AI Observer VM with separate Unraid and Komodo MCP services and independent outbound OpenAI tunnel clients. Keep the operational model **read-only** and do not install another Docker socket on `observer-prod-01`.

**Implemented:** Three MCP containers (`proxmox-mcp`, `unraid-mcp`, `komodo-mcp`) publish only to localhost ports **8080, 8082, 8084**; three native systemd tunnel services (`tunnel-client.service`, `tunnel-client-unraid.service`, `tunnel-client-komodo.service`) use readiness ports **8081, 8083, 8085**. Each tunnel has a dedicated restricted OpenAI runtime key, while all share one `tunnel-client` binary and unprivileged service account.

**Security decisions:** Proxmox v0.5.1 enables nine audited toolsets (84 read tools), disables write/delete/exec and uses a `PVEAuditor` account. Unraid uses a separate `READ_ANY` API key, a 26-tool MCP allowlist and local bearer authentication. Komodo uses a dedicated non-admin Read/Logs identity, local MCP `MCP_AUTH_ENABLED=false` (read-only anonymous HTTP scope on localhost), best-effort secret scrubbing and eight exclusions: container Inspect, exec, image Inspect, volume Inspect, image history, full stack info, and both TOML exports. The operator **chose to retain** `komodo_server_info`, `komodo_procedure_info`, and `komodo_builder_info` for useful diagnostics while acknowledging they can return detailed configuration.

**Why:** Retain broad live monitoring and logs without exposing arbitrary commands, full container environment secrets, broad configuration exports or write operations. Read tool labels and secret scrubbing do **not** guarantee the returned material is nonsensitive. The deliberate lack of local Komodo bearer authentication means any eligible observer-local process can invoke exposed reads; the loopback binding, VM boundary, least-privilege upstream account and tool exclusions remain necessary safeguards.

**Verification:** All three plugins discovered exactly **84 + 26 + 40 = 150 Read tools and zero Write tools**. The Komodo tunnel `homelab-komodo` (`tunnel_6ae6e984758881919f5c308207bab3d3`) was active/enabled and HTTP-ready; `komodo_health_check` and `komodo_container_list` succeeded from ChatGPT (26 containers). Proxmox `0.5.1` was confirmed from the running container rather than inferred from newer upstream documentation. Full tool-by-tool testing, observer reboot, and restored backup validation are still outstanding.

**Operational references:** `services/infra/ai-observer.md`, `runbooks/ai-observer-operations.md`.

---

### AI Observer — Expand Integrations Incrementally

**Decision:** Implement and verify Proxmox MCP first, then evaluate additional read-only integrations separately.

**Planned integrations:**

- Unraid monitoring and diagnostics.
    
- Docker host and container monitoring.
    
- Potential future controlled write operations, subject to additional authorization and security review.
    

**Why:** Each integration introduces different credentials, permissions, network dependencies, and potential failure modes.

Completing one end-to-end integration before introducing additional systems allows the architecture, security boundaries, operational workflow, and recovery procedures to be validated incrementally.

**Current state (November 1, 2026):** All three read-only integrations are implemented and verified end-to-end: Proxmox MCP `0.5.1` (**84 Read tools** across nine enabled toolsets), Unraid Rust MCP `0.6.3` (**26 Read tools** via an explicit allowlist), and Komodo MCP `1.5.0` (**40 Read tools**, eight explicit exclusions) connected to Komodo Core `2.3.4` on `docker-prod-01`. Each has its own outbound OpenAI Secure MCP Tunnel and private ChatGPT plugin. Phase 3 subsequently extended **that same Komodo Core** to `immich-prod-01` and `auth-prod-01` using remote Periphery agents, without new MCP tunnels. No write tools are exposed through ChatGPT; automated writes remain out of scope. The original incremental decision and implementation sequence remain valid historical context.

**Related documentation:** `services/infra/ai-observer.md` and `runbooks/ai-observer-operations.md`

---

### AI Observer — Extend Existing Komodo Core to Three Docker Hosts Using Outbound Periphery

**Decision (implemented and verified November 1, 2026 CST):** Retain the single Komodo Core installation on `docker-prod-01` and single read-only Komodo MCP/OpenAI tunnel on `observer-prod-01`; enroll `immich-prod-01` and `auth-prod-01` as additional Komodo servers using dedicated **outbound-only Periphery v2.3.4** agents. Avoid publishing Periphery TCP 8120 on either remote Docker host. Allow terminals for authorized Komodo administrators, while preserving the independent, non-admin, **40 Read / zero Write** ChatGPT MCP boundary.

**Implemented:** Each remote host runs its own Compose deployment at `/opt/stacks/komodo-periphery/compose.yaml`, with permanent private identity keys under `/opt/appdata/komodo/periphery/keys/`. Agents connect using verified TLS to `wss://komodo.giohosted.com` with `PERIPHERY_CONNECT_AS` set to the exact host name, and `PERIPHERY_SERVER_ENABLED=false`. Initial host enrollment used a separate non-privileged onboarding key, which was removed from host configuration/storage and revoked in Komodo after successful persistent-key reconnection. Existing `immich` and `authentik` Compose projects were registered by **Files on Server**, matching the live Docker project names and paths; no new application stack deployment was needed.

**Verification:** Live `komodo_server_list` showed `docker-prod-01`, `immich-prod-01` and `auth-prod-01` all `Ok` at Periphery `2.3.4`; `komodo_stack_list` showed both application stacks running; `komodo_container_list` confirmed local workloads visible on the new hosts. Live log reads were possible through the unchanged read-only plugin. The Immich application server briefly appeared `created`/stack unhealthy during rollout and was resolved separately; final live read-only checks confirmed it running. A specific incident root cause is not established here.

**Rationale and security:** Outbound Periphery avoids additional inbound service ports, preserves existing Traefik/TLS routing, and reuses the centrally managed Komodo API. Periphery has highly privileged local Docker-socket access; this **does not** imply ChatGPT gets terminal, inspect, exec, or write privileges. The permanent keys require secure backups and access controls. A post-November 1 observer backup, backup-integrity/credential verification and full observer reboot/reconnect test remain pending. An isolated restore exercise was explicitly excluded from Phase 3 scope and was not performed.

**Related documentation:** `services/infra/ai-observer.md`, `runbooks/ai-observer-operations.md`, `services/media/immich.md`, `services/infra/authentik.md`.

---

### AI Observer Phase 4 — Dedicated Read-Only UniFi MCP with Cloud API Access

**Decision (implemented and end-to-end verified November 5, 2026 CST):** Add `ry-ops/unifi-mcp-server` as a **fourth independent MCP container and outbound OpenAI Secure MCP Tunnel** on `observer-prod-01` to inspect UDM-SE / UniFi Network. Use UniFi's Site Manager and Network cloud connector over `https://api.ui.com` instead of adding broad management-VLAN reachability. Preserve the original Proxmox, Unraid and Komodo MCP deployments unchanged.

**Implementation:** Upstream ry-ops project version `2.1.0`, pinned to commit `e13d0e22dce1b6a498edc5982d78bd589073d44c` (Python MCP SDK 1.30.0); Docker image `homelab/unifi-mcp:ryops-e13d0e2`, stack `/opt/stacks/unifi-mcp/`, root-only API key in `.env`, separate fail-closed `http_server.py` launcher using native Streamable HTTP. Docker publishes **only `127.0.0.1:8086`**, with server binding `0.0.0.0:8000` **inside** the container for Docker networking. Fourth tunnel `homelab-unifi` (`tunnel_6aed5ddd030c8191bf32196220fd079b`) uses `tunnel-client-unifi.service`, protected config/runtime key in `/etc/tunnel-client-unifi/`, and readiness at `127.0.0.1:8087/readyz`.

**Security decision:** `UNIFI_READ_ONLY=true` is required; the launcher refuses to start otherwise. Actual MCP registration and live ChatGPT discovery each showed **55 Read / zero Write** tools, yielding **205 Read / zero Write tools** across all four integrations. The UniFi Site Manager key was created with a **one-year expiration (operator-confirmed)** and Site Manager/Network application access intended for the UDM-SE, stored in Bitwarden. **Critical distinction:** UniFi API-key access is **not independently demonstrated read-only**, so the API key may be write-capable even when MCP exposes only read tools; additionally, the loopback-only UniFi MCP has no local per-caller bearer authentication. Keep the VM, credentials, host port binding and tool list restricted; don't equate read-only tool labels with read-only upstream credentials. No UniFi management, reboot, firewall edit, or other write actions are authorized through ChatGPT.

**Verification:** Site Manager `/v1/hosts` and Network connector `/v1/info` returned HTTP 200; UDM-SE `UDMPROSE` was connected, UniFi OS `5.1.35`, Network `10.6.109`. After correcting the internal HTTP binding, the tunnel-client diagnostic returned `RESULT ok`, systemd was `active`, `/readyz` returned `ready` (HTTP 200), and the operator enabled boot startup. The new `Homelab UniFi` plugin discovered 55 tools; live ChatGPT `unifi_health` and `get_site_status` returned a connected console and **3 devices online, 0 offline, 30 clients (13 wired / 17 wireless)** at `2026-11-05 23:16 CST`. Counts are point-in-time. All 55 operations were not individually tested.

**Out-of-scope verification:** The operator elected **not** to perform VM reboot/reconnect or isolated restore/recovery testing for Phase 4. Current function is verified; post-reboot restoration/backup completeness remain unverified and must not be represented as completed. Exact UniFi key expiration timestamp and persisted application/site grants were not independently read back.

**Related documentation:** `services/infra/ai-observer.md`, `runbooks/ai-observer-operations.md`.

---
