# nas-prod-01 — Storage Architecture

**Host:** nas-prod-01  
**OS:** Unraid 7.3.4 (live-verified November 14, 2026)  
**Chassis:** Rosewill RSV-L4412U (4U, 12-bay)  
**Role:** Primary NAS, bulk media storage, ZFS-backed critical data, Plex, Proxmox Backup Server, and UPS/NUT server  
**Last Updated:** 2026-11-14

**Verification status:** Network interfaces, Unraid version, array disk status, Docker container state, shares, Proxmox NFS storage access, and PBS storage connectivity were checked through read-only integrations on November 14. Historical incident investigations and implementation details are retained with their original verification dates. Direct ZFS pool-state readback, current NUT server configuration, and PBS maintenance-job execution have not been independently verified in this review.

---

## Network Interfaces

The NAS uses separate management, service/data, and dedicated point-to-point storage interfaces.

| Interface | Type | IP | Network | Purpose |
|---|---|---|---|---|
| `br0` (onboard 2.5GbE) | Linux bridge | `192.168.10.10/24` | Management — UniFi default network | Unraid administration, NUT server, management-plane services |
| `eth1` (X710 Port 2, 10GbE) | Physical interface | `192.168.30.16/24` | Services — VLAN 30 | NFS, PBS, and service-facing data traffic |
| `eth2` (X710 Port 1, 10GbE) | Physical interface | `10.0.0.2/30` | Private point-to-point, untagged | Dedicated DAC to `pve-prod-01` |

### Live Network Verification — November 14, 2026

The Homelab Unraid integration returned:

| Interface | IPv4 Address | Link State | Speed | MTU |
|---|---|---|---|---|
| `br0` | `192.168.10.10` | UP | 2.5 Gbps | 1500 |
| `eth1` | `192.168.30.16` | UP | 10 Gbps | 1500 |
| `eth2` | `10.0.0.2` | UP | 10 Gbps | 1500 |

All three addresses were configured statically according to the returned interface information.

**Resolved documentation discrepancy:**

Earlier versions of this document described `eth2` as unconfigured, with a future `10.0.0.x/30` address.

That statement is superseded.

**`eth2` is implemented and operational at `10.0.0.2/30`, with an active 10 Gbps link.**

The corresponding Proxmox storage configuration confirms that `pve-prod-01` accesses the NAS ISO export through `10.0.0.2`.

### Dedicated Storage Network

```text
pve-prod-01 (Minisforum MS-A2)
  SFP+ Port 1 — 10.0.0.1/30
        |
        | 10GbE DAC — point-to-point
        |
nas-prod-01 (Intel X710-DA2)
  eth2 / X710 Port 1 — 10.0.0.2/30
```

This link:

- Does not traverse the UniFi switch.
- Does not use a VLAN tag.
- Uses the private subnet `10.0.0.0/30`.
- Provides a dedicated path for NAS storage access from `pve-prod-01`.
- Is not directly reachable by `pve-prod-02` or `pve-prod-03`.

### Proxmox NFS Storage Mapping

The three-node Proxmox cluster maintains two definitions for the same Unraid ISO export.

| Storage ID | Proxmox Nodes | NAS Endpoint | Export |
|---|---|---|---|
| `nas-isos` | pve-prod-01 | `10.0.0.2` | `/mnt/user/isos` |
| `nas-isos-p2` | pve-prod-02, pve-prod-03 | `192.168.30.16` | `/mnt/user/isos` |

**November 14 verification:**

- `nas-isos` was available on `pve-prod-01`.
- `nas-isos-p2` was available on `pve-prod-02` and `pve-prod-03`.
- Both storage definitions referenced the same NAS export.
- The private 10GbE path was operational.

The separate storage IDs preserve the primary node's dedicated DAC connection without requiring the additional nodes to use an unreachable storage subnet.

### Management Network Terminology

The NAS management subnet remains `192.168.10.0/24`.

Earlier documentation referred to this network as **VLAN 10**.

Live UniFi inspection on November 14 reported that the Management network is actually UniFi's **default network, VLAN ID 1**, using untagged management traffic.

This is a documentation terminology correction, not a network configuration change.

The NAS interface remains:

`192.168.10.10/24`

No management addressing or VLAN configuration was modified.

### Network Security Considerations

The intended design separates the Unraid management plane from production service access.

Services normally use the NAS data address:

`192.168.30.16`

However, Docker PBS uses Host networking, so TCP 8007 may also be reachable through the management address:

`192.168.10.10`

The Unraid Docker integration currently advertises the PBS web UI using the management address.

That alone does not establish which source networks can reach it.

**Open verification:** Confirm the actual PBS listener binding and firewall reachability before describing TCP 8007 as restricted to VLAN 30.

See:

- `architecture/network/ip-addresses.md`
- `architecture/network/vlan-design.md`
- `architecture/network/firewall-rules.md`

---

## Array — Parity Array (XFS)

**Current state:** STARTED  
**Last live verification:** November 14, 2026  
**Configuration:** Two parity disks and three XFS data disks

The read-only Unraid integration confirmed all five assigned array devices report `DISK_OK` with zero reported device errors.

| Role | Drive | Size | Serial | Verified Status |
|---|---|---|---|---|
| Parity 1 | WD Red Pro (WD121KFBX-68EF5N0) | 12 TB | `AAHKXP9H` | `DISK_OK` |
| Parity 2 | WD Red Pro (WD121KFBX-68EF5N0) | 12 TB | `2AH2JSWY` | `DISK_OK` |
| Disk 1 | Seagate IronWolf (ST6000VN0033) | 6 TB | `ZAD97T6M` | `DISK_OK` |
| Disk 2 | Seagate IronWolf (ST6000VN0033) | 6 TB | `ZAD97RT0` | `DISK_OK` |
| Disk 3 | WD Red Pro (WD121KFBX-68EF5N0) | 12 TB | `5PK7HDXF` | `DISK_OK` |

**Current configuration details:**

