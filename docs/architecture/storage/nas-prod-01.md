# nas-prod-01 — Storage Architecture

**Host:** nas-prod-01  
**OS:** Unraid 7.3.2 (verified by live MCP October 30, 2026)  
**Chassis:** Rosewill RSV-L4412U (4U, 12-bay)  
**Last Updated:** 2026-11-05

---

## Network Interfaces

| Interface | Type | IP | VLAN | Purpose |
|---|---|---|---|---|
| eth0 (onboard 2.5GbE) | br0 bridge | 192.168.10.10/24 | 10 — Management | Unraid web UI only |
| eth1 (X710 Port 2, 10GbE) | Direct | 192.168.30.16/24 | 30 — Services | NFS exports, Plex traffic, PBS Docker TCP 8007, verified Unraid GraphQL HTTPS access from AI Observer |
| eth2 (X710 Port 1, 10GbE) | Unconfigured | 10.0.0.x/30 (Phase 3) | None | Point-to-point DAC to pve-prod-01 |

---

## Array — Parity Array (XFS)

**Current status (verified October 31, 2026):** Array `STARTED`; dual parity valid after completed reconstruction. All five array devices report `DISK_OK` with 0 Unraid disk errors via the read-only Homelab Unraid MCP. This incident is **resolved**; the replaced SAS-to-SATA breakout cable is the probable, not conclusively proven, root cause.

| Role | Drive | Size | Serial | Verified status |
|---|---|---|---|---|
| Parity 1 | WD Red Pro (WD121KFBX-68EF5N0) | 12TB | `AAHKXP9H` | `DISK_OK` |
| Parity 2 | WD Red Pro (WD121KFBX-68EF5N0) | 12TB | `2AH2JSWY` | `DISK_OK` |
| Disk 1 | Seagate IronWolf (ST6000VN0033) | 6TB | `ZAD97T6M` | `DISK_OK` |
| Disk 2 | Seagate IronWolf (ST6000VN0033) | 6TB | `ZAD97RT0` | `DISK_OK` |
| Disk 3 | WD Red Pro (WD121KFBX-68EF5N0) | 12TB | `5PK7HDXF` | `DISK_OK` |

- **Configuration:** 2 parity + 3 XFS data disks. The third data disk is **installed and in service**, not a planned expansion.
- **Usable data capacity:** approximately 24 TB decimal (about 22 TiB), before filesystem overhead; parity disks do not contribute to share capacity.
- **Filesystem:** XFS on each array data disk; parity devices do not contain mounted data filesystems.
- **Historical parity sync:** March 11, 2026, 19h 28m, 0 errors; this describes the earlier configuration, not the current recovery.
- **Current recovery:** Simultaneous Parity 1 and Disk 3 reconstruction completed October 31, 2026, in approximately 20h 18m (73,119 seconds), with 0 reported errors. After completion, Unraid displayed valid parity and green status for both disks. MCP subsequently reported `DISK_OK` for all five array drives.

---

### Parity 1 and Disk 3 Connection Incident — Resolved (October 31, 2026)

**Incident history:** Parity 1 was disabled (`DISK_DSBL`, red X) while occupied by a 14 TB Seagate IronWolf Pro (`ST14000NT001`). The earlier 12 TB WD Red Pro in that slot had previously exhibited similar problems and had been swapped out, but the Seagate later developed the same symptom. Disk 3, a separate 12 TB WD Red Pro, also disappeared from the active array. The operator confirmed that **Parity 1 and Disk 3 shared one SAS-to-SATA breakout cable**. The exact original fault mechanism was not captured; the shared breakout cable or connector path is the leading explanation, with HBA, backplane, connector, and power-path alternatives not individually excluded.

**Recovery performed:**

1. Shut down `nas-prod-01` and physically replaced the suspected SAS-to-SATA breakout cable.
2. Removed the 14 TB Seagate from Parity 1 and reinstalled the original 12 TB WD Red Pro, serial `AAHKXP9H`.
3. Reassigned the existing 12 TB WD Red Pro, serial `5PK7HDXF`, to Disk 3. The user confirmed this was an established data-disk role, not a newly commissioned empty disk.
4. Started the array and accepted Unraid's recovery operation for both assignments. The syslog recorded `md: recovery thread: recon P D3 ...`, confirming simultaneous reconstruction of Parity 1 and Disk 3.
5. Reconstruction completed with 0 reported errors, restoring valid parity and `DISK_OK` status for both slots.

