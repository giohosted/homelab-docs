# nas-prod-01 — Storage Architecture

**Host:** nas-prod-01  
**OS:** Unraid 7.2.4  
**Chassis:** Rosewill RSV-L4412U (4U, 12-bay)  
**Last Updated:** 2026-03-18

---

## Network Interfaces

| Interface | Type | IP | VLAN | Purpose |
|---|---|---|---|---|
| eth0 (onboard 2.5GbE) | br0 bridge | 192.168.10.10/24 | 10 — Management | Unraid web UI only |
| eth1 (X710 Port 2, 10GbE) | Direct | 192.168.30.16/24 | 30 — Services | NFS exports, Plex traffic |
| eth2 (X710 Port 1, 10GbE) | Unconfigured | 10.0.0.x/30 (Phase 3) | None | Point-to-point DAC to pve-prod-01 |

---

## Array — Parity Array (XFS)

| Role | Drive | Size |
|---|---|---|
| Parity 1 | WD Red Pro (WD121KFBX) | 12TB |
| Parity 2 | WD Red Pro (WD121KFBX) | 12TB |
| Disk 1 | Seagate IronWolf (ST6000VN0033) | 6TB |
| Disk 2 | Seagate IronWolf (ST6000VN0033) | 6TB |

- **Usable:** ~12TB
- **Filesystem:** XFS per disk (Unraid default)
- **Parity sync completed:** 2026-03-11, 19h 28m, 0 errors
- **Planned expansion:** 3rd WD Red Pro 12TB data drive to be added after TrueNAS migration completes

---

## Pool — ZFS Mirror (zfs-mirror)

| Role | Drive | Size |
|---|---|---|
| Mirror disk 1 | WD Red Plus (WD40EFRX) | 4TB |
| Mirror disk 2 | WD Red Plus (WD40EFPX) | 4TB |

- **Usable:** ~3.68TB
- **Filesystem:** ZFS mirror
- **Snapshots:** ZnapZend — daily kept 7 days, weekly kept 30 days on photos and backups datasets

---

## Shares

| Share | Pool | Path | Purpose |
|---|---|---|---|
| data | Parity Array | /mnt/user/data | Single share for all media and downloads — required for hardlinks. Contains `media/` and `downloads/` as subfolders. |
| appdata | Parity Array | /mnt/user/appdata | Docker/app config (NAS perspective) |
| isos | Parity Array | /mnt/user/isos | Proxmox ISO images |
| photos | ZFS Mirror | /mnt/user/photos | Immich library |
| backups | ZFS Mirror | /mnt/user/backups | PBS backups, Docker stacks, Docker appdata |

### data share folder structure
```
data/
├── media/
│   ├── movies/
│   │   ├── 1080p/
│   │   └── 4k/
│   ├── tv/
│   ├── anime/
│   └── books/
│       ├── ebooks/
│       └── audiobooks/
└── downloads/
    ├── movies/
    ├── tv/
    ├── anime/
    └── books/
        ├── ebooks/
        │   ├── downloads/
        │   └── ingest/
        └── audiobooks/
            └── downloads/
```

### backups share folder structure
```
/mnt/user/backups/
├── pbs/                    ← PBS datastore (owned by uid 34 — do not touch)
├── appdata/                ← rsync appdata backups (owned by root)
│   ├── docker-prod-01/
│   ├── auth-prod-01/
│   └── immich-prod-01/
└── stacks/                 ← rsync stacks backups (owned by root)
    ├── docker-prod-01/
    ├── auth-prod-01/
    └── immich-prod-01/
```

> **Warning:** The `pbs/` subdirectory must remain owned by uid 34 (PBS `backup` user). Do not place root-owned files inside it. PBS GC scans the entire datastore path and will abort with a permission error if it encounters files it cannot read. This is why PBS is scoped to `/mnt/user/backups/pbs` rather than the share root.

### NFS Exports

- **Exported shares:** `data`, `appdata`, `isos`, `photos`, `backups`
- **Security:** Private — rules for 192.168.30.0/24 and 10.0.0.0/30 with `no_root_squash`
- **Protocol:** NFSv3 (Unraid Max Server Protocol set to NFSv3)
- **Permissions:** `nobody:users`, chmod `a=,a+rX,u+w,g+w`
- **SMB:** Disabled on all shares (NFS only)

> **Note:** The `data` share is mounted on docker-prod-01 as a single NFSv3 mount at `/data`. This is required for hardlinks and atomic moves to work correctly across media and downloads. See `architecture/decisions-log.md` for rationale.

---

## SSH Access

SSH is enabled on nas-prod-01. All SSH sessions authenticate as `root`.

> **Note:** Unraid's user management (Users tab in the UI) is SMB-only. SSH always authenticates as root regardless of what users exist in the Unraid UI.

### Authorized Keys

`/root/.ssh/authorized_keys` contains dedicated backup keys for each Docker host. These keys are used exclusively by the nightly rsync backup scripts — they are not interactive login keys.

| Key | Host | Purpose |
|-----|------|---------|
| `backup@docker-prod-01` | docker-prod-01 | Nightly rsync backup script |
| `backup@auth-prod-01` | auth-prod-01 | Nightly rsync backup script |
| `backup@immich-prod-01` | immich-prod-01 | Nightly rsync backup script |

Key location on each host: `/root/.ssh/backup_rsa`  
Target IP used by rsync scripts: `192.168.30.16` (VLAN 30 data interface — not the management IP)

> **Note:** SSH to nas-prod-01 must use `192.168.30.16` from VLAN 30 hosts. The management IP `192.168.10.10` is blocked by firewall Rule 6 (Services → Management DENY).

---

## Docker Containers

| Container | IP | Purpose |
|---|---|---|
| plex | 192.168.30.2 | Plex Media Server — bound to eth1 (VLAN 30) |

- **iGPU passthrough:** /dev/dri → Intel Raptor Lake-S UHD Graphics (QuickSync)
- **Hardware transcoding:** enabled — HEVC encoding set to "HEVC sources only"
- **Media path:** /data → /mnt/user/data (data share)
- **Appdata:** /mnt/user/appdata/plex

---

## UPS & Graceful Shutdown

- **UPS:** Tripp-Lite SMART1500LCDXL connected via USB-B cable to nas-prod-01
- **NUT plugin:** dmacias/Network UPS Tools, NUT 2.8.4
- **NUT mode:** Netserver — nas-prod-01 is the NUT server, Proxmox nodes connect as clients
- **Driver:** usbhid-ups (auto-detected), port: auto
- **UPS confirmed:** On Line, 100% battery, 34 min runtime at 15% load (225 VA), 1500VA nominal
- **Shutdown timer:** 6 minutes on battery before initiating shutdown sequence
- **Shutdown order:** Proxmox nodes receive NUT signal and shut down VMs/LXCs then hypervisor → NAS shuts down last
- **NUT clients:** pve-prod-01, pve-prod-02 — configured in Phase 3

---

## Plugins Installed

| Plugin | Purpose |
|---|---|
| NUT | UPS graceful shutdown |
| ZFS Master | ZFS pool management UI |
| ZnapZend | ZFS snapshot scheduling |
| Intel-GPU-TOP | iGPU monitoring |
| Dynamix System Temp | CPU/board temp monitoring |
| Unassigned Devices | External drive management |
| Fix Common Problems | Configuration health checks |