- Two parity disks.
- Three installed XFS data disks.
- Approximately 24 TB decimal usable data capacity (approximately 22 TiB), excluding filesystem overhead.
- Parity disks do not contribute usable share storage.
- Disk 3 is installed and operational, not a future expansion.
- The array remains online after the October 31 reconstruction.

### November 14 Array Verification

The live integration reported:

| Check | Result |
|---|---|
| Array state | `STARTED` |
| Parity 1 | `DISK_OK`, zero reported errors |
| Parity 2 | `DISK_OK`, zero reported errors |
| Disk 1 | `DISK_OK`, zero reported errors |
| Disk 2 | `DISK_OK`, zero reported errors |
| Disk 3 | `DISK_OK`, zero reported errors |
| Parity check status | `COMPLETED` |
| Active parity-check operation | None reported |

The API did not supply a fresh parity-check completion timestamp or a new full parity-integrity assessment.

The reported state confirms that the array is operating and its assigned devices are healthy according to the available Unraid status information.

It does not replace a fresh non-correcting parity check, extended SMART testing, or end-to-end file-content verification.

### Historical Parity Verification

The March 11, 2026 parity synchronization completed in approximately 19 hours and 28 minutes with zero reported errors.

This result describes an earlier array configuration.

The October 31 reconstruction is documented separately below.

---

### Parity 1 and Disk 3 Connection Incident — Resolved (October 31, 2026)

**Status:** Resolved  
**Probable cause:** Shared SAS-to-SATA breakout cable or connector-path fault  
**Root cause certainty:** Probable, not conclusively isolated

### Incident History

Parity 1 was disabled (`DISK_DSBL`, red X) while assigned to a 14 TB Seagate IronWolf Pro (`ST14000NT001`).

The previously installed 12 TB WD Red Pro had exhibited similar symptoms and was replaced with the Seagate, which subsequently developed the same problem.

Disk 3, a separate 12 TB WD Red Pro, also disappeared from the active array.

The operator confirmed that Parity 1 and Disk 3 shared one SAS-to-SATA breakout cable.

Possible fault sources included:

- SAS-to-SATA breakout cable.
- Cable connectors.
- HBA or backplane connection path.
- Power delivery.
- Other interface-level problems.

The breakout cable was considered the leading suspect, but no component-specific electrical test conclusively isolated the original fault.

### Recovery Performed

1. Shut down `nas-prod-01`.
2. Physically replaced the suspected SAS-to-SATA breakout cable.
3. Removed the temporary 14 TB Seagate from the Parity 1 slot.
4. Reinstalled the original 12 TB WD Red Pro, serial `AAHKXP9H`, as Parity 1.
5. Reassigned the established 12 TB WD Red Pro, serial `5PK7HDXF`, as Disk 3.
6. Started the array and accepted the Unraid reconstruction operation.
7. Confirmed the system performed simultaneous Parity 1 and Disk 3 reconstruction.
8. Verified completion with zero reported reconstruction errors.

The system log included:

```text
md: recovery thread: recon P D3 ...
```

The reconstruction completed in approximately 20 hours and 18 minutes (73,119 seconds).

### Post-Recovery Verification — October 31

| Check | Result |
|---|---|
| Array | STARTED |
| Parity 1 | Valid, `DISK_OK` |
| Parity 2 | `DISK_OK` |
| Disk 1 | `DISK_OK` |
| Disk 2 | `DISK_OK` |
| Disk 3 | `DISK_OK` |
| Disk errors | Zero reported |
| Disk 3 filesystem | `/dev/md3p1`, XFS, mounted read/write at `/mnt/disk3` |
| Disk 3 data | Approximately 2.2 TB used at time of check |
| Directory traversal | Completed successfully |
| Parity 1 SMART | PASSED |
| Disk 3 SMART | PASSED |
| Parity 1 SATA CRC historical count | 454 |
| Disk 3 SATA CRC historical count | 29 |
| New interface errors | None observed in reviewed post-replacement logs |

### Historical XFS Warning

Before reconstruction, an Unassigned Devices mount of physical Disk 3 (`/dev/sdf1`) failed on October 30 with XFS metadata corruption messages.

At that time, the emulated Unraid filesystem (`/dev/md3p1`) mounted successfully.

Following reconstruction:

- `/dev/md3p1` mounted normally.
- Its directory tree was traversable.
- No active filesystem-access problem remained in the verified observations.

The earlier physical-mount warning is retained as historical diagnostic evidence.

It should not be described as proof of an ongoing active array failure.

### Root Cause Assessment

Both affected drives had historical interface CRC errors.

The replacement Parity 1 drive developed the same type of problem as the earlier drive.

After replacing the shared breakout cable:

- Reconstruction completed successfully.
- Short SMART tests passed.
- No new interface CRC counts were observed during the reviewed verification period.
- Both affected devices returned to `DISK_OK`.

**Conclusion:** A cable or shared connection-path fault is the best-supported explanation.

However, a cable-specific test was not performed, so the root cause remains probable rather than definitive.

### Remaining Assurance Measures

The following were not completed as part of the repair:

- Full XFS metadata check.
- Full-file-content verification.
- Extended SMART surface tests.
- Separate non-correcting parity check after reconstruction.

These are optional assurance checks rather than evidence of a continuing known failure.

Continue normal SMART, parity, temperature, and error-counter monitoring.

### Historical Documentation Correction

An earlier storage document described Disk 3 as not installed on October 30.

The operator confirmed Disk 3 was already an established data disk.

The original Phase 2 commissioning documentation described an earlier two-data-disk configuration.

The precise date Disk 3 entered service was not established.

This runbook records the verified five-device array configuration.

### ZFS Impact

The independent `zfs-mirror` pool was not involved in the parity reconstruction.

Its October 30 `zpool status -v` result was healthy, and no ZFS configuration change was required as part of the repair.

---

## Pool — ZFS Mirror (`zfs-mirror`)

The NAS uses a separate mirrored ZFS pool for critical storage.

