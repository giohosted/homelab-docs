# pve-prod-02 — Secondary Proxmox Node

**Role:** Secondary compute node hosting `immich-prod-01` (VM 105)  
**Status:** Active — replacement hardware deployed November 13, 2026  
**Last Updated:** 2026-11-13

---

## Hardware

| Component | Detail |
|-----------|--------|
| **Model** | Dell OptiPlex 5000 SFF (replaced OptiPlex 3070 Micro on 2026-11-13) |
| **CPU** | Intel Core i7-12700 (12 cores: 8P + 4E / 20 threads) |
| **RAM** | 48 GB DDR4 (2×16 GB + 2×8 GB); Proxmox reports ~46.7 GiB usable |
| **Boot Drive 1** | SK hynix BC711 NVMe 512 GB (`/dev/nvme0n1`) |
| **Boot Drive 2** | Kioxia KBG50ZNS512G NVMe 512 GB (`/dev/nvme1n1`, PCIe adapter) |
| **Boot Storage** | ZFS RAID-1 mirror (`rpool`); `rpool/data` backs VM and CT disks |
| **Networking** | Built-in 1 GbE RJ45, connected to USW-Pro-Max-24 |
| **BIOS** | Dell OptiPlex 5000 BIOS 1.43.0 (operator-confirmed upgrade November 13) |

**Disk health verified November 13, 2026:** Both NVMe devices reported SMART `PASSED`, 0 critical warnings, and 0 media/data integrity errors. SK hynix reported 0% endurance used at 44°C; Kioxia reported 16% used at 33°C. These are point-in-time readings, not guarantees of remaining life.

---

## Software

| Field | Value |
|-------|-------|
| **OS** | Proxmox VE 9.2.26 (`pve-manager`), kernel `7.0.14-28-pve` |
| **Debian base** | Trixie (Debian 13) |
| **Boot mode** | UEFI, Secure Boot disabled |
| **Time sync** | Chrony, `America/Chicago`; synchronized after 2026-11-13 correction and reboot |

---

## Network

| Field | Value |
|-------|-------|
| **Hostname (FQDN)** | `pve-prod-02.local` |
| **IP** | `192.168.10.12/24` |
| **VLAN** | 10 — Management, untagged on `vmbr0` |
| **Gateway** | `192.168.10.1` |
| **DNS** | `192.168.10.1`; search domain `local` (live-verified) |
| **Switch** | USW-Pro-Max-24; operator reports `Trunk All` port profile |
| **Link** | Built-in 1 GbE interface |

### Network Interfaces

| Interface | Type | Role |
|-----------|------|------|
| `nic0` | Physical Ethernet | Bridge port for host management and guest traffic |
| `vmbr0` | Linux bridge | `192.168.10.12/24`; VLAN-aware, VLAN IDs `2-4094`; guest VLANs tagged |

The bridge is verified active. Immich uses VLAN tag `30` on `net0` and retains IP `192.168.30.14`; application/UI and NFS access were tested after migration.

---

## Proxmox Configuration

### Apt Repositories

- Debian Trixie, Trixie updates, and security repositories enabled.
- Proxmox `pve-no-subscription` repository enabled in `/etc/apt/sources.list.d/proxmox.sources`.
- `pve-enterprise.sources` and `ceph.sources` retained but disabled (`Enabled: no`).
- November 13 upgrade: 213 packages upgraded, no removals in the simulation; updated Proxmox/kernel verified after reboot.
- No subscription key. **Subscription-popup modifications are not documented as applied to the replacement host.**

### Cluster

- Member of `homelab-v3`; node ID `2`; Corosync address `192.168.10.12`.
- Rejoined on November 13 as a fresh installation using `pvecm add 192.168.10.11` after removal of the retired hardware's cluster membership and stale configuration.
- Three active voting members: `pve-prod-01`, `pve-prod-02`, and `pve-prod-03`; no QDevice.
- HA is deliberately disabled. A healthy cluster quorum does not imply automatic VM failover.

### Storage

| Storage | Type | Notes |
|---------|------|-------|
| `local` | Directory | Proxmox templates, ISOs, and other configured directory content |
| `local-zfs` | ZFS | Local `rpool/data`; VM/LXC disks; storage definition scoped to all three nodes, but **not shared** |
| `nas-isos-p2` | NFS | `192.168.30.16:/mnt/user/isos`, shared ISO library; assigned to `pve-prod-02` **and** `pve-prod-03` |
| `pbs-prod-01` | PBS | Cluster-wide backup storage; Docker PBS at `192.168.30.16:8007`, datastore `nas-backups` |

The separate `nas-isos` definition, assigned only to `pve-prod-01`, accesses the **same** NAS ISO share over that node's direct DAC storage link (`10.0.0.2`). Do not assign `nas-isos` to the other nodes unless they gain reachability to the DAC subnet.

**ZFS maintenance:** `/etc/cron.d/zfsutils-linux` schedules TRIM at 00:24 on the first Sunday of each month and scrub at 00:24 on the second Sunday. `rpool` autotrim is `off`; scheduled TRIM is the chosen mechanism. On November 13 the mirror was `ONLINE` with zero read/write/checksum errors.

