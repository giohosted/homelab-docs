# Backup Schedule

**Last Updated:** 2026-11-05  
**Status:** PBS Docker deployment active; November 5 scheduled backups succeeded on both nodes, targeted verification and file-level restore passed; full-guest restore and long-term reliability remain unverified

Reference document for backup jobs across Homelab v3. Lists what is configured, when jobs run, where backups are stored, and how their success is verified.

**Important:** A configured or enabled backup job does not guarantee that usable recovery points exist. Backup execution, archive availability, integrity, and restoration must be verified separately.

---

## Schedule Overview

|Time|Job|Source|Destination|Host|Status|
|---|---|---|---|---|---|
|02:00 daily|PBS — VM & CT backup|Proxmox VMs and LXCs, excluding retired VM 103|nas-prod-01 `/mnt/zfs-mirror/backups/pbs` (direct ZFS bind mount)|pve-prod-01 and pve-prod-02|**Verified successful 2026-11-05 on both nodes**|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|docker-prod-01|Documented operational; latest execution not reverified|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|auth-prod-01|Documented operational; latest execution not reverified|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|immich-prod-01|Documented operational; latest execution not reverified|
|Every 3 days, ~04:03|Plex built-in DB backup|Plex library and blobs databases|`/mnt/user/appdata/plex`|nas-prod-01|Last verified 2026-10-26|
|04:00 Saturday|PBS — Garbage Collection|nas-backups datastore|—|nas-prod-01 (PBS Docker)|Configured on new PBS; execution not yet verified|
|05:00 Saturday|PBS — Verify|nas-backups datastore|—|nas-prod-01 (PBS Docker)|Configured on new PBS; scheduled execution not yet verified|
|TBD — Wave 8|Dedicated Plex full-appdata backup|`/mnt/user/appdata/plex`|NAS `backups` share — final path TBD|nas-prod-01|Not implemented|
|TBD — Wave 8|Synology ABB|nas-prod-01 `backups` share|Synology NAS|Synology|Not implemented|

**AI Observer:** `observer-prod-01` (VM 107) remained included in the November 5 successful cluster backup job on pve-prod-01. Its individual snapshot integrity and isolated full-VM recovery have not been independently verified. No separate observer rsync backup has been verified.

The dedicated Plex full-appdata backup and Synology ABB remain planned work, not confirmed operational backup systems.

---

## Ordering Rationale

Jobs are sequenced intentionally:

1. **02:00 — PBS** runs first while the homelab is at its quietest.
    
2. **03:00 — rsync** runs after the scheduled PBS start, reducing intended overlap.
    
3. **04:00–06:00 — Plex maintenance** includes the built-in database backup every three days.
    
4. **04:00 Saturday — PBS GC** is scheduled after the regular backup window.
    
5. **05:00 Saturday — PBS Verify** is scheduled after GC.
    
6. **Dedicated Plex backup (Wave 8)** — schedule TBD; must account for Plex maintenance, any service interruption, and other NAS backup jobs.
    
7. **Synology ABB (Wave 8)** — intended to run after source backups complete so it captures consistent recovery data.
    

Scheduling alone does not guarantee that jobs finish before the next job begins.

Following failure recurrence on October 31–November 2, PBS was migrated to Docker on Unraid. The November 5 scheduled backups succeeded on both nodes, and a separate CT 100 file-level restoration was verified. Continued reliability and full-guest recovery remain to be tested.

---

## PBS Backup

**Backup configuration:** Proxmox VE Datacenter → Backup  
**PBS deployment:** `proxmox-backup-server` container on `nas-prod-01` (Unraid)  
**Image:** `ayufan/proxmox-backup-server:latest` (unofficial; installed PBS UI version 4.2.0)  
**PBS endpoint:** `192.168.30.16:8007` (Services VLAN; Host networking may also expose 8007 on the NAS management IP)  
**UI:** `https://192.168.30.16:8007`  
**Proxmox storage ID:** `pbs-prod-01` (existing ID preserved)  
**PBS datastore:** `nas-backups`  
**Container datastore path:** `/backups`  
**NAS datastore path:** `/mnt/zfs-mirror/backups/pbs`  
**Storage:** `zfs-mirror/backups` ZFS dataset, not a separate PBS-specific dataset  
**Old PBS:** VM 103 on pve-prod-02 (`192.168.30.12`), shut down and retained as a temporary rollback artifact; **must not run concurrently against this datastore**.

Proxmox guests back up to the existing `pbs-prod-01` storage definition on both cluster nodes. The cluster-wide storage definition now points to the Docker-based PBS endpoint, uses a dedicated API token, and still targets `nas-backups`. The datastore is bind-mounted from the physical ZFS path; PBS no longer reads its datastore through the old VM's NFSv3 → Unraid `shfs`/FUSE path. The NAS's other NFS exports and share configuration were not changed.