**Post-recovery verification (October 31):**

| Check | Result |
|---|---|
| Unraid array / parity | Array `STARTED`; valid parity; Parity 1, Parity 2, Disk 1, Disk 2 and Disk 3 all `DISK_OK`, 0 reported disk errors |
| Disk 3 mount | `/dev/md3p1` mounted as XFS at `/mnt/disk3` read/write; approximately 2.2 TB used; `appdata`, `data`, and `isos` directories visible |
| Directory traversal | `find /mnt/disk3 -xdev -print > /dev/null` returned exit code `0` |
| Parity 1 SMART | Overall-health `PASSED`; fresh short self-test completed without error; 0 reallocated, pending, or offline-uncorrectable sectors |
| Disk 3 SMART | Overall-health `PASSED`; fresh short self-test completed without error; 0 reallocated, pending, or offline-uncorrectable sectors |
| Parity 1 SATA CRC history | Attribute 199 = **454**, unchanged in post-recovery check |
| Disk 3 SATA CRC history | Attribute 199 = **29**, unchanged in post-recovery check |
| New connection errors | No matching new SATA reset, `ICRC`, or block-I/O communication errors observed in the reviewed post-replacement syslog |

**Filesystem context:** Before the reassignment, a direct Unassigned Devices mount of physical Disk 3 (`/dev/sdf1`) failed at 23:31 CDT on October 30 with XFS metadata corruption messages. Unraid's emulated `/dev/md3p1` filesystem had mounted cleanly. After Disk 3 reconstruction, `/dev/md3p1` again mounted normally and its directory tree traversed without reported errors. The earlier physical-mount warning is historical evidence, **not an active unresolved array incident**.

**Root cause assessment:** **Probable shared breakout-cable/connection fault.** Both affected drives had historical interface CRC errors, and the substituted Parity 1 Seagate had developed the same disabling symptom. Both drives completed a sustained reconstruction and short SMART testing after the cable replacement with no new CRC counts or disk errors. A cable-specific electrical test was not performed, so this is a well-supported inference rather than definitive component isolation.

**Verification boundaries:** No full XFS metadata check, full-file-content verification, extended SMART surface test, or separate non-correcting parity check was performed. These are optional additional assurance measures and do **not** represent an ongoing known failure. Continue ordinary SMART, parity, and error-counter monitoring as routine NAS operations.

**Historical documentation discrepancy:** The earlier storage document described `Disk 3 — Not installed` on October 30, whereas the operator confirmed Disk 3 had been an established third data disk and the active XFS/emulated contents were consistent with that. This record now reflects the serial-number-verified five-disk array. The original Phase 2 commissioning record separately describes the early build with two data disks; the exact date Disk 3 first entered service was not established here.

**ZFS impact:** The independent `zfs-mirror` pool was not part of this parity reconstruction. Its existing `zpool status -v` check and subsequent Unraid pool-device status were healthy; no ZFS reconfiguration was required by this repair.

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
├── pbs/                    ← PBS Docker datastore, direct path /mnt/zfs-mirror/backups/pbs (owned by uid 34 — do not touch)
├── appdata/                ← rsync appdata backups (owned by root)
│   ├── docker-prod-01/
│   ├── auth-prod-01/
│   └── immich-prod-01/
└── stacks/                 ← rsync stacks backups (owned by root)
    ├── docker-prod-01/
    ├── auth-prod-01/
    └── immich-prod-01/