| Role | Drive | Size | Model |
|---|---|---|---|
| Mirror disk 1 | WD Red Plus | 4 TB | WD40EFRX |
| Mirror disk 2 | WD Red Plus | 4 TB | WD40EFPX |

**Usable capacity:** Approximately 3.68 TB  
**Filesystem:** ZFS mirror  
**Purpose:** Photos, backups, and selected system data

### Current Disk Status — November 14, 2026

The live Unraid array API reported:

| Device | Status | Reported Errors |
|---|---|---|
| WD40EFRX | `DISK_OK` | 0 |
| WD40EFPX | `DISK_OK` | 0 |

Both devices were present in the `zfs-mirror` pool-device inventory.

The read-only integration did not expose a current native `zpool status -v` result during this review.

Therefore, the November 14 disk-level status should not be represented as a new direct verification of the complete ZFS pool state or scrub history.

### Previous ZFS Verification — October 30, 2026

A direct `zpool status -v` check reported:

- Pool `zfs-mirror`: `ONLINE`.
- Both mirror members: `ONLINE`.
- Read errors: 0.
- Write errors: 0.
- Checksum errors: 0.
- Known data errors: None.

No ZFS reconfiguration was performed during the October 31 parity incident.

### Datasets and Snapshot Policy

The critical shares include:

- `photos`
- `backups`
- `system`

Historical documentation describes ZnapZend snapshots on the photos and backups datasets:

| Snapshot Cadence | Historical Retention |
|---|---|
| Daily | 7 days |
| Weekly | 30 days |

These settings were documented before the November PBS changes.

Their current enabled state, actual snapshot inventory, and latest successful execution were not independently rechecked on November 14.

**Important:** The PBS datastore is a directory under the backups dataset, not a separately verified child ZFS dataset.

The operator elected not to create a migration-time protective ZFS snapshot during the November 3–5 PBS migration.

### Storage Safety

A healthy mirror protects against certain single-drive failures.

It does not protect against every form of data loss, including:

- Accidental deletion.
- Application-level corruption.
- Destructive administrative changes.
- Loss of the entire NAS.
- Simultaneous loss of both mirror devices.
- Damage to an unprotected recovery history.

Independent backup and restoration procedures remain necessary for irreplaceable photos, configuration, and backup data.

---

## Shares

| Share | Storage Location | Path | Purpose |
|---|---|---|---|
| `data` | Parity array | `/mnt/user/data` | Single share for media and downloads; supports hardlinks |
| `appdata` | Parity array | `/mnt/user/appdata` | NAS Docker application configuration |
| `isos` | Parity array | `/mnt/user/isos` | Proxmox ISO images |
| `photos` | ZFS mirror | `/mnt/user/photos` | Immich photo library |
| `backups` | ZFS mirror | `/mnt/user/backups` | PBS datastore and Docker-host rsync backups |
| `system` | ZFS mirror | `/mnt/user/system` | Unraid system share |

All six share names were returned by the live Unraid integration on November 14.

The locations above reflect the established storage design and prior direct filesystem checks.

The share API response did not independently revalidate every configured storage policy, NFS export option, or underlying mount path.

### `data` Share Folder Structure

```text
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

This directory structure reflects the previously documented media application layout.

Its contents were not recursively rechecked during the November 14 review.

### Media and Downloads — Hardlink Design

The `data` share combines media and download directories into one NFS-exported filesystem.

`docker-prod-01` mounts:

```text
192.168.30.16:/mnt/user/data → /data
```

**Protocol:** NFSv3, per established host configuration.

The single mount is intentional.

Separate NFS mounts for downloads and media can prevent applications from creating hardlinks across paths, even if the underlying data resides on the same NAS.

The consolidated `data` share allows ARR applications and qBittorrent to operate on a consistent `/data` filesystem.

See `architecture/decisions-log.md` for the original design rationale.

### `backups` Share Folder Structure

```text
/mnt/user/backups/
├── pbs/
│   └── [PBS-managed datastore contents]
├── appdata/
│   ├── docker-prod-01/
│   ├── auth-prod-01/
│   └── immich-prod-01/
└── stacks/
    ├── docker-prod-01/
    ├── auth-prod-01/
    └── immich-prod-01/
```

The directory structure is the established rsync and PBS design.

A new complete directory inventory was not performed during the November 14 documentation review.

### PBS Datastore Ownership

**Active datastore directory:**

```text
/mnt/zfs-mirror/backups/pbs
```

**Docker container directory:**

```text
/backups
```

The datastore is managed by PBS.

The November migration verified:

- Directory ownership: UID/GID `34:34`.
- Datastore root mode: `755`.
- `.chunks` directory mode: `750`.
- Existing `ct` and `vm` backup groups.
- Existing `.lock` file and PBS metadata.

Do not manually place unrelated files inside the datastore.

Do not recursively change ownership or permissions without identifying the specific problem.

Do not manually delete PBS chunks, manifests, indexes, or lock files as a general troubleshooting technique.

---

## NFS Exports

The following reflects the established NFS configuration and the available Proxmox storage verification.

### Documented Exports

- `data`
- `appdata`
- `isos`
- `photos`
- `backups`

### Historical NFS Configuration

| Setting | Documented Value |
|---|---|
| Maximum NFS protocol | NFSv3 |
| Share access | Private |
| Permitted networks | Service and storage networks, with individual exports previously inspected |
| Root squashing | `no_root_squash` |
| SMB | Disabled on documented NAS shares |
| Historical share permissions | `nobody:users` with documented access permissions |

The October incident also identified `async` on the backups export.

These export options were not freshly read back from Unraid on November 14.

### Verified Proxmox ISO Connectivity

The live Proxmox integration confirmed:

| Node | Storage | Network Path | Result |
|---|---|---|---|
| pve-prod-01 | `nas-isos` | 10.0.0.2 | Available |
| pve-prod-02 | `nas-isos-p2` | 192.168.30.16 | Available |
| pve-prod-03 | `nas-isos-p2` | 192.168.30.16 | Available |

This verifies successful storage access through the relevant interfaces.

It does not independently establish all NFS export options or access restrictions.

### Security Considerations

NFSv3 with `no_root_squash` allows privileged client-side operations to carry substantial authority on exported storage.

Maintain narrowly scoped export access and avoid exposing NFS to untrusted networks.

Do not change working exports as part of documentation cleanup.

---

## PBS Docker — Active Deployment

**Deployment:** November 3–5, 2026  
**Host:** nas-prod-01  
**Current status:** RUNNING  
**Image:** `ayufan/proxmox-backup-server:latest`  
**Image type:** Unofficial third-party build  
**Container name:** `proxmox-backup-server`  
**PBS UI version:** 4.2.0 at last direct application-version verification  
**Last container-state verification:** November 14, 2026

### Architecture

The current PBS service runs as a Docker container directly on Unraid.

```text
Proxmox cluster
├── pve-prod-01
├── pve-prod-02
└── pve-prod-03
        |
        | PBS API / TCP 8007
        |
