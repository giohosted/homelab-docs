
# pve-prod-01 — Primary Proxmox Node

**Role:** Primary Proxmox node — runs all primary VMs and LXCs  
**Status:** Active  
**Last Updated:** 2026-11-13

---

## Hardware

|Component|Detail|
|---|---|
|**Model**|Minisforum MS-A2|
|**CPU**|AMD Ryzen 9 7945HX (16C/32T, 5.4GHz boost)|
|**iGPU**|AMD Radeon 680M (available for future ML/transcoding — not used at launch)|
|**RAM**|32GB DDR5 SO-DIMM (1x 32GB — 2nd slot empty, max 96GB)|
|**Boot Drive 1**|Samsung 980 NVMe 1TB — S/N: S64ANS0W120169T|
|**Boot Drive 2**|Sabrent Rocket NVMe 1TB|
|**Boot Config**|ZFS RAID-1 mirror (pool: rpool) — configured in Proxmox installer|
|**LAN NIC**|2x 2.5GbE RJ45 built-in (igc + r8169)|
|**Storage NIC**|2x 10GbE SFP+ built-in — Port 1: DAC to nas-prod-01|

---

## Software

|Field|Value|
|---|---|
|**OS**|Proxmox VE 9.1.6|
|**Kernel**|6.17.13-1-pve|
|**Debian base**|Trixie (Debian 13)|

---

## Network

| Field           | Value                                     |
| --------------- | ----------------------------------------- |
| **IP**          | 192.168.10.11                             |
| **VLAN**        | 10 — Management                           |
| **Gateway**     | 192.168.10.1                              |
| **DNS**         | 192.168.10.1                              |
| **Switch port** | USW-Pro-Max-24 ports 17–24 (2.5GbE range) |
| **Link speed**  | 2500Mb/s confirmed via ethtool            |

### Network Interfaces

|Interface|Type|Role|
|---|---|---|
|nic0 (igc)|2.5GbE RJ45|Management + VM/LXC traffic via vmbr0|
|nic1 (r8169)|2.5GbE RJ45|Unused — no cable connected|
|vmbr0|Linux bridge on nic0|VLAN-aware — management untagged, VMs/LXCs tagged per guest|
|SFP+ Port 1|10GbE|DAC to nas-prod-01 X710 Port 1 — dedicated storage link|
|SFP+ Port 2|10GbE|Unused at launch|

> **vmbr0 is VLAN-aware.** The Proxmox management interface (192.168.10.11) runs untagged on the bridge. VMs and LXCs use VLAN tags appropriate to their roles. Production services, including docker-prod-01, auth-prod-01, and observer-prod-01, use VLAN 30 (Services). The Dragonwilds dedicated server uses VLAN 50 (Game Servers).

---

## Proxmox Configuration

### Apt Repositories

- Enterprise repos removed: `pve-enterprise.sources`, `ceph.sources`
    
- No-subscription repo added: `pve-no-subscription` (trixie)
    
- Subscription nag popup removed from proxmoxlib.js
    

### Cluster

- Member of cluster `homelab-v3`
    
- Node ID: 0x00000001
    
- QDevice configured — pi-prod-01 (192.168.10.20) acts as tiebreaker
    

### Storage

|Storage|Type|Path / Pool|Notes|
|---|---|---|---|
|local|Directory|/var/lib/vz|ISO images, CT templates|
|local-zfs|ZFS|rpool/data|VM disks, LXC rootfs|
|nas-isos|NFS|10.0.0.2:/mnt/user/isos|ISO images — via DAC link. Scoped to pve-prod-01 only.|
|pbs-prod-01|PBS|nas-backups datastore|Docker PBS on nas-prod-01 at 192.168.30.16:8007; shared PVE storage ID preserved|

> local-lvm ghost entry was present after install and removed via `pvesm remove local-lvm` — pve-prod-01 uses local-zfs for VM disks.  
> local-zfs entry was lost at one point and re-added via: `pvesm add zfspool local-zfs --pool rpool/data --content rootdir,images`