### Container configuration and safeguards

- **Docker image:** `ayufan/proxmox-backup-server:latest` (unofficial third-party build, described by its maintainer as unmaintained; do not assume it follows official Proxmox releases).
- **Networking:** Host; PBS accessed by the PVE nodes at `192.168.30.16:8007`. Confirm intended access restrictions because the process may also bind to `192.168.10.10:8007`.
- **Privileges:** Unraid Docker template `Privileged` disabled.
- **Persistent mappings:** `/mnt/user/appdata/pbs/etc` → `/etc/proxmox-backup`, `/mnt/user/appdata/pbs/logs` → `/var/log/proxmox-backup`, `/mnt/user/appdata/pbs/lib` → `/var/lib/proxmox-backup`.
- **Datastore mapping:** `/mnt/zfs-mirror/backups/pbs` → `/backups` (read/write); `pbs` root owned by `34:34` with mode `755`, `.chunks` owned by `34:34` with mode `750` following the reuse permission check.
- **Timezone:** `America/Chicago`.
- **Extra Docker parameters:** `--restart unless-stopped --memory=2g --mount type=tmpfs,destination=/run --stop-signal=SIGHUP`.
- **Auto-start:** Enabled; actual start-after-ZFS-mount ordering has not been independently validated.
- **Backup data:** Existing groups/snapshots reused; 7 groups / 91 snapshots visible immediately after registration (point-in-time migration observation, not a current count).
- **Update management:** Via Unraid's container image update/recreation workflow, not an in-container `apt upgrade`. The November 5 live integration reported no image update available, but latest image is older than official PBS 4.2.8 documentation. Monitor image maintenance and security fixes.
- **Subscription:** No subscription is required for core PBS features. The GUI no-subscription warning is expected; do not modify distributed application code merely to remove it.

### Authentication

- Local emergency administrator: `admin@pbs`, default password changed and stored privately.
- Human SSO administrator: `gio@authentik`, Admin permission on `/`. The existing `OIDC - Proxmox` Authentik provider is still shared with both PVE nodes; the redirect URI was updated to `https://192.168.30.16:8007`. Shared OIDC client secret was rotated and fresh interactive SSO logins passed on PBS and both PVE nodes.
- Scheduled PVE backups: `pve-backup@pbs!pve-cluster` API token. Parent `pve-backup@pbs` and token both have `DatastoreBackup` permission for `/datastore/nas-backups`; all seven existing backup groups were reassigned to the token identity.
- Secrets, webhook URLs, password values, and OIDC client secret are intentionally excluded from documentation; store them in Bitwarden or the appropriate protected configuration.

### Maintenance and alerts

- **Prune:** daily; keep-last 3, keep-daily 7, keep-weekly 4, keep-monthly 3; hourly/yearly unset.
- **Garbage collection:** Saturday `04:00`.
- **Verification:** Saturday `05:00`, skip previously verified snapshots unless older than 30 days; UI showed 1 read thread and 4 verification threads.
- **Discord:** `Discord` webhook target test delivered to `#homelab-alerts` on November 5; `default-matcher` was edited to select warning/error severities. The matcher was not independently read back via CLI after that edit, and a real failure-event delivery test was not performed.
- **Capacity planning:** Operator preference ~500 GB soft PBS budget, not a quota. Avoid equating total ZFS dataset usage with PBS-exclusive storage.

### Verified Backup Job Configuration

The job settings were verified through live Proxmox inspection during the October incident and retained through the November 5 migration; the new storage endpoint and successful scheduled execution on both nodes were independently verified November 5.

|Setting|Value|
|---|---|
|Job ID|`backup-54015f90-26e5`|
|Enabled|Yes|
|Schedule|02:00|
|Backup mode|Snapshot|
|Target storage|`pbs-prod-01`|
|Guest selection|All guests|
|Excluded guest|VM 103 — pbs-prod-01|
|Notification mode|Proxmox notification system|
|Notes template|`{{guestname}}`|

The job is configured to run on both Proxmox nodes, with each node processing its applicable guests.

**VM 103 exclusion:** The retired PBS VM remains explicitly excluded from this job. It is stopped and preserved temporarily for fallback, but must not be started while Docker PBS is using the same datastore. The Docker configuration and ZFS datastore are not protected by this VM backup job.

### Verified Guest Selection

|VMID|Guest|Proxmox node|Included in job|
|---|---|---|---|
|100|dns-prod-01|pve-prod-01|Yes|
|101|docker-prod-01|pve-prod-01|Yes|
|102|dns-prod-02|pve-prod-02|Yes|
|103|pbs-prod-01|pve-prod-02|**No — explicitly excluded**|
|104|auth-prod-01|pve-prod-01|Yes|
|105|immich-prod-01|pve-prod-02|Yes|
|106|dragonwilds-server|pve-prod-01|Yes|
|107|observer-prod-01|pve-prod-01|Yes|

