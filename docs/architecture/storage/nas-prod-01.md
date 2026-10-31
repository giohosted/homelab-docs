# nas-prod-01 — Storage Architecture

**Host:** nas-prod-01  
**OS:** Unraid 7.3.2 (verified by live MCP October 30, 2026)  
**Chassis:** Rosewill RSV-L4412U (4U, 12-bay)  
**Last Updated:** 2026-10-30

---

## Network Interfaces

| Interface | Type | IP | VLAN | Purpose |
|---|---|---|---|---|
| eth0 (onboard 2.5GbE) | br0 bridge | 192.168.10.10/24 | 10 — Management | Unraid web UI only |
| eth1 (X710 Port 2, 10GbE) | Direct | 192.168.30.16/24 | 30 — Services | NFS exports, Plex traffic, verified Unraid GraphQL HTTPS access from AI Observer |
| eth2 (X710 Port 1, 10GbE) | Unconfigured | 10.0.0.x/30 (Phase 3) | None | Point-to-point DAC to pve-prod-01 |

---

## Array — Parity Array (XFS)

| Role | Drive | Size |
|---|---|---|
| Parity 1 | Seagate IronWolf Pro (ST14000NT001) | 14TB — **DISABLED** (`DISK_DSBL`, red X) |
| Parity 2 | WD Red Pro (WD121KFBX) | 12TB — `DISK_OK` |
| Disk 1 | Seagate IronWolf (ST6000VN0033) | 6TB — `DISK_OK` |
| Disk 2 | Seagate IronWolf (ST6000VN0033) | 6TB — `DISK_OK` |

- **Usable:** ~12TB
- **Filesystem:** XFS per disk (Unraid default)
- **Historical parity sync completed:** 2026-03-11, 19h 28m, 0 errors. This predates the parity-drive change and is **not** proof of current parity health.
- **Planned expansion (historical, not implemented):** 3rd WD Red Pro 12TB data drive to be added after TrueNAS migration completes. Unraid Main shows `Disk 3 — Not installed` as of October 30, 2026.

---

### Disabled Parity 1 — Unresolved (October 30, 2026)

A read-only `unraid_array` query through the newly implemented Homelab Unraid MCP, followed by direct verification in Unraid **Main → Array Devices**, confirmed that the first parity slot contains a 14 TB Seagate `ST14000NT001` and has a red X / API status `DISK_DSBL` (disabled). The array reports `STARTED` and last parity check status `COMPLETED`, but this does **not** mean parity 1 is healthy. Its displayed error counter is zero; that does not rule out previous failed writes or connection problems.

- **Other array devices:** `parity2` 12 TB WD Red Pro, `disk1`, and `disk2` reported `DISK_OK`. This should not be interpreted as verified dual-parity protection while parity 1 is disabled.
- **History:** Original parity 1 was a 12 TB WD Red Pro. The operator previously replaced it with the 14 TB Seagate after encountering a similar Unraid disabled-disk problem. The replacement operated for a period before the same symptom recurred. The earlier parity-1 entry in this document was outdated.
- **Suspected cause:** SAS-to-SATA breakout cable or another shared connection/power/backplane/controller path. **Not diagnosed or confirmed**; disk hardware failure has not been excluded.
- **Unassigned Devices:** A 12 TB WD Red Pro appears in the Unassigned Devices area. Its role as the former parity disk is consistent with the replacement history, but its physical identity and health have not been independently verified.
- **Evidence:** The operator downloaded Unraid diagnostics before any further action; archive retained locally by the operator, not added to documentation.
- **Actions performed:** **None** toward parity repair. No cable swap, disk reassignment, rebuild, parity sync, NAS shutdown, or other storage change was performed during MCP integration.
- **Next action:** Address in a **separate controlled troubleshooting conversation**. Review saved diagnostics, inspect connections and SMART data, verify actual drive health, and follow an appropriate Unraid recovery procedure only after the cause is understood. Do not reboot, reassign, or initiate parity rebuilding casually; coordinate clean NAS shutdown with PBS/Immich/NFS dependencies before any physical cabling work.

The ZFS mirror is a separate storage pool; its last independently verified `ONLINE` state is recorded below. The disabled parity issue does not by itself establish a ZFS pool failure. The current state of parity protection should be rechecked before any consequential array operation.

---

## Pool — ZFS Mirror (zfs-mirror)

| Role | Drive | Size |
|---|---|---|
| Mirror disk 1 | WD Red Plus (WD40EFRX) | 4TB |
| Mirror disk 2 | WD Red Plus (WD40EFPX) | 4TB |

- **Usable:** ~3.68TB
- **Filesystem:** ZFS mirror
- **Snapshots:** ZnapZend — daily kept 7 days, weekly kept 30 days on photos and backups datasets (historically documented; retention not reverified during the PBS incident)
- **Health checked 2026-10-30:** `zpool status -v` reported pool and both mirror members `ONLINE`, zero READ/WRITE/CKSUM errors, and no known data errors.

---