### PBS Backup Migration and Validation — November 5, 2026

**Active:** PVE cluster-wide storage `pbs-prod-01` targets `nas-backups` on Docker PBS at `192.168.30.16:8007`, hosted by Unraid (`nas-prod-01`). The old PBS VM 103 at `192.168.30.12` on pve-prod-02 is shut down. Existing PVE storage ID and daily 02:00 cluster job `backup-54015f90-26e5` were preserved. Cluster storage now authenticates using dedicated `pve-backup@pbs!pve-cluster` token, whose permissions and backup-group ownership were migrated. No secret values are recorded here.

The previous PBS VM connected to the Unraid datastore through NFSv3 and `shfs` FUSE. A stale proxy `.lock` file descriptor caused failed backups. A targeted proxy restart on October 30 temporarily restored backups, but errors recurred October 31–November 2 on both Proxmox nodes. The Docker replacement accesses the same ZFS datastore directly through `/mnt/zfs-mirror/backups/pbs` → `/backups`, bypassing the former PBS datastore NFS/FUSE path.

**Verified on pve-prod-01:**

- Manual CT 100 (`dns-prod-01`) backup at 00:36 CST on November 5: `TASK OK`, 4 seconds, approximately 79.7% reuse. Its new snapshot passed targeted PBS integrity verification.
- 02:00 scheduled backup job on November 5 finished `OK` at 02:05:50 CST (347 seconds). Backup selection includes CT 100, VMs 101, 104, 106, and 107 (as configured; individual per-guest integrity not exhaustively verified).
- A file-level restore of CT 100's `/etc/hostname` succeeded via the PVE backup browser. A full VM/CT restore has not been performed.
- Shared Authentik OIDC secret was rotated; a fresh SSO login passed on this PVE node and on pve-prod-02 and PBS.

**To monitor:** Continued 02:00 backups, PBS scheduled prune/GC/verification, actual warning/error notification delivery if desired, and a later isolated full guest recovery test. Direct Docker Host networking may expose PBS TCP 8007 on both NAS interfaces; access restrictions were not independently validated. See `runbooks/backup-schedule.md`, `runbooks/backup-restore.md`, and `architecture/storage/nas-prod-01.md`.

---

## Guests

The original guest inventory was verified through the live Homelab Proxmox MCP integration on October 30, 2026. VM 101 and VM 104 resource allocations and running state were reverified on November 13, 2026. Other inventory entries retain their previous verification scope.

|ID|Name|Type|VLAN|IP|Status|
|---|---|---|---|---|---|
|100|dns-prod-01|LXC (Debian 12)|30|192.168.30.10|Running — 1 GiB RAM, 1 vCPU, 4 GiB disk|
|101|docker-prod-01|VM (Ubuntu 24.04)|30|192.168.30.11|Running — 7 GiB RAM, 4 vCPU, 64 GiB disk|
|104|auth-prod-01|VM (Ubuntu 24.04)|30|192.168.30.13|Running — 4 GiB RAM, 2 vCPU, 32 GiB disk|
|106|dragonwilds-server|VM (Linux)|50|Not verified|Stopped — 8 GiB RAM, 4 vCPU, 40 GiB disk|
|107|observer-prod-01|VM (Ubuntu 26.04.1)|30|192.168.30.17|Running — 2 GiB RAM, 2 vCPU, 20 GiB disk|

> **Inventory note:** Guest statuses represent the state observed during the applicable live verification, not permanent operating requirements. The Dragonwilds dedicated-server VM was stopped at the time of its last inventory verification. Its configured IP address was not verified.

> **auth-prod-01:** VM 104 previously had 5 GiB allocated, verified in November 2026. Its allocation was reduced to 4 GiB and verified after reboot on November 13, 2026.

> immich-prod-01 was originally planned for pve-prod-01 but moved to pve-prod-02 due to RAM headroom. See `architecture/decisions-log.md`.