The guest selection establishes intended coverage. The November 5 02:00 scheduled job on pve-prod-01 completed `OK` at 02:05:50 CST (347 seconds); the pve-prod-02 job completed `OK` at 02:04:30 CST (265 seconds). This confirms node-level backup jobs, not individually verified integrity or full restorability for every guest.

The live job configuration inspection reported that VM 107's primary disk and EFI disk are selected for backup.

### Retention

**Configured and visually verified on the new PBS Docker instance, November 5:** daily prune with keep-last `3`, keep-daily `7`, keep-weekly `4`, keep-monthly `3`; keep-hourly and keep-yearly unset. The PVE storage definition retains `prune-backups keep-all=1` so PBS owns datastore retention. A successful scheduled prune/GC execution has not yet been checked on the new instance. The ~500 GB desired PBS storage budget is a soft planning threshold rather than an enforced quota.

### Monitoring

**PVE notification system:** Enabled for the scheduled job. On PBS, the new `Discord` webhook target delivered an explicit test notification into `#homelab-alerts`; its default matcher was manually configured for `warning` and `error` notifications. Actual warning/error job-event delivery remains untested, as does the exact saved matcher configuration via CLI.

**Healthchecks.io:** A dedicated PBS heartbeat has not been independently confirmed as configured.

---

## PBS Docker Migration — November 3–5, 2026

**Status:** Migration implemented; initial backup, integrity, and file-level restore validation passed. Full-guest restore and continued reliability remain unverified.

After the October 30 temporary proxy restart, scheduled backups failed again on October 31, November 1, and November 2 on both PVE nodes. The same PBS proxy process held a stale NFS `.lock` descriptor (`ESTALE`), proving recurrence of the immediate failure mechanism; the original invalidation trigger was not isolated.

With the user's explicit decision to forgo a new protective ZFS snapshot and accept potential loss of historical archives, VM 103 was gracefully stopped on pve-prod-02. The PBS server was freshly configured using the Unraid Community Apps template with image `ayufan/proxmox-backup-server:latest`. The old VM remains stopped. The existing ZFS datastore was registered with **Reuse existing datastore**; its root directory mode was changed from `700` to `755` after PBS rejected the ownership/permission check. No recursive ownership change or datastore wipe was performed.

The original `pbs-prod-01` PVE storage definition was preserved; the endpoint changed from `192.168.30.12` to `192.168.30.16`, and authentication changed from `root@pam` to the least-privilege token `pve-backup@pbs!pve-cluster`. Its TLS fingerprint was replaced with the new PBS certificate fingerprint; the token secret and certificate private material are not documented here. Proxmox's `server` field is fixed in the `pvesm set` API and was updated by editing the corresponding `server` line in cluster-wide `/etc/pve/storage.cfg`. Password/identity/fingerprint were updated through `pvesm set`. No duplicate storage ID was created.

**Verified November 5:**

- Manual CT 100 (`dns-prod-01`) snapshot at 00:36 CST: `TASK OK`, completed in approximately 4 seconds, reused approximately 79.7% of its file data against the previous manifest.
- PBS integrity verification of the new CT 100 snapshot: passed (user-confirmed).
- PVE 02:00 scheduled jobs: `OK` on both nodes, independently checked via live Proxmox task history.
- File-level recovery: `/etc/hostname` downloaded from CT 100 backup and contents checked by the operator. **No full VM/CT recovery test was performed.**
- Authentik SSO: fresh logins succeeded on Docker PBS and both PVE nodes after the shared provider secret was rotated.
- Discord webhook target: explicit test delivered; matcher setup was reported complete.
- Docker container remained `RUNNING` with auto-start enabled in subsequent live Unraid inspection.

**Remaining operational checks:** monitor later scheduled backups, verify that prune/GC/weekly verify jobs run normally, consider an isolated full guest restore, examine host-networking exposure on port `8007`, and plan secure backup/recovery of the new PBS appdata plus datastore. The third-party container image's maintenance lag relative to official PBS should be reviewed periodically.

---

## PBS Backup Incident — October 2026 (historical)

**Status as of October 30:** Immediate operational failure cleared temporarily; the same stale-handle problem recurred October 31–November 2 and the PBS deployment was replaced November 3–5.  
**Identified:** 2026-10-29 local / 2026-10-30 UTC  
**Last confirmed successful execution at time of this incident record:** October 30, 2026 at 02:00 CDT, on both PVE nodes  
**Affected systems:** pve-prod-01, pve-prod-02, pbs-prod-01, and the Unraid NFS datastore