nas-prod-01 — 192.168.30.16
        |
        | Docker Host networking
        |
proxmox-backup-server container
        |
        | /backups
        |
/mnt/zfs-mirror/backups/pbs
        |
        | Direct local ZFS filesystem
        |
zfs-mirror/backups
```

The active datastore does **not** pass through an NFS mount or Unraid `shfs`/FUSE layer inside PBS.

This is a deliberate change from the retired VM-based deployment.

### PBS Configuration

| Setting | Value |
|---|---|
| Proxmox storage ID | `pbs-prod-01` |
| PBS host address | `192.168.30.16` |
| PBS TCP port | `8007` |
| PBS datastore | `nas-backups` |
| Datastore container path | `/backups` |
| Datastore host path | `/mnt/zfs-mirror/backups/pbs` |
| Networking | Docker Host |
| Privileged mode | Disabled |
| Timezone | `America/Chicago` |
| Auto-start | Enabled |
| Memory limit | 2 GiB |

**Extra Docker parameters:**

```text
--restart unless-stopped
--memory=2g
--mount type=tmpfs,destination=/run
--stop-signal=SIGHUP
```

### Persistent Container Paths

| Host Path | Container Path | Purpose |
|---|---|---|
| `/mnt/user/appdata/pbs/etc` | `/etc/proxmox-backup` | PBS configuration and protected secrets |
| `/mnt/user/appdata/pbs/logs` | `/var/log/proxmox-backup` | Logs |
| `/mnt/user/appdata/pbs/lib` | `/var/lib/proxmox-backup` | PBS runtime state |
| `/mnt/zfs-mirror/backups/pbs` | `/backups` | PBS datastore, direct ZFS access |

These mappings were established during the November PBS Docker migration.

The November 14 read-only Docker API confirmed the container was running but did not independently return its complete bind-mount configuration.

### Docker Status — November 14

The live Unraid integration reported:

| Check | Result |
|---|---|
| Container | `proxmox-backup-server` |
| State | RUNNING |
| Auto-start | Enabled |
| Image | `ayufan/proxmox-backup-server:latest` |
| Image update flag | No update reported |
| Orphaned | No |

The absence of a reported image update does not establish that the unofficial image is current with upstream Proxmox Backup Server security releases.

### Proxmox Cluster Integration

The Proxmox storage definition was independently verified November 14.

| Setting | Value |
|---|---|
| Storage ID | `pbs-prod-01` |
| Storage type | PBS |
| Server | `192.168.30.16` |
| Datastore | `nas-backups` |
| Authentication | `pve-backup@pbs!pve-cluster` |
| Shared across cluster | Yes |
| pve-prod-01 | Available |
| pve-prod-02 | Available |
| pve-prod-03 | Available |

The same storage ID is retained across the three-node cluster.

**Important:** The active PBS service is on the NAS, not inside a Proxmox VM.

The former VM 103 has been destroyed.

### Authentication and Access

**Automated PVE backups:**

- Account/token: `pve-backup@pbs!pve-cluster`.
- Access restricted to the intended `nas-backups` datastore according to the documented configuration.
- Existing backup-group ownership was reassigned during migration.

**Interactive administration:**

- Authentik OIDC.
- Local emergency administrator `admin@pbs`.

The shared Proxmox/Authentik OIDC client secret was rotated during PBS migration and interactive login tests passed on the then-existing Proxmox nodes and PBS.

A post-migration Authentik SSO functional test also passed November 14 after VM 104 was moved to `pve-prod-03`.

No credentials or client secrets are recorded in this document.

### Current Backup Schedule

Proxmox Datacenter backup job:

`backup-54015f90-26e5`

| Setting | Value |
|---|---|
| Enabled | Yes |
| Schedule | Daily 02:00 |
| Mode | Snapshot |
| Storage | `pbs-prod-01` |
| Guest selection | All current cluster guests |
| Exclusion | VM 103 (retired; harmless stale exclusion) |

This configuration was live-verified November 14.

The backup job covers all seven registered Proxmox guests.

### Backup Verification — November 14

The scheduled 02:00 jobs completed successfully on all three nodes.

| Proxmox Node | Guests Backed Up | Result |
|---|---|---|
| pve-prod-01 | 100, 101, 104, 106, 107 | `TASK OK` |
| pve-prod-02 | 105 | `TASK OK` |
| pve-prod-03 | 102 | `TASK OK` |

The scheduled backup of Authentik VM 104 ran on node 01 before the host migration.

Later on November 14, VM 104 was moved to node 03.

A separate manual snapshot backup then completed successfully from the new host.

**Authentik post-migration backup:**

| Metric | Result |
|---|---|
| Guest | VM 104 — auth-prod-01 |
| Current host | pve-prod-03 |
| Target | `pbs-prod-01` |
| Completion | November 14, 2026, 14:09:45 CST |
| Backup mode | Snapshot |
| Virtual disk size | 32 GiB |
| Sparse zero data | 11.59 GiB (36%) |
| Data reused | 30.52 GiB (95%) |
| Duration | 44 seconds |
| Result | Successful |

The EFI disk was also included.

The post-migration backup demonstrates that the Lenovo can back up the Authentik VM successfully through the shared PBS configuration.

### Backup Recovery Verification

**Previously verified November 5:**

- Manual CT 100 backup completed successfully.
- CT 100 snapshot passed targeted PBS integrity verification.
- File-level recovery of CT 100 `/etc/hostname` succeeded.
- Scheduled backup jobs passed on the then-existing two-node cluster.

**Subsequently verified November 14:**

- Successful scheduled backups from all three current nodes.
- Successful manual backup of Authentik from its new node.

**Still outstanding:**

- Isolated full VM restore.
- Isolated full LXC restore.
- Verification of recent backup archives across all guests.
- Successful execution of scheduled prune/GC/verify tasks following the Docker migration.
- Validation of actual warning/error notification delivery.
- Tested recovery of PBS container configuration and datastore.

A successful backup task is not equivalent to a complete disaster recovery test.

### Retention and Maintenance

The following settings were last visually verified on the Docker PBS instance in November 2026:

| Operation | Schedule / Configuration |
|---|---|
| Prune | Daily |
| Keep last | 3 |
| Keep daily | 7 |
| Keep weekly | 4 |
| Keep monthly | 3 |
| Garbage collection | Saturday 04:00 |
| Verification | Saturday 05:00 |
| Reverification of older snapshots | After approximately 30 days |

Actual maintenance-job success has not been reverified during this documentation update.

**Capacity planning:** Approximately 500 GB is the operator's preferred soft PBS budget, not an enforced quota.

### PBS Security and Lifecycle Risks

- The image is unofficial and has been described as unmaintained.
- It may lag upstream security fixes.
- Docker Host networking can expose PBS on multiple host interfaces.
- PBS container settings and secrets are not covered by the Proxmox guest backup job.
- The NAS ZFS pool and datastore remain separate recovery dependencies.
- The container must not start before required persistent storage is available.
- Automatic start-after-ZFS-mount behavior has not been independently tested.
- Changes to the datastore should use supported PBS operations.
- Image updates should be handled through Unraid's container update/recreation workflow with the configuration and storage mappings preserved.

Do not run the retired PBS VM against the active datastore.

VM 103 no longer exists.

---

### PBS NFS Filesystem and October 2026 Incident — Historical

**Historical only:** This section describes the retired PBS VM architecture.

The active Docker PBS deployment does not use the old NFS/FUSE datastore access path.

### Former Storage Topology

```text
PBS VM 103 — former pbs-prod-01
  /mnt/backups/pbs
        |
        | NFSv3
        |