### VM Memory Right-Sizing — November 13, 2026

A resource-sizing review was conducted after Proxmox displayed nearly full memory utilization for Linux VMs despite substantial guest filesystem cache.

Live Proxmox configuration and performance history were reviewed alongside guest-level `free -h`, swap, memory-pressure and Docker container statistics.

**Completed changes:**

|VM|Previous RAM|Current RAM|vCPU|Ballooning|
|---|---|---|---|---|
|docker-prod-01 (101)|8 GiB|7 GiB|4|Disabled|
|auth-prod-01 (104)|5 GiB|4 GiB|2|Disabled|
|observer-prod-01 (107)|2 GiB|2 GiB|2|Disabled|

The Immich VM on pve-prod-02 remains allocated 6 GiB RAM and 2 vCPUs. No CPU allocation changes were made to any VM.

Both modified VMs were rebooted, and the new Proxmox memory configurations were confirmed active with no pending memory changes.

**Post-resize guest validation:**

|Metric|docker-prod-01|auth-prod-01|
|---|---|---|
|RAM allocated|7 GiB|4 GiB|
|Guest RAM used|3.5 GiB|1.4 GiB|
|Guest RAM available|3.2 GiB|2.4 GiB|
|Swap occupied|268 KiB|268 KiB|
|Recent memory PSI averages|0.00|0.00|
|Active swap-in/out during sample|None|None|

Additional live verification through Komodo confirmed:

- docker-prod-01: 25 of 25 containers running.
- auth-prod-01: 6 of 6 containers running.
- Both Komodo server connections reported `Ok`.
- pve-prod-01 had approximately 9.97 GiB available host memory following the changes.

**Result:** A total of 2 GiB of VM memory allocation was reclaimed on pve-prod-01. Both VMs passed initial post-reboot resource and container-state checks, with no observed recent memory pressure or active swapping.

**Operational considerations:**

- Proxmox host-side memory consumption is not equivalent to guest application working-set usage when ballooning is disabled.
- Linux filesystem cache is reclaimable and should not be interpreted as permanently consumed application RAM.
- The earlier 6 GiB Docker configuration experienced swap usage during Phase 4; the current 7 GiB allocation is a conservative trial rather than an established minimum.
- The low swap occupancy immediately following reboot partly reflects the reset of prior swap state.
- Normal workload behavior was verified at the time of measurement; sustained peak-load, indexing and maintenance windows were not separately stress-tested.
- Reassess allocation only if ongoing memory pressure, active swap activity, OOM kills, service degradation or workload growth warrants it.

### AI Observer — observer-prod-01

VM 107 hosts the Homelab v3 AI Observer infrastructure, providing AI-assisted, read-only monitoring and diagnostics of Proxmox, Unraid, Komodo, and UniFi.

The observer is intentionally isolated in its own Ubuntu VM rather than sharing the primary Docker application host.

**Implementation status:** Four MCP integrations and four independent OpenAI Secure MCP Tunnels were deployed and verified end-to-end by November 5, 2026. The UniFi integration remained reachable and reported read-only operation during a live ChatGPT check on November 13, 2026.

#### VM Configuration

|Setting|Value|
|---|---|
|VMID|107|
|Host|pve-prod-01|
|OS|Ubuntu Server 26.04.1 LTS|
|vCPU|2|
|CPU type|x86-64-v3|
|RAM|2 GiB|
|Memory ballooning|Disabled|
|Disk|20 GiB on local-zfs|
|Firmware|OVMF / UEFI|
|Machine|q35|
|Network bridge|vmbr0|
|VLAN|30 — Services|
|IP|192.168.30.17|
|QEMU guest agent|Enabled|
|Start at boot|Enabled|
|Startup delay|90 seconds|

The VM's Proxmox configuration and running state were reverified on November 13, 2026.