### Failure history and recovery-point exposure

- The October 29 02:00 backup job failed for CT 100, VMs 101/104/106/107 on pve-prod-01 and CT 102/VM 105 on pve-prod-02. The clients reported `connection reset` and `HTTP/2.0 connection failed`.
- The latest 30 daily tasks queried on pve-prod-01 (September 30–October 29) showed `job errors`; recent tasks on pve-prod-02 also failed. These results do not establish the exact first failure date.
- Inspection of the PBS datastore through its web UI and direct ZFS path showed older snapshots, including August 12 backups with existing manifests and previously reported `Verify OK` status. October 29 failed attempts left empty snapshot directories without `index.json.blob`.
- Prior to the successful October 30 run, no complete snapshot manifest was found for VM 106 or VM 107. The read-only Homelab Proxmox integration returned zero archives for some guests even when older archives were visible through PBS; treat that as an API/visibility limitation, not proof of absent backups.

### Confirmed immediate failure mechanism

PBS's own `proxmox-backup-proxy` logs recorded `TASK ERROR: unable to get shared lock - ESTALE: Stale file handle` at the same moment backups failed. This was reproduced during a manual CT 100 backup at **2026-10-30 00:31:10 CDT**.

The proxy was PID `594`, started on August 11, and held file descriptor `21` for `/mnt/backups/pbs/.lock`. `readlink /proc/594/fd/21` showed `.lock (deleted)` and `stat -L /proc/594/fd/21` returned `Stale file handle`. Meanwhile, opening the current `.lock` pathname and acquiring new `flock` and POSIX `fcntl` shared locks succeeded. The old and new references reported the same inode number; the old descriptor itself was invalid.

This establishes a **stale, long-lived NFS file reference in the running PBS proxy** as the immediate mechanism behind the failed lock acquisition. It does **not** establish which upstream event invalidated that reference.

### NAS/NFS evidence and unresolved trigger

*This section records the former VM-based PBS access path. It is not the current Docker deployment.*

- Unraid pool `zfs-mirror` was `ONLINE`, with zero reported READ, WRITE, or CKSUM errors and no known data errors.
- PBS mounted `192.168.30.16:/mnt/user/backups` at `/mnt/backups` using **NFSv3** and systemd automount. `/mnt/user` uses Unraid `fuse.shfs`; `/mnt/zfs-mirror/backups` is the direct ZFS dataset.
- Unraid's share configuration placed `backups` only on `zfs-mirror`; the `backups`, `photos`, and `system` directories were found only under `/mnt/zfs-mirror`. `fuse_remember="330"` and **Permit exclusive shares = No** were confirmed.
- The underlying ZFS `.lock` file's timestamps dated to March 18, 2026; the `shfs` process had run since July 28. The original file-handle invalidation event was not proven. FUSE/NFS handle management remains a leading hypothesis, not a confirmed originating root cause.
- The PBS NFS client currently uses a `soft` mount and the Unraid NFS export has `async` enabled. These durability choices warrant future review; neither was changed during this incident.

### Corrective action completed — October 30

Following a check that PBS reported no active tasks, the operator manually executed:

```bash
systemctl restart proxmox-backup-proxy
```

The proxy restarted at **00:55:42 CDT** with new PID `1372698`; the former proxy and its stale file descriptor were closed. No PBS VM reboot, Unraid restart, NFS remount, export change, datastore move, or backup deletion was performed.

**Verification evidence:**

|Test|Observed result|
|---|---|
|Manual CT 100 backup, 00:57 CDT|`TASK OK`; new snapshot `ct/100/2026-10-30T05:57:23Z` visible in PBS|
|CT 100 integrity verification, 01:02 CDT|`TASK OK`; root archive and catalog checked with **0 errors**|
|Scheduled pve-prod-01 job, 02:00–02:22 CDT|`TASK OK`; CT 100 and VMs 101, 104, 106, 107 completed|
|Scheduled pve-prod-02 job, 02:00–02:05 CDT|`TASK OK`; CT 102 and VM 105 completed|

The two scheduled task results and guest completion messages were independently confirmed through the live Homelab Proxmox integration on October 30. The new backup snapshots for all guests have **not** all undergone separate integrity verification or restoration testing.

### Operational decision and remaining work (as of October 30; superseded)

*The monitoring-only decision below predates the November Docker migration. It is preserved for incident history.*

The operator chose to retain the existing NAS storage architecture and monitor the configured Discord backup alerts rather than introduce immediate NFS/FUSE changes. **Permit exclusive shares remains disabled**; no dedicated direct-ZFS NFS export was implemented. A FUSE-free PBS-specific export is an **unimplemented option for evaluation only**, not the current configuration or a guaranteed cure for NFS handle failures.

