# pve-prod-01 — Primary Proxmox Node

**Role:** Primary Proxmox node — runs all primary VMs and LXCs  
**Status:** Active  
**Last Updated:** 2026-11-05

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

The following guest inventory was verified through the live Homelab Proxmox MCP integration on 2026-10-30 UTC.

|ID|Name|Type|VLAN|IP|Status|
|---|---|---|---|---|---|
|100|dns-prod-01|LXC (Debian 12)|30|192.168.30.10|Running — 1 GiB RAM, 1 vCPU, 4 GiB disk|
|101|docker-prod-01|VM (Ubuntu 24.04)|30|192.168.30.11|Running — 8 GiB RAM, 4 vCPU, 64 GiB disk|
|104|auth-prod-01|VM (Ubuntu 24.04)|30|192.168.30.13|Running — 5 GiB RAM, 2 vCPU, 32 GiB disk|
|106|dragonwilds-server|VM (Linux)|50|Not verified|Stopped — 8 GiB RAM, 4 vCPU, 40 GiB disk|
|107|observer-prod-01|VM (Ubuntu 26.04.1)|30|192.168.30.17|Running — 2 GiB RAM, 2 vCPU, 20 GiB disk|

> **Inventory note:** Guest statuses represent the state observed during the live verification, not permanent operating requirements. The Dragonwilds dedicated-server VM was stopped at the time of inspection. Its configured IP address was not verified.

> **auth-prod-01:** The VM currently has 5 GiB of allocated RAM, verified through its Proxmox configuration. Earlier documentation listed 4 GB.

> immich-prod-01 was originally planned for pve-prod-01 but moved to pve-prod-02 due to RAM headroom. See `architecture/decisions-log.md`.

### AI Observer — observer-prod-01

VM 107 hosts the Homelab v3 AI Observer infrastructure.

Its initial read-only Proxmox MCP implementation has since been expanded to **three separately verified integrations**: Proxmox, Unraid and Komodo (Docker on `docker-prod-01`), each accessed from ChatGPT through its own OpenAI Secure MCP Tunnel.

**Verified VM configuration:**

|Setting|Value|
|---|---|
|VMID|107|
|Host|pve-prod-01|
|OS|Ubuntu Server 26.04.1 LTS|
|vCPU|2|
|CPU type|x86-64-v3|
|RAM|2 GiB|
|Disk|20 GiB on local-zfs|
|Firmware|OVMF / UEFI|
|Machine|q35|
|Network bridge|vmbr0|
|VLAN|30 — Services|
|IP|192.168.30.17|
|QEMU guest agent|Enabled|
|Start at boot|Enabled|
|Startup delay|90 seconds|

The AI Observer includes:

- Three Docker-hosted read-only MCP servers on the observer: Proxmox (`0.5.1`, 84 read tools), Unraid (`0.6.3`, 26 read tools), and Komodo (`1.5.0`, 40 read tools).
    
- Three independent native systemd-managed OpenAI Secure MCP Tunnel clients, sharing one installed binary.
    
- A dedicated Proxmox API identity with read-only permissions.
    
- A targeted firewall exception permitting access to the Proxmox API on TCP 8006.
    
- Three localhost-only MCP listeners (TCP 8080, 8082, 8084) and three localhost tunnel readiness listeners (TCP 8081, 8083, 8085).
    
- Protected credentials stored outside the Git repository.
    

All three MCP services and their independent OpenAI tunnels were verified through successful live tool invocation from ChatGPT by November 1, 2026. The Komodo upstream account is read-only with scoped Logs access; its local MCP is intentionally unauthenticated but loopback-bound, and eight sensitive tools are excluded.

The observer does not currently expose write operations through Proxmox, Unraid or Komodo. A full November 1 backup and isolated restore test have **not** been verified; see the AI Observer operations runbook.

**Full service documentation:** `services/infra/ai-observer.md`

**Operational runbook:** `runbooks/ai-observer-operations.md`

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