```

> **Warning:** The `pbs/` subdirectory must remain owned by UID/GID `34:34` (PBS `backup` user); the root is mode `755` and `.chunks` is mode `750` after registration of the existing datastore in PBS 4.2.0. Do not place unrelated or root-owned files in the datastore. Docker PBS uses the **direct** `/mnt/zfs-mirror/backups/pbs` path, not the `/mnt/user` FUSE alias. The datastore is scoped to the `pbs/` subdirectory rather than the share root.

### NFS Exports

- **Exported shares:** `data`, `appdata`, `isos`, `photos`, `backups`
- **Security:** Private — rules for 192.168.30.0/24 and 10.0.0.0/30 with `no_root_squash`
- **Protocol:** NFSv3 (Unraid Max Server Protocol set to NFSv3)
- **Permissions:** `nobody:users`, chmod `a=,a+rX,u+w,g+w`
- **SMB:** Disabled on all shares (NFS only)

> **Note:** The `data` share is mounted on docker-prod-01 as a single NFSv3 mount at `/data`. This is required for hardlinks and atomic moves to work correctly across media and downloads. See `architecture/decisions-log.md` for rationale.

### PBS Docker — Active Deployment (verified November 5, 2026)

**Current architecture:** The Unraid Community Apps `proxmox-backup-server` container (`ayufan/proxmox-backup-server:latest`, unofficial image; PBS UI 4.2.0) runs on `nas-prod-01`. It uses Docker Host networking and is accessed by PVE through `192.168.30.16:8007` (Services VLAN). Because Host networking uses host interfaces, TCP 8007 may also be reachable through management IP `192.168.10.10`; the desired firewall exposure has not been independently assessed.

**Datastore:** Existing `nas-backups` archives remain in `/mnt/zfs-mirror/backups/pbs` and are bind-mounted to `/backups` inside the PBS container. No separate child dataset was present beneath `zfs-mirror/backups` when checked. The parent dataset also stores separate `appdata/` and `stacks/` rsync backups; do not confuse its capacity usage with exclusive PBS usage. The directory ownership is `34:34`; its root mode was corrected from `700` to `755` for PBS reuse, while `.chunks` remains `750`. No recursive `chown`, datastore wipe, or migration-time ZFS snapshot was performed.

**Container configuration:** Privileged disabled, timezone `America/Chicago`, image tag `latest`, auto-start enabled; Extra Parameters: `--restart unless-stopped --memory=2g --mount type=tmpfs,destination=/run --stop-signal=SIGHUP`. Persistent paths:

| Host path | Container path | Purpose |
|---|---|---|
| `/mnt/user/appdata/pbs/etc` | `/etc/proxmox-backup` | PBS configuration and sensitive secrets |
| `/mnt/user/appdata/pbs/logs` | `/var/log/proxmox-backup` | PBS logs |
| `/mnt/user/appdata/pbs/lib` | `/var/lib/proxmox-backup` | PBS state |
| `/mnt/zfs-mirror/backups/pbs` | `/backups` | Existing PBS datastore, direct ZFS |

**Validated:** The old VM 103 is stopped. Existing snapshots were visible (7 groups / 91 snapshots at initial registration). A November 5 manual backup and targeted integrity verification of CT 100 passed, as did both nodes' 02:00 scheduled backup jobs; a file-level restore of `/etc/hostname` succeeded. No isolated full VM/CT restore, post-migration GC/prune/weekly verify execution, or storage mount-order boot test has been completed. The container was still reported RUNNING with auto-start enabled during follow-up live Unraid inspection.

**Authentication and alerts:** PVE uses dedicated PBS API token `pve-backup@pbs!pve-cluster` (token and parent least privilege on `nas-backups`). Authentik OIDC remains shared with Proxmox; client secret rotated and interactive logins tested across PBS and both PVE nodes. Local `admin@pbs` password was changed from the default. The Discord webhook test message was delivered; matcher configured for warning/error. Do not record tokens or client secrets in Markdown.

**Maintenance:** Daily pruning 3 last / 7 daily / 4 weekly / 3 monthly; GC Saturday 04:00; verification Saturday 05:00, skip verified if younger than 30 days. Operator's target is ~500 GB for PBS, as a **soft budget**, not a configured quota.

**Lifecycle risks:** The image is explicitly unofficial/unmaintained and may lag official Proxmox security updates; no new image update was advertised by Unraid on November 5. Update via Unraid Docker image recreation, preserving `/mnt/user/appdata/pbs` and datastore mounts, and monitor new releases. Avoid running the retired VM concurrently. Verify startup ordering after the ZFS pool mount and determine whether to restrict Host-network access to TCP 8007 on the management VLAN. The operator accepted the risk of foregoing a migration-time protective ZFS snapshot and is willing to recreate the datastore if unrecoverable.

### PBS NFS Filesystem and October 2026 Incident — Historical

**Historical only:** The NFSv3 → `fuse.shfs` path below belonged to the retired PBS VM. Docker PBS does **not** currently use that path for its datastore. The October 30 proxy restart was temporary; identical stale-handle errors recurred October 31–November 2 and prompted the November 2026 Docker deployment.

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
| proxmox-backup-server | Host networking, 192.168.30.16:8007 | PBS 4.2.0 image `ayufan/proxmox-backup-server:latest` — direct ZFS datastore |

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