Still required:

1. Monitor subsequent scheduled runs on **both** PVE nodes and confirm Discord failure-alert delivery.
2. Verify the new PBS snapshots for remaining guests, especially VM 106 and VM 107, and confirm usable recovery points.
3. Plan and perform an isolated, non-destructive restoration test; protect production IPs, secrets, and tunnel identities.
4. Revisit NFS/FUSE and mount durability if the stale-handle problem recurs, without assuming FUSE caused the original invalidation.
5. Reconfirm current retention/prune behavior before relying on older recovery windows; avoid manual prune, GC, or datastore-file deletion as troubleshooting shortcuts.
6. Establish an independent recovery strategy for excluded PBS VM 103.

**Incident classification:** Backup execution operationally recovered, but not considered permanently resolved or end-to-end recovery-tested.

---

## AI Observer — Backup and Recovery

**Host:** observer-prod-01  
**IP:** 192.168.30.17  
**Proxmox VMID:** 107  
**Proxmox node:** pve-prod-01  
**Status:** October 30 VM backup task succeeded; archive integrity and restoration not yet verified

AI Observer hosts the read-only Proxmox MCP server and OpenAI Secure MCP Tunnel client.

Although the VM is included in the nightly PBS job, its recoverability has not been established.

### Backup Coverage

The Proxmox backup-job inspection confirmed that VM 107's virtual disks are included:

- `scsi0` — primary 20 GiB virtual disk.
    
- `efidisk0` — EFI disk.
    

When a complete VM-level backup succeeds, data stored on included VM disks is normally captured as part of that backup.

This means the Git-excluded application configuration and systemd files located on the primary VM disk are not inherently excluded just because they live outside `/opt/stacks`.

However, that protection exists only when a usable VM backup has been created.

**Current finding:** VM 107's October 29 scheduled backup failed, but its October 30 scheduled backup completed successfully (`TASK OK`) on pve-prod-01. Separate PBS snapshot-level integrity verification, detailed archive inspection, and restoration testing are pending.

### Configuration and Credentials Requiring Recovery

|Path|Purpose|Recovery consideration|
|---|---|---|
|`/opt/stacks/proxmox-mcp/compose.yaml`|MCP Docker Compose configuration|Version-controlled in `homelab-infra`|
|`/opt/stacks/proxmox-mcp/.env`|Proxmox API connection, MCP authentication, and settings|Git-excluded secret configuration|
|`/etc/tunnel-client/config.yaml`|Tunnel ID, MCP routing, credential-file references|Host-level configuration, not in Git|
|`/etc/tunnel-client/openai-runtime.key`|OpenAI runtime API key|Protected secret|
|`/etc/tunnel-client/mcp-authorization`|MCP bearer authorization header|Protected secret|
|`/etc/systemd/system/tunnel-client.service`|Persistent systemd service configuration|Host-level service unit|
|`/opt/tunnel-client/`|Tunnel-client binaries and supporting files|Installed binaries; reproducible from verified official releases|

### Current Protection

- MCP Compose configuration exists in the private `homelab-infra` repository.
    
- Sensitive credentials were retained separately through the established Bitwarden workflow.
    
- The scheduled PBS job includes VM 107.
    
- VM 107 completed a scheduled PBS backup on October 30; the resulting snapshot has not yet been independently integrity-verified or restore-tested.
    
- No independent observer file-level backup job has been verified.
    
- No observer restoration test has been completed.
    

Existing rsync jobs on docker-prod-01, auth-prod-01, and immich-prod-01 do not automatically protect observer-prod-01.

### Recovery Design Requirements

The preferred recovery design is:

1. **VM-level recovery:** Maintain successful PBS backups of observer-prod-01.
    
2. **Source-controlled configuration:** Recover committed MCP Compose configuration from `homelab-infra`.
    
3. **Credential recovery:** Preserve required credentials in Bitwarden or another approved secure recovery location.
    
4. **Host-level configuration recovery:** Ensure the tunnel configuration, systemd unit, and permissions can be restored.
    
5. **Verified restoration:** Test the recovery procedure without unexpectedly overwriting the production observer VM.
    

A successful full-VM restoration should preserve the included virtual-disk contents, including protected configuration files, assuming the backup was complete and the data was present when the backup was taken.

A fresh installation using Git alone will not restore the complete observer environment.

### Additional Backup Decision

A separate encrypted configuration backup may be considered for the observer if VM-level backups and Bitwarden recovery are insufficient for the desired recovery objectives.

This would need to account for:

- Encryption at rest and during transfer.
    
- Secure key storage and recovery.
    
- Restricted access to API tokens and bearer credentials.
    
- Backup scheduling and retention.
    
- Restore testing.
    
- Avoiding accidental inclusion of secrets in Git.
    