**Redundant boot:** `proxmox-boot-tool status` showed two configured UEFI boot partitions (`60B0-9005` and `60B0-E015`), both containing kernels `7.0.14-28-pve` and `7.0.2-6-pve`. **Independent physical boot from each SSD has not been tested.**

### PBS Backup and Retired VM 103

**Active PBS:** Unraid Docker `proxmox-backup-server` at `192.168.30.16:8007`, datastore `nas-backups`; cluster storage ID `pbs-prod-01` preserved. PBS datastore is bound directly from `/mnt/zfs-mirror/backups/pbs` on Unraid. The cluster backup job `backup-54015f90-26e5` runs at 02:00, selects all existing guests, and still contains a harmless exclusion for VM 103.

**VM 103 was retired and deleted before replacement of the old `pve-prod-02`.** It is not present on the new host and is **not** an available rollback VM. As of November 5, the former PBS VM was still stopped and retained on the OptiPlex 3070 Micro; that historical arrangement has since been superseded. The former NFSv3/Unraid-FUSE PBS stale-handle incident and migration are documented in `runbooks/backup-schedule.md`.

**November 13 verification:** Manual snapshot backup of VM 105 from the replacement `pve-prod-02` to PBS completed `TASK OK` at 17:16 CST. The scheduled 02:00 job had succeeded earlier that day on `pve-prod-01` and `pve-prod-03`; the **first scheduled 02:00 run on the replacement `pve-prod-02` is not yet separately verified**. Full-guest restore testing remains open.

See `runbooks/backup-schedule.md`, `runbooks/backup-restore.md`, and `architecture/storage/nas-prod-01.md`.

---

## Guests

| ID | Name | Type | VLAN | IP | Status |
|----|------|------|------|----|--------|
| 105 | `immich-prod-01` | Ubuntu 24.04 VM | 30 | `192.168.30.14` | Running; 2 vCPU, 6 GiB RAM, 32 GB ZFS disk |

`dns-prod-02` (CT 102) now resides on `pve-prod-03` at `192.168.30.15`. PBS runs on Unraid rather than as a PVE guest.

**Immich migration:** VM 105 was temporarily hosted on `pve-prod-03` during the hardware replacement. On November 13 it was shut down and migrated offline to the replacement Dell using `qm migrate 105 pve-prod-02 --with-local-disks --targetstorage local-zfs`. Guest configuration, VLAN tag, virtual NIC MAC, and UEFI disk were retained. Post-migration checks confirmed containers running, UI access, and the NAS photos NFS mount at `/data/photos`.

---

## Design Decisions

### Mirrored NVMe Boot and Local VM Storage

The replacement uses two 512 GB NVMe drives in a ZFS mirror rather than the old OptiPlex's single NVMe/LVM-thin installation. The mirror protects against one member disk failing, subject to backups and the untested independent-boot scenario; it is not a substitute for PBS.

### Stronger Compute for Immich

The i7-12700 and 48 GB RAM provide additional CPU and memory headroom for Immich, especially its CPU-based machine-learning tasks. VM 105 remains at **2 vCPU and 6 GiB** until separately evaluated; a stronger host does not automatically change its guest allocations.

### HA Not Enabled

Cluster membership is for unified management and operational flexibility. Automatic HA is not enabled; recovery/failover needs a separate capacity-aware design and tested procedures.

---

## Installation Notes

### Replacement — November 13, 2026

1. Retired the old OptiPlex 3070 Micro after moving CT 102 and VM 105 away and deleting retired VM 103; old node had zero remaining guests.
2. Updated Dell OptiPlex 5000 BIOS to 1.43.0; installed Proxmox VE 9.2 with ZFS RAID-1 on both NVMe drives.
3. Kept the new Dell off the production network while the old host still held `192.168.10.12`; shut down the old host, removed it with `pvecm delnode pve-prod-02`, and cleared its empty stale `/etc/pve/nodes/pve-prod-02` directory.
4. Removed obsolete `local-lvm` storage definition, rejoined the new host, extended `local-zfs` to all three nodes, and assigned `nas-isos-p2` to nodes 02 and 03.
5. Enabled VLAN-aware `vmbr0`; applied package updates and rebooted into kernel `7.0.14-28-pve`.
6. Corrected a roughly six-hour clock error: Chrony initially had unresolved NTP sources, causing web/API `401 Authentication failed`. `chronyc refresh` populated sources and the clock synchronized; verified stable after reboot.
7. Migrated VM 105 back from node 03, verified Immich and NAS storage, completed a fresh PBS backup, and confirmed the existing Uptime Kuma host monitor reports UP.

### Historical Hardware — Superseded

The original `pve-prod-02` was a Dell OptiPlex 3070 Micro (i5-9500T, 16 GB RAM, single 256 GB NVMe, Proxmox 9.1.6 with LVM-thin). During its first installation, setting SATA from Intel RAID/RST to AHCI resolved installer disk detection; its `nas-isos` mount required a distinct NFS definition because the primary node's `10.0.0.2` DAC address was unreachable. Those instructions and the former QDevice arrangement **do not describe the current replacement host**.
