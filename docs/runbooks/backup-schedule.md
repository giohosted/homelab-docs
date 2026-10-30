# Backup Schedule

**Last Updated:** 2026-10-29  
**Status:** Active backup incident — scheduled Proxmox backups failing

Reference document for backup jobs across Homelab v3. Lists what is configured, when jobs run, where backups are stored, and how their success is verified.

**Important:** A configured or enabled backup job does not guarantee that usable recovery points exist. Backup execution, archive availability, integrity, and restoration must be verified separately.

---

## Schedule Overview

|Time|Job|Source|Destination|Host|Status|
|---|---|---|---|---|---|
|02:00 daily|PBS — VM & CT backup|Proxmox VMs and LXCs, excluding VM 103|nas-prod-01 `/mnt/user/backups/pbs`|pve-prod-01 and pve-prod-02|**Failing — investigation required**|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|docker-prod-01|Documented operational; latest execution not reverified|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|auth-prod-01|Documented operational; latest execution not reverified|
|03:00 daily|rsync — stacks + appdata|`/opt/stacks` + `/opt/appdata`|nas-prod-01 `/mnt/user/backups/`|immich-prod-01|Documented operational; latest execution not reverified|
|Every 3 days, ~04:03|Plex built-in DB backup|Plex library and blobs databases|`/mnt/user/appdata/plex`|nas-prod-01|Last verified 2026-10-26|
|04:00 Saturday|PBS — Garbage Collection|nas-backups datastore|—|pbs-prod-01|Configured historically; latest execution not reverified|
|05:00 Saturday|PBS — Verify|nas-backups datastore|—|pbs-prod-01|Configured historically; latest execution not reverified|
|TBD — Wave 8|Dedicated Plex full-appdata backup|`/mnt/user/appdata/plex`|NAS `backups` share — final path TBD|nas-prod-01|Not implemented|
|TBD — Wave 8|Synology ABB|nas-prod-01 `backups` share|Synology NAS|Synology|Not implemented|

**AI Observer:** `observer-prod-01` (VM 107) is included in the configured PBS job, but no successful backup archive was visible during verification on October 30, 2026 UTC. It does not have a separately verified rsync backup.

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

The current PBS failures must be resolved before assuming the planned ordering is producing healthy recovery points.

---

## PBS Backup

**Backup configuration:** Proxmox VE Datacenter → Backup  
**Backup destination:** Proxmox Backup Server on pbs-prod-01  
**PBS host:** pbs-prod-01 (192.168.30.12)  
**PBS version:** 4.1.4 (last documented)  
**UI:** `https://192.168.30.12:8007`  
**Proxmox storage ID:** `pbs-prod-01`  
**PBS datastore:** `nas-backups`  
**PBS datastore mount:** `/mnt/backups/pbs`  
**NAS destination:** `/mnt/user/backups/pbs`

The Proxmox backup job executes on the Proxmox nodes and sends VM/LXC backup data to PBS.

PBS stores the resulting archives in its configured datastore.

### Verified Backup Job Configuration

The following settings were verified through the live Homelab Proxmox integration on October 30, 2026 UTC.

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

**VM 103 exclusion:** The PBS VM is explicitly excluded from this job. Its recovery strategy must be evaluated separately; it is not protected by this scheduled job.

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

This confirms intended backup coverage, not successful backup execution.

The live job configuration inspection reported that VM 107's primary disk and EFI disk are selected for backup.

### Retention

Previously documented PBS retention settings:

- Keep Last: 3
    
- Keep Daily: 7
    
- Keep Weekly: 4
    
- Keep Monthly: 3
    

**Verification status:** These retention settings were preserved from the existing backup documentation and were not independently reconfirmed during this update.

Confirm the effective retention policy in Proxmox/PBS before relying on the documented recovery window.

### Monitoring

**Proxmox notification system:** Enabled for the scheduled backup job.

The October 29 task logs on both nodes reported notifications being sent through the `Discord` target.

This verifies that the backup jobs invoked the configured notification mechanism; it does not independently confirm message delivery or user acknowledgment.

**Healthchecks.io:** A dedicated PBS backup heartbeat was not configured according to the existing documentation.

The latest monitoring configuration has not been independently rechecked.

---

## PBS Backup Incident — October 2026

**Status:** Open — unresolved  
**Date identified:** 2026-10-29 local / 2026-10-30 UTC  
**Affected systems:** Both Proxmox nodes  
**Impact:** Scheduled VM/LXC backups to PBS are failing.

### Findings

Live Proxmox task history showed repeated scheduled backup failures on both nodes.

On `pve-prod-01`, the latest 30 queried daily backup tasks, covering September 30 through October 29, 2026, reported `job errors`.

On `pve-prod-02`, the recent queried backup tasks also reported `job errors`.

These findings establish repeated failures. They do not establish the date of the last successful backup or the initial cause of the problem.

### October 29 — pve-prod-01

The backup job started at approximately 02:00 local time.

|Guest|Result|
|---|---|
|dns-prod-01 (100)|Failed|
|docker-prod-01 (101)|Failed|
|auth-prod-01 (104)|Failed|
|dragonwilds-server (106)|Failed|
|observer-prod-01 (107)|Failed|

Representative errors:

```
HTTP/2.0 connection failed
Error: connection reset
backup register image failed: command error: connection reset
Backup job finished with errors
```

