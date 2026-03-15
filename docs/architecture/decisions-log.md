# Architecture Decisions Log

**Last Updated:** 2026-03-15  

---

## Hardware

### NAS Motherboard — ASUS TUF Gaming Z690-Plus WiFi D4

**Decision:** Replace original Gigabyte B760M DS3H DDR4 with ASUS TUF Gaming Z690-Plus WiFi D4.

**What was considered:**

- Gigabyte B760M DS3H DDR4 (original plan — already owned)
- MSI PRO B660M-A DDR4 (found in IT closet, spare)
- ASUS TUF Gaming Z690-Plus WiFi D4 (purchased used ~$100 on Reddit r/hardwareswap)

**Why:** The Gigabyte board has only 1x usable PCIe slot (one x16 physical slot, two x1 slots). The NAS needs to run both the LSI SAS 9120-8i HBA (x8 card, requires x8 or x16 slot) and the 10GbE SFP+ NIC (requires x4 minimum) simultaneously — physically impossible on the Gigabyte board. The MSI board was ruled out due to known Realtek NIC driver issues in Linux. The ASUS TUF Z690 has a second x16 physical slot wired at x4 from the chipset, which fits both cards with no conflict. ATX form factor, solid VRM, Intel 2.5GbE onboard, and BIOS already updated for 13th gen.

---

### NAS 10GbE NIC — Dell Intel X710-DA2

**Decision:** Dell Intel X710-DA2 dual-port SFP+ NIC (~$25 eBay).

**What was considered:**

- Intel X520 (older, very common in homelabs)
- Dell Intel X710-DA2 (newer chipset, dual port)

**Why:** X710 has a newer chipset than the X520 with better long-term driver support in Linux. Dual port provides future flexibility at no meaningful cost difference at used prices. Only one port is in use (DAC to MS-A2) but the second is available if needed later.

---

### NAS CPU — i5-13400 over i5-13600

**Decision:** Keep i5-13400 for the NAS, sell the i5-13600.

**Why:** Unraid is IO-bound, not compute-bound. The performance delta between the two chips is irrelevant for a NAS workload. Selling the i5-13600 partially offsets the MS-A2 cost. The i5-13400 also has Intel UHD 730 iGPU for Plex QuickSync hardware transcoding — which is the only compute-heavy task the NAS does.

---

### NAS RAM — 32GB, sold extra sticks

**Decision:** Keep 32GB DDR4 for the NAS, sell the extra 2x 16GB sticks.

**Why:** No heavy VM workloads run on Unraid — it's a pure NAS with one Plex Docker container. 32GB is generous for this role. Selling the extra sticks recoups cost.

---

### NAS Chassis — Rosewill RSV-L4412U

**Decision:** Rosewill RSV-L4412U 4U 12-bay chassis (~$345).

**Why:** 12 drive bays accommodates current drive inventory (5x 12TB + 5x 4TB + 2x 6TB) with room to expand. 4U rackmount fits the open-frame rack plan. Includes sliding rails which is important for accessing rear cabling in a rack.

---

### Primary Compute — Minisforum MS-A2

**Decision:** Minisforum MS-A2 (Ryzen 9 7945HX, 64GB DDR5) as primary Proxmox node.

**Why:** 16-core/32-thread CPU with strong single-thread performance handles multiple concurrent VMs and LXCs without breaking a sweat. Dual 10GbE SFP+ built-in eliminates need for a separate NIC for storage link. Compact form factor fits on a 2U shelf in the rack. 64GB DDR5 is sufficient for all planned workloads with headroom; max is 96GB.

---

### MS-A2 Boot — ZFS RAID-1 Mirror (Samsung + Sabrent NVMe)

**Decision:** Two NVMe drives in ZFS RAID-1 mirror configured in the Proxmox installer.

**Why:** Single NVMe boot was the primary fragility in v2 — one drive failure killed the entire hypervisor. Mirrored boot means a single drive can fail and Proxmox keeps running. Selected Proxmox installer ZFS RAID-1 option at install time; no additional configuration needed.

---

### Secondary Compute — Dell Optiplex 3070 Micro (existing hardware)

**Decision:** Use existing Optiplex 3070 Micro as pve-prod-02 rather than building a second full Proxmox host from spare parts.

**What was considered:**

- Building a second full node using spare i5-13600, spare mobo, and extra 32GB RAM
- Using the Optiplex as-is (free from work)

**Why:** The Optiplex's workload is intentionally light — PBS VM and one AdGuard LXC. HA is explicitly not enabled in this design (Optiplex cannot absorb MS-A2's workload anyway), so a matched second node provides no real benefit. Building a second powerful node means higher idle power draw and cost for no meaningful gain. Selling the i5-13600 and extra RAM recoups ~$250-300. The Optiplex is free and perfectly adequate for its role.

---

### Optiplex RAM — Left at 16GB for now

**Decision:** Leave Optiplex at 16GB DDR4 (2x 8GB Micron, from work). Do not upgrade yet.

**Why:** Current workload (PBS VM + AdGuard LXC) does not justify the upgrade. Revisit if workload grows.

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

---

### No HA (High Availability)

**Decision:** Proxmox cluster without HA enabled.

**Why:** HA requires matched hardware and a 3+ node quorum to be meaningful. The Optiplex cannot absorb the MS-A2's workload. Enabling HA without these conditions is false security — it creates the illusion of resilience without delivering it. Clustering is enabled for unified management UI only. If pve-prod-01 goes down, services on it are down until it comes back — that is acceptable for a homelab.

---

### QDevice on Pi

**Decision:** Run Proxmox QDevice on pi-prod-01 as cluster tiebreaker.

**Why:** A 2-node Proxmox cluster without a quorum device can suffer split-brain — both nodes think they're the primary and fence each other. QDevice on the Pi costs nothing (10-minute setup) and prevents this. The Pi is lightweight, always-on, and perfectly suited for this role.

---

### DNS — AdGuard Home (carry forward)

**Decision:** Keep AdGuard Home as the DNS solution. Primary LXC on pve-prod-01 (dns-prod-01), secondary LXC on pve-prod-02 (dns-prod-02), synced via adguardhome-sync.

**Why:** Works well in v2, ad-blocking is a bonus, no reason to migrate. Two instances on different Proxmox nodes means DNS survives either node going down. Secondary AdGuard moved off the Pi (v2 arrangement) — Pi is now dedicated to QDevice and monitoring only.

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

### PBS for VM/LXC Snapshots

**Decision:** Proxmox Backup Server (PBS) on pve-prod-02 as a VM (pbs-prod-01), backing up all VMs on both nodes to the NAS ZFS mirror pool.

**Why:** PBS is the right tool for VM-level backups in a Proxmox environment. Running it on pve-prod-02 means the backup server is on a different node than the primary workloads — a backup server on the same node it's backing up is a single point of failure.

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

**Why:** QDevice is infrastructure — it belongs on the management network with the Proxmox nodes it serves as a tiebreaker. VLAN 10 has outbound reach to all VLANs, so Beszel and Uptime Kuma can probe services on VLAN 30 without any extra firewall rules. Return traffic is handled by the stateful firewall (allow established connections). Placing the Pi on VLAN 30 would require punching a hole from VLAN 30 → VLAN 10 for QDevice, which is exactly the kind of management VLAN exposure the design aims to avoid.

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