A November 2026 memory audit found approximately 458 MiB used and 1.1 GiB available inside the guest, with no observed recent memory pressure or active swapping. Its 2 GiB allocation was deliberately retained to preserve headroom for simultaneous MCP queries, maintenance, and future integrations.

#### MCP Integration Inventory

|Integration|Server version|Read tools|Purpose|
|---|---|---|---|
|Proxmox|mattoddie/proxmox-mcp 0.5.1|84|Cluster, nodes, VMs, containers, storage, backups and diagnostics|
|Unraid|dinglebear-ai/unraid-rmcp 0.6.3|26|NAS array, disks, shares, Docker and system health|
|Komodo|MP-Tool/komodo-mcp-server 1.5.0|40|Read-only Docker monitoring across registered hosts|
|UniFi|ry-ops/unifi-mcp-server 2.1.0|55|UniFi Network, devices, clients, VLANs, firewall and WAN monitoring|
|**Total**|**4 MCP services**|**205**|**Read-only tool exposure**|

All four are exposed to ChatGPT through dedicated plugins:

- `Homelab Proxmox`
- `Homelab Unraid`
- `Homelab Komodo`
- `Homelab UniFi`

The documented tool inventory contains 205 read tools and zero exposed write tools.

The UniFi MCP version 2.1.0 is pinned to upstream commit `e13d0e22dce1b6a498edc5982d78bd589073d44c`.

#### Multi-Host Docker Monitoring

Komodo Core runs on `docker-prod-01` and monitors three registered Docker hosts:

|Docker host|Role|Komodo connection|
|---|---|---|
|docker-prod-01|Primary Docker application host|Local Periphery|
|immich-prod-01|Immich application host|Remote Periphery over outbound WSS|
|auth-prod-01|Authentik application host|Remote Periphery over outbound WSS|

All three Komodo hosts returned `Ok` during live verification on November 13, 2026, using Periphery version 2.3.4.

The observer's Komodo MCP connects to the existing Komodo Core HTTPS endpoint. No separate MCP server or OpenAI tunnel is needed for each monitored Docker host.

#### MCP Services and OpenAI Tunnel Architecture

Each integration runs as an independent Docker MCP service on `observer-prod-01`, with a corresponding native systemd-managed outbound OpenAI Secure MCP Tunnel.

|Integration|MCP listener|Tunnel readiness|systemd service|
|---|---|---|---|
|Proxmox|127.0.0.1:8080|127.0.0.1:8081|tunnel-client.service|
|Unraid|127.0.0.1:8082|127.0.0.1:8083|tunnel-client-unraid.service|
|Komodo|127.0.0.1:8084|127.0.0.1:8085|tunnel-client-komodo.service|
|UniFi|127.0.0.1:8086|127.0.0.1:8087|tunnel-client-unifi.service|

MCP HTTP endpoint paths use `/mcp`, and the tunnel readiness endpoints use `/readyz`.

The four tunnel services share one installed tunnel-client binary while maintaining independent tunnel identities, runtime credentials, configuration directories and service units.

**Architecture characteristics:**

- All MCP HTTP listeners and tunnel readiness ports are bound to the observer's loopback interface.
- No public MCP hostname, Traefik route, inbound internet port forwarding, or external firewall exposure is required.
- Tunnel connections are initiated outbound by the observer.
- The dedicated `tunnel-client` Linux account has restricted privileges and no Docker group membership.
- Tunnel credentials, upstream API secrets and MCP environment files are protected outside the Git repository.
- The OpenAI Secure MCP Tunnels are independent of the homelab's application-oriented Cloudflare Tunnel infrastructure.

#### Security Boundaries

The observer implements several independent security controls:

**Proxmox**

- Uses a dedicated `observer@pve!proxmox-mcp` API identity with `PVEAuditor` permissions.
- Write, delete and command-execution tool categories are disabled.
- Access to the Proxmox management API on TCP 8006 is permitted through a targeted firewall exception rather than general VLAN 30 access.
- The MCP container does not mount the Proxmox host's Docker socket.