**Status:** Not implemented. Do not assume a separate observer configuration archive currently exists.

### Verification Required

Before marking AI Observer backups complete:

- Continue monitoring PBS for recurrence of the stale-handle failure; successful scheduled execution was restored October 30.
    
- Confirmed: VM 107 scheduled backup task completed successfully October 30.
    
- Still required: inspect the VM 107 snapshot in PBS and independently verify its integrity.
    
- Establish and test a safe recovery procedure.
    
- Confirm that Bitwarden contains the credentials necessary to rebuild the integration.
    

See `services/infra/ai-observer.md`.

The detailed recovery procedure belongs in `runbooks/ai-observer-operations.md`.

---

## rsync — Stacks & Appdata

**Script location:** `/usr/local/bin/backup-appdata.sh` on each participating Docker host  
**Log location:** `/var/log/backup-appdata.log`  
**SSH key:** `/root/.ssh/backup_rsa`  
**Scheduled via:** Root crontab  
**Run manually:** `sudo /usr/local/bin/backup-appdata.sh`  
**Transport:** SSH and rsync to nas-prod-01 at `192.168.30.16`

### Participating Hosts

- docker-prod-01
    
- auth-prod-01
    
- immich-prod-01
    

observer-prod-01 is **not** included in the currently documented rsync backup deployment.

### What Is Backed Up

- `/opt/stacks` — Compose configuration and Git-excluded `.env` files.
    
- `/opt/appdata` — Container configuration and persistent application data stored under that path.
    

This is a separate file-level backup system from PBS.

**Important distinction:** PBS VM backups can capture files inside included virtual disks. The rsync jobs provide separate file-level recovery copies and operational flexibility rather than compensating for an inherent inability of PBS to capture guest files.

### Behavior

The documented rsync implementation is a rolling mirror using `--delete`.

The destination is synchronized with the source.

This does not provide independent historical file versions.

If a file was deleted or corrupted before the latest successful rsync operation, the backup mirror may reflect that state.

Versioned recovery through Synology ABB remains planned for Phase 5 Wave 8.

### Healthchecks.io

The existing backup monitoring uses Healthchecks.io.

|Host|Check|
|---|---|
|docker-prod-01|`backup-appdata-docker-prod-01`|
|auth-prod-01|`backup-appdata-auth-prod-01`|
|immich-prod-01|`backup-appdata-immich-prod-01`|

The raw Healthchecks ping URLs are not reproduced here because they contain credential-like identifiers that can be used to send heartbeat signals.

Retain those URLs securely in the service configuration and a suitable credential manager rather than embedding them in general documentation.

**Verification:** The current heartbeat states have not been independently checked during this documentation update.

The existing URLs should be reviewed and rotated if their exposure in earlier documentation is considered inappropriate.

---

## Plex — Built-in Database Backup

**Host:** nas-prod-01  
**Managed by:** Plex Media Server → Settings → Scheduled Tasks  
**Schedule:** Every three days during the 04:00–06:00 maintenance window  
**Status:** Last verified operational on 2026-10-26

### Source and Backup Location

`/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases/`

### What Is Backed Up

- Plex library database.
    
- Plex library blobs database.
    

### Verified Backup Dates

- 2026-10-16
    
- 2026-10-19
    
- 2026-10-22
    
- 2026-10-25
    

Both databases had dated backup copies for all four dates, created around 04:03–04:04.

### Verification Command

```
find "/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases" \
  -maxdepth 1 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM | %s bytes | %f\n' \
  | sort -r | head -30
```

Look for recent date-suffixed copies of:

- `com.plexapp.plugins.library.db`
    
- `com.plexapp.plugins.library.blobs.db`
    

### Limitations

- Backups are stored within the same Plex appdata directory as the live databases.
    
- They do not provide an independent complete copy of Plex application data.
    
- Backup file creation was verified; integrity testing and restoration remain unverified.
    
- Retention settings were not independently verified.
    
- The latest backup dates have not been rechecked since October 26.
    

---

## Plex — Dedicated Full-Appdata Backup — Pending (Wave 8)

A dedicated custom backup script for Plex is planned under Phase 5 Wave 8.

**Planned source:** `/mnt/user/appdata/plex`  
**Planned destination:** NAS `backups` share; final subdirectory TBD  
**Backup mechanism:** Custom script on Unraid  
**Schedule:** TBD  
**Retention:** TBD  
**Status:** Not implemented as of the October 26, 2026 verification

The original Plex service documentation described a stop–backup–restart script writing to `/mnt/user/backups/plex/db/`.

Investigation on 2026-10-26 found:

- The documented destination directory did not exist.
    
- No Plex-specific root cron entry was found.
    
- The Unraid User Scripts directory was absent.
    