192.168.30.16:/mnt/user/backups
        |
        | Unraid fuse.shfs
        |
zfs-mirror/backups
        |
/mnt/zfs-mirror/backups/pbs
```

### Historical NAS Configuration

The October 30 investigation established:

- Share: `backups`.
- Primary pool: `zfs-mirror`.
- Secondary pool: None.
- `shareUseCache="only"`.
- `shareCachePool="zfs-mirror"`.
- Permit exclusive shares: No.
- NFS export path: `/mnt/user/backups`.
- Export `fsid=101`.
- Export used `async`, `no_subtree_check`, and `no_root_squash`.
- Permitted networks included `192.168.30.0/24`, `192.168.10.0/24`, and `10.0.0.0/30`.
- Unraid FUSE filesystem: `fuse.shfs`.
- `fuse_remember="330"` observed.

### Historical PBS VM Client Mount

The former PBS VM mounted the NAS export using NFSv3.

Observed mount options included:

```text
soft,timeo=30,retrans=3
```

The client used systemd automount.

Those options are historical and do not describe the current PBS Docker datastore.

### Incident Evidence

The former PBS proxy retained a stale descriptor to:

```text
/mnt/backups/pbs/.lock
```

Backup tasks failed with:

```text
ESTALE: Stale file handle
```

Clients reported connection resets and HTTP/2 failures.

The historical investigation confirmed:

- A long-lived stale file descriptor existed.
- The current `.lock` pathname was accessible.
- New locks could be acquired.
- Restarting the proxy closed the stale descriptor.
- Backup execution temporarily recovered.

The original event invalidating the descriptor was never conclusively identified.

FUSE/NFS file-handle handling was a leading hypothesis.

### Temporary Recovery — October 30

The operator restarted:

```bash
systemctl restart proxmox-backup-proxy
```

The subsequent manual CT 100 backup and both scheduled PVE backup jobs succeeded.

CT 100 snapshot integrity verification also passed.

No NAS export, share, ownership, filesystem, or global exclusive-share configuration was modified.

### Recurrence and Final Architecture Decision

The same failure returned October 31–November 2.

The operator elected to replace the VM-based PBS deployment with Docker PBS directly on Unraid.

The new implementation uses:

```text
/mnt/zfs-mirror/backups/pbs → /backups
```

This eliminates the previous NFS/FUSE datastore access path from PBS.

The former VM 103 was initially retained stopped, then deleted before the November 13 Proxmox node replacement.

**Do not use the historical proxy restart, NFS mount, or old VM recovery procedures as instructions for the current Docker PBS deployment.**

The operator accepted the risk of proceeding without an additional migration-time ZFS snapshot.

For complete incident chronology, see:

- `runbooks/backup-schedule.md`
- `runbooks/backup-restore.md`

---

## SSH Access

SSH is enabled on `nas-prod-01`.

The established backup scripts authenticate using the `root` SSH account.

### Authorized Keys

The documented `/root/.ssh/authorized_keys` configuration includes dedicated backup public keys for:

| Key Identity | Host | Purpose |
|---|---|---|
| `backup@docker-prod-01` | docker-prod-01 | Nightly rsync |
| `backup@auth-prod-01` | auth-prod-01 | Nightly rsync |
| `backup@immich-prod-01` | immich-prod-01 | Nightly rsync |

Private keys reside on participating backup clients at:

```text
/root/.ssh/backup_rsa
```

The rsync destination address is:

`192.168.30.16`

This uses the Services network rather than the NAS management address.

### SSH Security

The backup keys are service credentials.

They should be:

- Restricted to authorized hosts.
- Protected by appropriate filesystem permissions.
- Stored outside version control.
- Rotated if compromised.
- Included in the recovery/credential management plan.

The live Unraid integration did not expose the current `authorized_keys` file, so the exact active key inventory was not independently read back on November 14.

### Firewall Boundaries

The documented Services-to-Management firewall policy blocks general access to the NAS management IP from production service hosts.

The current firewall design also contains targeted exceptions for other infrastructure services.

Do not assume every possible route to a NAS management service has been packet-level tested.

See `architecture/network/firewall-rules.md`.

---

## AI Observer — Unraid Monitoring Integration

`observer-prod-01` runs the read-only Unraid MCP integration.

| Setting | Value |
|---|---|
| Observer host | `observer-prod-01` |
| Observer IP | `192.168.30.17` |
| Proxmox node | `pve-prod-01` |
| NAS endpoint | `192.168.30.16:443` |
| Unraid API | GraphQL |
| MCP implementation | dinglebear-ai Rust Unraid MCP |
| MCP version | 0.6.3 |
| Exposed tools | 26 read-only |
| Write tools | None exposed |

The MCP connects using HTTPS and protected API credentials.

Its permissions and tool exposure are documented in:

`services/infra/ai-observer.md`

Operational procedures are maintained in:

`runbooks/ai-observer-operations.md`

### November 14 Integration Verification

The read-only integration successfully returned:

- Unraid operating system version.
- CPU and RAM inventory.
- Network interface status.
- Array state and disks.
- Share inventory.
- Docker container inventory.
- Basic MCP service status.

The Unraid UPS and UPS configuration requests failed upstream during this review.

These failures do not demonstrate that NUT itself is unhealthy; they establish only that the integration could not retrieve those data.

Do not infer current UPS server settings from an unsuccessful API query.

The network API's broader summary query also returned an error, while the more specific network-interface query succeeded.

The verified interface results are recorded in the Network Interfaces section.

### Security Boundaries

The ChatGPT-facing Unraid MCP exposes read-only tools.

It does not provide general NAS shell access, Docker write operations, direct filesystem modification, or Unraid administrative changes.

Read-only queries can still return operational details and potentially sensitive configuration information, so results should be handled appropriately.

---

## Docker Containers

The November 14 Unraid integration returned three running containers.

| Container | Image | Network / Access | Purpose | Status |
|---|---|---|---|---|
| `plex` | `lscr.io/linuxserver/plex` | Documented Plex service IP `192.168.30.2` | Plex Media Server | RUNNING |
| `proxmox-backup-server` | `ayufan/proxmox-backup-server:latest` | Docker Host; intended PBS endpoint `192.168.30.16:8007` | Proxmox Backup Server | RUNNING |
| `beszel-agent` | `henrygd/beszel-agent:latest` | Agent networking not independently inspected | Host monitoring | RUNNING |

All three reported auto-start enabled.

The image-update flag was false for all three at the time of inspection.

This is a point-in-time container status and does not establish that images are current with upstream releases.

### Plex Configuration

The documented Plex configuration includes:

- Intel QuickSync hardware transcoding.
- `/dev/dri` GPU access.
- Intel UHD 730 integrated GPU.
- Hardware encoding enabled.
- HEVC encoding configured for HEVC sources only.
- Media path `/data` mapped to `/mnt/user/data`.
- Persistent appdata at `/mnt/user/appdata/plex`.
- Service address `192.168.30.2`.
- Direct external access through the documented Plex port-forward architecture.

The November 14 Docker API verified that Plex was running but did not independently re-read its full environment variables, bind mounts, or transcoder configuration.

Those details remain based on the established service documentation.

See `services/media/plex.md`.

### Beszel Agent

The NAS runs `beszel-agent` for monitoring.

The container was confirmed running on November 14.

Its exact port binding, token configuration, and monitoring hub registration were not independently inspected during this review.

See `services/monitoring/beszel.md`.

---

## UPS & Graceful Shutdown

**Architecture:** A Tripp-Lite UPS is connected to `nas-prod-01` by USB.

Unraid hosts the Network UPS Tools (NUT) server.

The three Proxmox hosts run NUT network clients.

### Documented UPS Server Configuration

| Setting | Documented Value |
|---|---|
| UPS | Tripp-Lite SMART1500LCDXL |
| Capacity | 1500 VA / 900 W |
| USB connection | UPS → nas-prod-01 |
| NUT plugin | dmacias/Network UPS Tools |
| Previously documented NUT version | 2.8.4 |
| Server mode | Netserver |
| Driver | `usbhid-ups` |
| Driver port | `auto` |
| NUT server address | `192.168.10.10` |
| UPS identifier | `ups` |
| Client endpoint | `ups@192.168.10.10` |
| Documented NAS shutdown threshold | Six minutes on battery |

### Historical UPS Measurements

Previously documented values included:

- UPS status: On Line.
- Battery charge: 100%.
- Estimated runtime: Approximately 34 minutes.
- Load: Approximately 15%.
- Apparent load: Approximately 225 VA.
- Nominal capacity: 1500 VA.

These were historical point-in-time readings.

They should not be treated as current battery performance guarantees.

### Proxmox NUT Clients

| Node | Management IP | Role | Last Documented Verification |
|---|---|---|---|
| pve-prod-01 | 192.168.10.11 | Secondary network client | Operational before November 13 |
| pve-prod-02 | 192.168.10.12 | Secondary network client | Enabled, active, UPS `OL` on November 13 |
| pve-prod-03 | 192.168.10.13 | Secondary network client | Enabled, active, UPS `OL` on November 14 |

The November 13 deployment established operational NUT monitoring on both newly deployed Proxmox nodes.

The replacement Dell and Lenovo use `nut-client` 2.8.1-5.

### Client Configuration

**File:** `/etc/nut/nut.conf`

```text
MODE=netclient
```

**File:** `/etc/nut/upsmon.conf`

Documented settings include:

- MONITOR target: `ups@192.168.10.10`.
- Secondary monitoring role.
- Power value: 1.
- `MINSUPPLIES 1`.
- `POLLFREQ 5`.
- `HOSTSYNC 15`.
- `DEADTIME 15`.
- `FINALDELAY 5`.
- Shutdown command: `/sbin/shutdown -h +0`.
- Powerdown flag: `/etc/killpower`.

Protected monitoring credentials are not recorded in this document.

### Debian NUT Systemd Workaround

Nodes 02 and 03 required a local systemd drop-in because the installed Debian NUT service referenced a nonexistent tmpfiles configuration.

The corrected pre-start reference is:

```text
/usr/lib/tmpfiles.d/nut-client.conf
```

Both services subsequently started successfully.

The systemd pre-start operation reported `0/SUCCESS`.

This workaround should be reviewed after future NUT package updates.

### November 14 Destination-Node Verification

Following physical maintenance and the Authentik migration, the operator checked `pve-prod-03`:

```bash
systemctl is-enabled nut-monitor
systemctl is-active nut-monitor
upsc ups@192.168.10.10 ups.status
```

Results:

```text
enabled
active
Init SSL without certificate database
OL
```

This verifies that the Lenovo's client was active and could query the remote UPS.

It does not establish that the entire UPS-triggered shutdown process has been tested.

### Current NAS Server Verification Boundary

On November 14, the read-only Unraid integration could not retrieve UPS or UPS-configuration information because the upstream API returned errors.

The documented NUT server configuration remains based on earlier observations.

The successful `OL` query from `pve-prod-03` confirms that the remote UPS endpoint responded at the time of the operator's check.

It does not independently confirm the NAS's current six-minute shutdown timer, detailed server policy, or physical UPS outlet mapping.

### Intended Shutdown Sequence

1. Utility power fails.
2. NUT server detects the UPS on battery.
3. The configured NAS shutdown timer allows for a temporary outage.
4. At the configured shutdown condition, the server signals its NUT clients.
5. Proxmox nodes shut down their VMs and LXCs.
6. All three Proxmox hosts shut down.
7. The NAS shuts down after dependent systems where the configured sequence permits.

### End-to-End Validation Still Outstanding

The following have not been tested as a complete production sequence:

- Real UPS-triggered forced-shutdown signaling.
- Guest shutdown completion on each Proxmox node.
- NAS-last shutdown behavior.
- Actual power connectivity of all three Proxmox nodes to battery-backed UPS outlets.
- Battery runtime under full production load.
- Restart behavior following restoration of utility power.

Do not trigger `upsmon -c fsd` or deliberately interrupt UPS power during routine monitoring.

A full shutdown test requires an approved maintenance plan, suitable backups, and explicit recovery steps.

### Rack and UPS Cabling

The NAS is installed in the Rosewill chassis at U8–U11.

The UPS is installed at U18.

The latest physical rack layout is:

| Device | Rack Position |
|---|---|
| pve-prod-01 | U5–U6 |
| pve-prod-02 | U17 |
| pve-prod-03 | U7 |
| pi-prod-01 | U7, sharing shelf with Lenovo |
| nas-prod-01 | U8–U11 |
| UPS | U18 |

The latest rack documentation confirms the physical locations of the new Dell and Lenovo nodes.

UPS outlet assignment details remain unverified.

See `architecture/hardware/rack-layout.md`.

---

## Plugins Installed

The following plugin inventory is preserved from the existing NAS documentation.

| Plugin | Purpose | Verification |
|---|---|---|
| NUT | UPS monitoring and shutdown | Previously documented; live plugin query unavailable November 14 |
| ZFS Master | ZFS pool management UI | Historical installed-plugin inventory |
| ZnapZend | ZFS snapshot scheduling | Historical installed-plugin inventory |
| Intel-GPU-TOP | GPU monitoring | Historical installed-plugin inventory |
| Dynamix System Temp | CPU and board temperature monitoring | Historical installed-plugin inventory |
| Unassigned Devices | External drive management | Historical installed-plugin inventory |
| Fix Common Problems | Configuration health checks | Historical installed-plugin inventory |

The live Unraid plugin listing returned an upstream error during this documentation review.

Therefore, this table is not a newly verified installed-plugin inventory.

Do not remove entries based solely on the integration's inability to retrieve the list.

---

## Three-Node Proxmox Integration

The NAS serves the current `homelab-v3` cluster.

**Cluster state verified November 14, 2026:**

| Node | Hardware | Management IP | NAS Dependencies |
|---|---|---|---|
| pve-prod-01 | Minisforum MS-A2 | 192.168.10.11 | PBS, ISO export over private DAC, service-facing NAS exports |
| pve-prod-02 | Dell OptiPlex 5000 SFF | 192.168.10.12 | PBS, ISO export over Services network, Immich photo library |
| pve-prod-03 | Lenovo ThinkCentre M90q Tiny | 192.168.10.13 | PBS, ISO export over Services network, Authentik/secondary DNS backup coverage |

**Cluster characteristics:**

- Three voting Proxmox nodes.
- Two votes required for quorum.
- No external QDevice.
- No automatic Proxmox HA.
- Local ZFS guest storage on each node.
- Shared PBS backup storage hosted by Unraid.

### Current Guest Distribution

| Node | Guests |
|---|---|
| pve-prod-01 | CT 100, VM 101, VM 107; VM 106 stopped |
| pve-prod-02 | VM 105 (Immich) |
| pve-prod-03 | CT 102 (secondary AdGuard), VM 104 (Authentik) |

The Authentik VM was migrated from node 01 to node 03 on November 14.

Its application health, DNS, reverse proxy, SSO, and fresh PBS backup checks succeeded.

The NAS PBS endpoint did not need to change during the migration.

### Architectural Recovery Considerations

The NAS is a major dependency for the homelab.

Its failure can affect:

- NFS-mounted application data.
- Immich photo storage.
- Plex media storage.
- Central PBS backups.
- Docker-host rsync backups.
- Shared ISO library.
- NUT server availability.

Three-node Proxmox quorum does not eliminate NAS-related single points of failure.

The cluster is not configured for automatic guest failover.

Recovery assurance depends on usable independent copies of critical NAS data and tested restoration procedures.

---

## Open Verification Items

### Networking

- Confirm intended PBS TCP 8007 access through both NAS interfaces.
- Periodically recheck the dedicated `eth2` 10GbE link and Proxmox NFS access.
- Review export permissions and `no_root_squash` exposure during a separate security assessment.
- Confirm physical cabling and port profiles if topology changes.

### Storage and Data Protection

- Obtain a fresh native `zpool status -v` result.
- Review ZFS scrub and snapshot task history.
- Verify current ZnapZend schedule and retention.
- Continue parity, SMART, and interface-error monitoring.
- Review independent recovery protection for the photos and backups datasets.
- Test an appropriate NAS critical-data recovery scenario.

### PBS

- Verify scheduled prune/GC/snapshot-verification execution.
- Perform an isolated full VM or LXC restore.
- Confirm actual backup warning/error notification delivery.
- Review the unofficial PBS Docker image maintenance strategy.
- Verify persistent configuration backup and datastore recovery.
- Confirm PBS container startup ordering after ZFS pool availability.
- Confirm the first scheduled VM 104 backup from `pve-prod-03`.

### UPS

- Read back current NUT server settings directly from Unraid.
- Confirm the six-minute shutdown threshold.
- Reconcile physical UPS outlet assignments.
- Plan and safely validate end-to-end shutdown behavior.
- Verify recovery behavior following restoration of mains power.

### Monitoring and Documentation

- Recheck the current Unraid installed-plugin inventory when the API issue is resolved.
- Confirm Docker-host rsync success after the Authentik VM migration.
- Keep hardware, backup, network, and recovery documentation consistent with the three-node architecture.
- Preserve historical findings separately from current implementation state.

None of the above items should be marked complete without appropriate verification.

---

## Related Documentation

- `architecture/hardware/inventory.md` — NAS hardware, disk inventory, Proxmox hosts.
- `architecture/hardware/rack-layout.md` — Physical placement and cabling.
- `architecture/compute/pve-prod-01.md` — Primary node and dedicated storage link.
- `architecture/compute/pve-prod-02.md` — Dell node and Immich.
- `architecture/compute/pve-prod-03.md` — Lenovo node, secondary DNS, and Authentik.
- `architecture/network/ip-addresses.md` — Network addressing and interfaces.
- `architecture/network/vlan-design.md` — Management, Services, and storage network topology.
- `architecture/network/firewall-rules.md` — Network access restrictions and exceptions.
- `architecture/decisions-log.md` — NAS, storage, backup, and network architecture decisions.
- `architecture/storage/drives.md` — Detailed drive inventory.
- `services/media/plex.md` — Plex deployment and data paths.
- `services/photos/immich.md` — Immich database and NAS photo storage.
- `services/infra/authentik.md` — Authentik migration and service configuration.
- `services/infra/ai-observer.md` — Read-only infrastructure integrations.
- `services/monitoring/beszel.md` — Beszel deployment.
- `runbooks/backup-schedule.md` — Backup scheduling and verified status.
- `runbooks/backup-restore.md` — Recovery procedures.
- `runbooks/ai-observer-operations.md` — Observer diagnostics and operational procedures.

---

## Verification Summary — November 14, 2026

| Component | Result |
|---|---|
| Unraid version | **7.3.4**, live-verified |
| NAS management | `br0` at `192.168.10.10`, UP, 2.5 Gbps |
| NAS Services interface | `eth1` at `192.168.30.16`, UP, 10 Gbps |
| NAS dedicated storage interface | `eth2` at `10.0.0.2/30`, UP, 10 Gbps |
| Dedicated NFS path | `nas-isos` available from pve-prod-01 through `10.0.0.2` |
| Other Proxmox ISO path | `nas-isos-p2` available from nodes 02 and 03 |
| Array | STARTED |
| Parity drives | Both `DISK_OK`, zero reported errors |
| Data drives | All three `DISK_OK`, zero reported errors |
| ZFS mirror members | Both reported `DISK_OK`, zero device errors |
| Native ZFS pool readback | Not performed during this review |
| Unraid shares | All six documented shares returned |
| Plex Docker | RUNNING |
| PBS Docker | RUNNING |
| Beszel Agent Docker | RUNNING |
| PBS cluster storage | Available on all three Proxmox nodes |
| PBS scheduled job | Enabled, daily 02:00 |
| November 14 scheduled backups | Successful on nodes 01, 02, and 03 |
| November 14 Authentik manual backup | Successful from pve-prod-03 |
| Former PBS VM 103 | Deleted; not a rollback VM |
| UPS client on pve-prod-03 | Previously verified active, enabled, UPS `OL` |
| Live NUT server configuration | Could not be retrieved through current Unraid integration |
| Current installed-plugin inventory | Could not be retrieved through current Unraid integration |
| Full guest restore | Not yet verified |
| Network/firewall changes | None |
| Storage configuration changes | None |

**Assessment:** The NAS's documented network topology is now reconciled with current evidence. Its dedicated 10GbE storage link is implemented and operational, the active array disks report healthy status, the PBS Docker service is running, and all three Proxmox nodes can access their intended NAS-backed storage.

The remaining work concerns deeper recovery testing, current ZFS maintenance verification, UPS shutdown validation, and PBS lifecycle assurance—not a known active storage or network failure.