## Shares

| Share | Pool | Path | Purpose |
|---|---|---|---|
| data | Parity Array | /mnt/user/data | Single share for all media and downloads — required for hardlinks. Contains `media/` and `downloads/` as subfolders. |
| appdata | Parity Array | /mnt/user/appdata | Docker/app config (NAS perspective) |
| isos | Parity Array | /mnt/user/isos | Proxmox ISO images |
| photos | ZFS Mirror | /mnt/user/photos | Immich library |
| backups | ZFS Mirror | /mnt/user/backups | PBS backups, Docker stacks, Docker appdata |
| system | ZFS Mirror | /mnt/user/system | Unraid system share (location confirmed; contents not reverified) |

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

### PBS NFS Filesystem and October 2026 Incident

**Verified storage topology (October 30):**

```text
PBS pbs-prod-01:/mnt/backups/pbs
  -> NFSv3 mount 192.168.30.16:/mnt/user/backups at /mnt/backups
  -> Unraid /mnt/user/backups (fuse.shfs)
  -> ZFS dataset zfs-mirror/backups at /mnt/zfs-mirror/backups
  -> /mnt/zfs-mirror/backups/pbs (existing PBS datastore)
```

The `backups` share is configured with `shareUseCache="only"`, `shareCachePool="zfs-mirror"`, and no secondary pool. The `photos` and `system` shares also live solely under `/mnt/zfs-mirror`, based on read-only directory checks. The PBS datastore directory is owned by UID/GID `34:34` and includes `.chunks`, `.lock`, `ct`, and `vm`.

**Current mount/export observations (not changes):**

- Unraid export: `/mnt/user/backups`, `fsid=101`, `async`, `no_subtree_check`, and `no_root_squash`; permitted networks include 192.168.30.0/24, 192.168.10.0/24, and 10.0.0.0/30.
- PBS client mount: NFSv3 with `soft,timeo=30,retrans=3` and systemd automount, configured in `/etc/fstab` on PBS.
- Unraid FUSE mount: `fuse.shfs` at `/mnt/user` with `fuse_remember="330"` (also visible as `remember=330` in the `shfs` process arguments).
- **Settings → Global Share Settings → Permit exclusive shares = No.** This setting remains unchanged.

**Incident evidence:** The long-running PBS proxy held a stale descriptor to `/mnt/backups/pbs/.lock`, and backup operations failed with `ESTALE` and client connection resets. The underlying ZFS lock file's timestamps dated to March 18 and the Unraid `shfs` process had run since July 28. The original invalidation event was not proven; do not record FUSE as a conclusively established root cause.

**Recovery on October 30:** Restarting only `proxmox-backup-proxy` cleared the immediate stale descriptor. The next manual CT 100 backup and both 02:00 scheduled PVE backup jobs succeeded; CT 100 passed integrity verification. **No NAS configuration, export, filesystem, ownership, or global exclusive-share change was made.**

**Operator decision:** Keep existing share behavior and monitor Discord alerts. A PBS-specific direct-ZFS NFS export may be investigated later as a limited-scope alternative, but remains **unimplemented** and needs a reviewed maintenance/cutover plan. Do not enable exclusive shares globally to address PBS without assessing impact on `photos`, `system`, and other services. Do not manually remove `.lock` or backup datastore data.

See `runbooks/backup-schedule.md` and `runbooks/backup-restore.md` for the incident chronology and recovery procedure.

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

## AI Observer — Unraid Monitoring Integration

`observer-prod-01` at `192.168.30.17` runs the [dinglebear-ai Rust Unraid MCP](https://github.com/dinglebear-ai/unraid/tree/main/unraid-rs), which queries `nas-prod-01` over verified HTTPS using its Services VLAN address `192.168.30.16:443` and the GraphQL endpoint `https://nas-prod-01.local/graphql`. The self-signed certificate is trusted explicitly, and the API key has 14 `READ_ANY` permission families with no write grants.

The Homelab Unraid ChatGPT plugin connects through its own outbound OpenAI Secure MCP Tunnel. The MCP publishes 26 read-only atomic tools and zero write tools. On October 30, 2026, authenticated calls retrieved Unraid version 7.3.2 and array state; a direct ChatGPT `unraid_info` invocation subsequently succeeded. Tool discovery was verified, but not every one of the 26 tool operations has been individually tested.

**Full deployment configuration, exposed tool list, unexposed capabilities, credentials model, and extension policy:** `services/infra/ai-observer.md`.  
**Operational procedures:** `runbooks/ai-observer-operations.md`.

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

| Plugin              | Purpose                     |
| ------------------- | --------------------------- |
| NUT                 | UPS graceful shutdown       |
| ZFS Master          | ZFS pool management UI      |
| ZnapZend            | ZFS snapshot scheduling     |
| Intel-GPU-TOP       | iGPU monitoring             |
| Dynamix System Temp | CPU/board temp monitoring   |
| Unassigned Devices  | External drive management   |
| Fix Common Problems | Configuration health checks |