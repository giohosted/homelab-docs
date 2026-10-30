# pve-prod-01 — Primary Proxmox Node

**Role:** Primary Proxmox node — runs all primary VMs and LXCs  
**Status:** Active  
**Last Updated:** 2026-10-30

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

|Field|Value|
|---|---|
|**IP**|192.168.10.11|
|**VLAN**|10 — Management|
|**Gateway**|192.168.10.1|
|**DNS**|9.9.9.9 (public — VLAN 10 does not use AdGuard)|
|**Switch port**|USW-Pro-Max-24 ports 17–24 (2.5GbE range)|
|**Link speed**|2500Mb/s confirmed via ethtool|

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
|pbs-prod-01|PBS|nas-backups datastore|Proxmox Backup Server at 192.168.30.12|

> local-lvm ghost entry was present after install and removed via `pvesm remove local-lvm` — pve-prod-01 uses local-zfs for VM disks.  
> local-zfs entry was lost at one point and re-added via: `pvesm add zfspool local-zfs --pool rpool/data --content rootdir,images`

### PBS Backup Status — October 30, 2026

The daily 02:00 Proxmox job `backup-54015f90-26e5` targets `pbs-prod-01` (PBS datastore `nas-backups`) and includes CT 100 and VMs 101, 104, 106, and 107 on this node. PBS VM 103 is excluded at the cluster level.

Following repeated failures through October 29, the PBS proxy on VM 103 was found holding a stale NFS datastore `.lock` descriptor. The operator restarted only `proxmox-backup-proxy` at 00:55 CDT on October 30. The October 30 scheduled pve-prod-01 job ran **02:00–02:22 CDT** and finished `TASK OK`, with success messages for all five guests (including Dragonwilds VM 106 and AI Observer VM 107).

The manual CT 100 backup at 00:57 CDT also succeeded, and its PBS integrity verification at 01:02 CDT reported zero errors. Independent integrity verification of all other new snapshots and an isolated restore test remain outstanding. No NAS/NFS/FUSE configuration changes were implemented; recurrence prevention remains open. See `runbooks/backup-schedule.md` and `runbooks/backup-restore.md`.

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

Its initial implementation provides read-only Proxmox monitoring and diagnostics through MCP, accessible to ChatGPT using OpenAI Secure MCP Tunnel.

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

- A dedicated Docker-hosted Proxmox MCP server.
    
- A native systemd-managed OpenAI Secure MCP Tunnel client.
    
- A dedicated Proxmox API identity with read-only permissions.
    
- A targeted firewall exception permitting access to the Proxmox API on TCP 8006.
    
- A localhost-only MCP HTTP listener.
    
- Protected credentials stored outside the Git repository.
    

The Proxmox MCP server and OpenAI tunnel were verified through successful live tool invocation from ChatGPT.

The observer does not currently have write permissions to Proxmox, and its planned Docker and Unraid integrations have not yet been implemented.

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