- No dedicated Plex backup script was located in the common configuration and executable directories searched.
    

Existing Docker-host rsync jobs do not cover Plex running directly on nas-prod-01.

### Implementation Requirements — Wave 8

- Safe handling of the running Plex container and its SQLite databases.
    
- Backup destination and archive format.
    
- Automated scheduling.
    
- Retention and historical recovery points.
    
- Logging, error handling, and notification.
    
- Backup integrity and restore testing.
    
- Coordination with ZFS snapshots and planned Synology ABB.
    

See `services/media/plex.md` and `phases/phase-5-hardening.md`.

---

## NAS Backup Share Structure

Documented structure before the dedicated Plex backup and Synology ABB are implemented:

```
/mnt/user/backups/
├── pbs/                    ← PBS datastore (uid 34 owned — do not modify manually)
├── appdata/                ← rsync appdata backups
│   ├── docker-prod-01/
│   ├── auth-prod-01/
│   └── immich-prod-01/
└── stacks/                 ← rsync stacks backups
    ├── docker-prod-01/
    ├── auth-prod-01/
    └── immich-prod-01/
```

The overall appdata/stacks share contents were not independently rechecked during this update. The PBS datastore directory, including `.chunks`, `.lock`, `vm`, and `ct`, was inspected during the October 2026 incident.

No observer-prod-01 rsync directories are shown because a separate observer rsync job has not been verified.

### PBS Datastore Safety

The active Docker bind mount is `/mnt/zfs-mirror/backups/pbs` → `/backups`; the `/mnt/user/backups/pbs` alias is not used by the PBS container.

The PBS datastore must remain isolated within its dedicated `pbs/` subdirectory.

Do not manually place unrelated files inside the PBS datastore.

Incorrect ownership or permissions within the datastore can interfere with backup operations, verification, or garbage collection.

PBS datastore content should be managed through supported PBS operations.

---

## Backup Health Verification

A backup is considered healthy only when the relevant execution and recovery evidence has been checked.

### Proxmox Backup Jobs

From Proxmox:

1. Open `https://192.168.10.11:8006`.
    
2. Navigate to **Datacenter → Backup**.
    
3. Confirm the expected job is enabled.
    
4. Review the most recent task history on both nodes.
    
5. Inspect failed tasks and their logs.
    
6. Confirm that the expected guests were processed.
    

**Latest verified state:** The November 5 02:00 scheduled jobs completed successfully on both nodes using Docker PBS. Targeted CT 100 integrity verification and file-level retrieval passed; a full-guest restore test remains open.

### PBS Archives

From PBS:

1. Open `https://192.168.30.16:8007`.
    
2. Navigate to **Datastore → nas-backups → Content**.
    
3. Confirm that expected VM and CT archives exist.
    
4. Check archive timestamps.
    
5. Review verification status where available.
    

A backup archive's presence is necessary but not sufficient proof of recoverability.

Perform an appropriate safe restore test to verify recovery.

### rsync Logs

On each participating Docker host:

```
sudo tail -n 50 /var/log/backup-appdata.log
```

Look for the most recent successful completion marker.

If a failure is recorded, inspect the related exit status and error messages.

### Healthchecks.io

Review the three documented checks:

- `backup-appdata-docker-prod-01`
    
- `backup-appdata-auth-prod-01`
    
- `backup-appdata-immich-prod-01`
    

Confirm that each has received a recent successful heartbeat.

A green heartbeat should be correlated with the host backup log rather than used as the only evidence of backup integrity.

### Plex Database Backups

Use the documented verification command to confirm recent dated database copies.

This confirms file creation and schedule consistency, not database integrity or recoverability.

### AI Observer

Verify:

1. VM 107 remains included in the backup selection.
    
2. Its scheduled backup task completes successfully.
    
3. A recent archive exists in PBS.
    
4. Its backup is verified or safely restored during a controlled test.
    
5. Required credentials are recoverable from approved secure storage.
    
6. The restored integration can successfully perform a read-only Proxmox query.
    

Until these checks pass, observer recoverability remains unverified.

---

## Pending

### High Priority — PBS Reliability and Recovery Validation

- **Completed:** Diagnose the former PBS VM's stale NFS `.lock` descriptor and observe recurrence October 31–November 2 after a temporary October 30 proxy restart.
- **Completed:** Migrate to Docker PBS with a direct-ZFS bind mount, retain the old VM stopped, register the existing datastore, and reconnect PVE without changing its storage ID.
- **Completed:** CT 100 backup and integrity verification; November 5 02:00 scheduled backup jobs on both Proxmox hosts; file-level recovery of `/etc/hostname`.
- **Completed:** Test shared Authentik SSO after provider-secret rotation on PBS and both PVE hosts; test Discord webhook target delivery.
- **Open:** Monitor subsequent backup cycles; verify first actual prune, GC, and scheduled verify runs on the new deployment.
- **Open:** Perform a safe isolated full VM/CT restore (and AI Observer-specific recovery testing when appropriate).
- **Open:** Verify Docker Host-network port 8007 access policy and PBS start-after-ZFS-mount ordering. Review update strategy for the unofficial image.
- **Open:** Define the desired protection/recovery approach for the PBS container's persisted config and the ZFS datastore; the operator intentionally declined a migration-time snapshot and accepts rebuilding a datastore if necessary.