**Unraid**

- Uses a dedicated API identity restricted to the intended monitoring categories.
- Exposes an explicit allowlist of 26 read tools.
- Uses verified TLS for the upstream GraphQL API.
- Does not mount the NAS filesystem or Unraid Docker socket into the observer.

**Komodo**

- Uses the dedicated `ai-observer-komodo` account with scoped Read and Logs permissions.
- Exposes 40 read tools, with eight sensitive tools deliberately excluded.
- Does not grant ChatGPT administrator access to Komodo.
- Local MCP authentication is disabled, but access is restricted to the observer's loopback interface and upstream Komodo credentials remain required.

**UniFi**

- Connects through UniFi Site Manager and Network API to the UDM-SE.
- Uses `UNIFI_READ_ONLY=true` and a fail-closed launcher restricting the exposed MCP tools to 55 read operations.
- Does not expose UniFi configuration modification operations to ChatGPT.
- The upstream UniFi API credential may have broader permissions than the read-only MCP tool surface; these are distinct security boundaries.

All four MCP integrations are read-only from the ChatGPT tool interface. This does not mean every upstream service credential independently has read-only permissions, nor does it mean all returned information is nonsensitive.

The observer does not expose direct write operations through the four installed Homelab integrations.

#### Deployment and Recovery References

The four MCP Compose deployments reside under:

- `/opt/stacks/proxmox-mcp/`
- `/opt/stacks/unraid-mcp/`
- `/opt/stacks/komodo-mcp/`
- `/opt/stacks/unifi-mcp/`

The private `homelab-infra` repository includes their deployment definitions and sanitized templates for the four tunnel configurations and systemd units.

The active tunnel configurations, protected credentials and runtime state remain outside Git.

**Verification history:**

- October 29–30: Proxmox MCP deployed and verified.
- October 30: Unraid MCP deployed and verified.
- November 1: Komodo MCP deployed; multi-host Docker monitoring verified.
- November 5: UniFi MCP and fourth tunnel deployed and verified.
- November 6: Infrastructure-as-code coverage for all four deployments and tunnel templates verified.
- November 13: Observer VM configuration and UniFi MCP connectivity rechecked through live integrations.

The existence of deployment definitions does not establish that a full observer disaster recovery has been tested. A complete isolated restore and recovery validation remains outstanding in the operational documentation.

**Full service documentation:** `services/infra/ai-observer.md`

**Operational runbook:** `runbooks/ai-observer-operations.md`

The service documentation and runbook are authoritative for container settings, security controls, credentials, service management, troubleshooting and recovery procedures. This node document maintains only the compute and architecture summary.

---

## Installation Notes

### Boot Drive Configuration

ZFS RAID-1 mirror selected during Proxmox installer. Both NVMe drives appear in rpool — single drive failure does not kill the hypervisor. This was the primary fragility in v2 (single NVMe boot) and is fixed in v3.

### Issues Encountered

- PVE 9 uses `.sources` apt format instead of `.list` — enterprise repo cleanup required targeting `pve-enterprise.sources` and `ceph.sources`
    
- MS-A2 initially connected to a 1GbE switch port (ports 1–16) — moved to 2.5GbE port (ports 17–24), confirmed 2500Mb/s
    
- local-lvm ghost entry present after install — removed with `pvesm remove local-lvm`
    
- local-zfs storage entry lost at one point — re-added with `pvesm add zfspool`
    

---

## NIC Selection Notes

The MS-A2 has two 2.5GbE RJ45 ports:

- **igc** (Intel I226-V) — primary port, selected for management interface. Better Linux driver support, more reliable at 2.5GbE.
    
- **r8169** (Realtek) — secondary port, no cable connected.
    

During Proxmox install, igc was selected as the management NIC. Only one cable runs from the MS-A2 to the switch — single VLAN-aware bridge (vmbr0) handles all traffic.