The observer's backup log specifically confirmed that VM 107 was selected and its backup was attempted.

Its backup did not complete.

### October 29 — pve-prod-02

The backup job also started at approximately 02:00 local time.

|Guest|Result|
|---|---|
|dns-prod-02 (102)|Failed|
|immich-prod-01 (105)|Failed|

The task log reported:

```
Couldn't download previous manifest - connection reset
HTTP/2.0 connection failed
Error: connection reset
backup register image failed: command error: connection reset
Backup job finished with errors
```

### Confirmed Scope

Both Proxmox nodes experienced errors while attempting to send backup data to PBS at `192.168.30.12:8007`.

The similar errors across nodes suggest that a shared dependency should be investigated.

Potential areas to investigate include:

- PBS service health.
    
- PBS datastore and NAS storage availability.
    
- Network connectivity between Proxmox nodes and PBS.
    
- PBS logs and resource availability.
    
- TLS, session, or protocol-level connection failures.
    

These are investigation targets, not confirmed root causes.

### Recovery-Point Visibility

The live Proxmox integration returned no visible backup archives when queried for:

- VM 107 — observer-prod-01
    
- VM 101 — docker-prod-01
    
- VM 105 — immich-prod-01
    

An additional storage-content query also returned no visible backup archives for the selected guests.

**Important:** The absence of archives in these queries does not conclusively prove that all historical PBS backup data is missing.

The datastore must be inspected directly through PBS to determine whether historical backups exist and are accessible.

### Immediate Follow-Up

Before declaring PBS healthy:

1. Inspect PBS service status and recent logs.
    
2. Verify the datastore is mounted, accessible, and healthy.
    
3. Investigate the connection resets.
    
4. Confirm the backup job completes successfully on both nodes.
    
5. Verify recent backup archives are visible in PBS.
    
6. Validate backup integrity and perform an appropriate safe restoration test.
    
7. Confirm notification delivery and backup failure alerting.
    

Follow the established one-step-at-a-time troubleshooting workflow.

Do not change PBS configuration, delete datastore contents, restart services, or trigger potentially disruptive recovery operations without reviewing their impact and obtaining authorization.

---

## AI Observer — Backup and Recovery

**Host:** observer-prod-01  
**IP:** 192.168.30.17  
**Proxmox VMID:** 107  
**Proxmox node:** pve-prod-01  
**Status:** VM backup configured, successful backup not verified

AI Observer hosts the read-only Proxmox MCP server and OpenAI Secure MCP Tunnel client.

Although the VM is included in the nightly PBS job, its recoverability has not been established.

### Backup Coverage

The Proxmox backup-job inspection confirmed that VM 107's virtual disks are included:

- `scsi0` — primary 20 GiB virtual disk.
    
- `efidisk0` — EFI disk.
    

When a complete VM-level backup succeeds, data stored on included VM disks is normally captured as part of that backup.

This means the Git-excluded application configuration and systemd files located on the primary VM disk are not inherently excluded just because they live outside `/opt/stacks`.

However, that protection exists only when a usable VM backup has been created.

**Current finding:** VM 107's October 29 scheduled backup failed, and no successful archive was visible through the live integration.

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
    
- No successful VM 107 backup archive has been verified.
    
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

- Resolve the PBS backup incident.
    
- Confirm a successful VM 107 backup.
    
- Verify the archive is visible in PBS.
    
- Validate the backup's integrity.
    
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

The directory structure reflects the previously documented configuration. Its current contents were not independently inspected during this update.

No observer-prod-01 rsync directories are shown because a separate observer rsync job has not been verified.

### PBS Datastore Safety

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
    

**Current state:** Failed on both nodes. Do not treat the configured schedule as successful protection.

### PBS Archives

From PBS:

1. Open `https://192.168.30.12:8007`.
    
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

### High Priority — PBS Backup Failure

- Investigate recurring connection resets between Proxmox nodes and PBS.
    
- Verify PBS and datastore health.
    
- Restore successful nightly backup execution.
    
- Confirm available historical recovery points.
    
- Verify recent backups for production VMs and LXCs.
    
- Establish a safe restoration test.
    
- Validate that failure notifications are delivered and actionable.
    

### AI Observer Recovery

- Confirm a successful PBS archive for VM 107.
    
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
    
- `architecture/compute/pve-prod-02.md` — Secondary Proxmox node and PBS VM.
    
- `runbooks/git-workflow.md` — Version-controlled infrastructure configuration.
    
- `services/media/plex.md` — Plex deployment and backup requirements.
    
- `phases/phase-5-hardening.md` — Planned operational hardening work.
    

---

## Verification Summary — 2026-10-30 UTC

|Item|Result|
|---|---|
|PBS scheduled job exists|Verified|
|PBS job enabled|Verified|
|VM 107 included in job|Verified|
|VM 107 disks selected|Verified|
|October 29 backup on pve-prod-01|Failed|
|October 29 backup on pve-prod-02|Failed|
|VM 107 successful archive|Not visible through live integration|
|PBS root cause|Unresolved|
|Existing Docker rsync latest executions|Not reverified|
|Plex built-in database backups|Last verified 2026-10-26|
|Dedicated Plex full-appdata backup|Pending|
|Synology ABB|Pending|
|AI Observer restoration test|Not performed|

**Do not mark PBS backup operations or AI Observer backup recovery as healthy until the required investigation and verification have been completed.**