---

### AI Observer Recovery

- Confirm the October 30 successful VM 107 backup is present in PBS, then perform snapshot integrity verification.
    
- Establish a tested recovery procedure for AI Observer.
    
- Verify secure recovery of Git-excluded configuration and tunnel credentials.
    
- Decide whether a separate encrypted observer configuration backup is necessary.
    
- Document the completed process in `runbooks/ai-observer-operations.md`.
    

### Existing Planned Work

- PBS Healthchecks.io heartbeat — not yet verified as configured.
    
- Dedicated Plex full-appdata backup — Phase 5 Wave 8.
    
- Plex backup retention, integrity checks, and restore procedure.
    
- Synology ABB — Phase 5 Wave 8, versioned backup of the `backups` share.
    

---

## Related Documentation

- `runbooks/backup-restore.md` — VM, LXC, Docker, and Plex restoration procedures.
    
- `runbooks/ai-observer-operations.md` — AI Observer troubleshooting and recovery
    
- `services/infra/ai-observer.md` — Observer architecture, credentials, and backup requirements.
    
- `architecture/compute/pve-prod-01.md` — Primary Proxmox node and observer VM.
    
- `architecture/compute/pve-prod-02.md` — Secondary Proxmox node; retired/stopped PBS VM 103.
    
- `runbooks/git-workflow.md` — Version-controlled infrastructure configuration.
    
- `services/media/plex.md` — Plex deployment and backup requirements.
    
- `phases/phase-5-hardening.md` — Planned operational hardening work.
    

---

## Verification Summary — 2026-10-30 (historical)

|Item|Result|
|---|---|
|PBS scheduled job configured and enabled|Verified|
|VM 107 included with primary/EFI disks|Verified|
|October 29 backup on pve-prod-01 and pve-prod-02|Failed|
|Immediate failure mechanism|Stale PBS proxy NFS `.lock` descriptor; `ESTALE` reproduced|
|Upstream trigger for stale handle|Not established; Unraid FUSE/NFS interaction suspected|
|PBS proxy restart, October 30 00:55 CDT|Successful; new process verified|
|CT 100 manual backup, October 30 00:57 CDT|Successful, visible in PBS|
|CT 100 integrity verification, October 30 01:02 CDT|Passed; 0 errors|
|October 30 scheduled pve-prod-01 job|`TASK OK` for guests 100, 101, 104, 106, 107|
|October 30 scheduled pve-prod-02 job|`TASK OK` for guests 102, 105|
|Integrity verification for all other October 30 snapshots|Not completed/independently confirmed|
|Safe restoration test|Not performed|
|Discord notification configuration|User reports configured; actual delivery not independently verified|
|Unraid exclusive shares / direct-ZFS export|Not enabled/implemented; existing architecture unchanged|
|Existing Docker rsync latest executions|Not reverified|
|Plex built-in database backups|Last verified 2026-10-26|
|Dedicated Plex full-appdata backup / Synology ABB|Pending|

**Historical note:** This October 30 assessment was superseded after failure recurrence and the November Docker migration. See the new migration section above.

### Verification Summary — 2026-11-05

| Item | Result |
|---|---|
| New PBS Docker container | Running on nas-prod-01; image `ayufan/proxmox-backup-server:latest`, PBS UI 4.2.0 |
| Old PBS VM 103 | Stopped; retained temporarily, not to run concurrently |
| Datastore reuse | Existing 7 groups/91 snapshots visible immediately after registration |
| Proxmox PBS storage | `pbs-prod-01` preserved and available from both nodes at `192.168.30.16` |
| Manual CT 100 backup and targeted integrity check | Successful |
| November 5 02:00 scheduled jobs on both nodes | `OK` |
| CT 100 `/etc/hostname` file-level restore | Successful, contents inspected |
| Full-guest isolated restore | Not performed |
| Authentik OIDC secret rotation and SSO | Fresh PBS and both PVE logins successful |
| Discord target | Test message delivered; actual warning/error routing not failure-tested |
| New PBS prune, GC, and scheduled verify execution | Configured; first executions not yet confirmed |
| NFS/FUSE stale-handle recurrence under Docker | Not observed in initial backup run; continued monitoring needed |

