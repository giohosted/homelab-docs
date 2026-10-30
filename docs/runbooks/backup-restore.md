# Backup & Restore Runbook

**Last Updated:** 2026-10-29  
**Status:** Active PBS backup incident — recovery-point availability requires verification

Procedures for restoring Homelab v3 infrastructure following host failure, data loss, corruption, or application-level problems.

See `runbooks/backup-schedule.md` for backup schedules, destinations, retention, verification status, and known failures.

---

## Overview

Homelab v3 uses several complementary backup mechanisms:

- Proxmox Backup Server (PBS) for VM and LXC backups.
    
- rsync for Docker Compose stacks and persistent application data.
    
- Plex built-in database backups.
    
- Planned dedicated Plex full-appdata backups.
    
- Planned Synology Active Backup for Business (ABB) for versioned backup storage.
    

These mechanisms provide different recovery capabilities and must not be treated as interchangeable.

A scheduled backup job does not guarantee that a usable recovery point exists.

Before performing any restoration:

1. Identify the affected service and failure scope.
    
2. Determine whether restoration is necessary or whether a smaller repair is appropriate.
    
3. Verify that the intended backup exists and is accessible.
    
4. Confirm the backup is suitable for the required recovery.
    
5. Assess the risk of overwriting existing data or configurations.
    
6. Plan the restoration and any required service interruption.
    
7. Perform the approved recovery operation.
    
8. Verify the restored service and its dependencies.
    

**Important:** Restoration procedures can overwrite or replace production data. Follow the established one-step-at-a-time workflow and obtain authorization before consequential changes.

---

## Backup Types

|Type|Tool|What It Covers|Location|Status|
|---|---|---|---|---|
|VM & CT backups|PBS|Selected Proxmox VM/LXC disks and configurations|nas-prod-01 `/mnt/user/backups/pbs`|Scheduled jobs failing|
|Stacks & appdata|rsync|Docker Compose files, `.env` files, and persistent appdata on participating Docker hosts|nas-prod-01 `/mnt/user/backups/stacks` and `/mnt/user/backups/appdata`|Documented operational; latest runs not reverified|
|Plex built-in DB backup|Plex Scheduled Tasks|Plex library and blobs databases only|nas-prod-01 `/mnt/user/appdata/plex`|Last verified 2026-10-26|
|Plex full-appdata backup|Custom script planned|Complete Plex persistent appdata|NAS `backups` share — destination TBD|Pending|
|Versioned backup|Synology ABB planned|NAS `backups` share with historical recovery points|Synology NAS|Pending|

### Scope Limitations

PBS backs up the virtual disks and other resources included in a guest's backup configuration. It does not necessarily include external NAS shares, independently mounted remote filesystems, or guest disks explicitly excluded from the backup.

Docker rsync jobs cover the documented participating hosts:

- docker-prod-01
    
- auth-prod-01
    
- immich-prod-01
    

They do not automatically protect observer-prod-01 or Plex appdata running directly on Unraid.

The dedicated Plex full-appdata backup and Synology ABB remain pending Phase 5 Wave 8 implementation.

---

## Current PBS Backup Incident

**Status:** Open — unresolved  
**Identified:** October 29–30, 2026  
**Affected:** pve-prod-01 and pve-prod-02

Live Proxmox investigation confirmed that scheduled backups were failing on both nodes.

Representative errors:

```
HTTP/2.0 connection failed
Error: connection reset
backup register image failed: command error: connection reset
Backup job finished with errors
```

The October 29 backup attempt failed for all guests included in that night's tasks.

Repeated failures were also visible in recent backup-task history.

### Recovery Impact

The scheduled PBS job remains enabled and correctly configured to select its intended guests.

However, successful backup execution and available recovery points have not been established.

The live Proxmox integration returned no visible archives for selected guests, including:

- VM 101 — docker-prod-01
    
- VM 105 — immich-prod-01
    
- VM 107 — observer-prod-01
    

This does not conclusively establish that historical archives are absent. Direct inspection of the PBS datastore is still required.

**Do not assume that a VM or LXC can be restored merely because it is included in the scheduled backup job.**

The underlying connection-reset issue will be investigated separately.

See `runbooks/backup-schedule.md` for the verified task results and investigation requirements.

---

## Restore — VM or LXC from PBS

Use this procedure when a VM or LXC has been lost, corrupted, or must be restored to an earlier recovery point.

**Status:** General restore workflow documented; current backup availability and end-to-end restoration have not been verified.

### Backup Infrastructure

|Component|Value|
|---|---|
|Proxmox storage ID|`pbs-prod-01`|
|PBS host|`192.168.30.12`|
|PBS datastore|`nas-backups`|
|PBS datastore mount|`/mnt/backups/pbs`|
|NAS destination|`/mnt/user/backups/pbs`|
|Primary Proxmox host|`192.168.10.11`|
|Secondary Proxmox host|`192.168.10.12`|

### Step 1 — Assess the Guest

Before restoring:

- Identify the affected VMID and hostname.
    
- Confirm its current state.
    
- Determine whether its virtual disks are intact.
    
- Identify any external storage dependencies.
    
- Determine whether an in-place repair is possible.
    
- Confirm whether restoration would overwrite current data.
    

Avoid replacing a VM when the problem is limited to one repairable application or configuration file.

### Step 2 — Verify the Backup Exists

Access Proxmox:

- pve-prod-01: `https://192.168.10.11:8006`
    
- pve-prod-02: `https://192.168.10.12:8006`
    

From the Proxmox interface:

1. Select a node with access to the `pbs-prod-01` storage.
    
2. Select the `pbs-prod-01` storage entry.
    
3. Open its backup-content view.
    
4. Locate the intended guest.
    
5. Review available backup timestamps.
    

Alternatively, inspect PBS directly:

`https://192.168.30.12:8007`

Navigate to the `nas-backups` datastore and review its backup content.

**If no usable archive is available, stop.**

Do not attempt to restore from an assumed or nonexistent backup.

### Step 3 — Validate the Recovery Point

Before proceeding:

- Confirm the correct VMID or CTID.
    
- Confirm the backup date and time.
    
- Check whether the archive is accessible.
    
- Review any available verification results.
    
- Confirm the required disks and configuration are included.
    
- Check whether relevant application data resides outside the VM backup.
    
- Determine whether the backup predates the failure or corruption.
    

A backup's presence alone does not guarantee its integrity.

For consequential recovery, prefer an archive that has passed verification and, where practical, has been tested through a safe restoration.

### Step 4 — Plan the Restore

Decide whether the operation is:

- **Test restoration:** Restore to an isolated environment or different VMID.
    
- **Production replacement:** Restore the original workload after confirming the existing instance will not conflict.
    

Confirm:

- Target Proxmox node.
    
- Target storage.
    
- Available capacity.
    
- Intended VMID or CTID.
    
- Networking and VLAN requirements.
    
- Whether the original guest still exists.
    
- Whether the restored guest may start automatically.
    
- Potential application and database implications.
    

**Do not enable automatic startup for a test restoration when the restored guest could conflict with an active production service.**

This is especially important for services using static IP addresses, authentication credentials, cluster membership, or externally connected tunnels.

### Step 5 — Restore

After selecting the verified backup:

1. Select **Restore**.
    
2. Confirm the target node.
    
3. Select the target storage.
    
4. Confirm the VMID or CTID.
    
5. Review any overwrite warnings.
    
6. Confirm startup behavior.
    
7. Start the approved restoration.
    
8. Monitor the task log until completion.
    

Do not overwrite the existing production guest without an approved recovery plan.

### Step 6 — Verify

After restoring:

- Confirm the VM or LXC boots.
    
- Verify networking and storage.
    
- Check application services.
    
- Verify access to required dependencies.
    
- Review relevant service logs.
    
- Confirm user-facing functionality.
    
- Confirm that the restored service does not conflict with another active instance.
    

Record the archive used, recovery actions, and verification results.

A successful Proxmox restore task alone does not establish application recovery.

### Retention

Previously documented PBS retention settings:

- Keep Last: 3
    
- Keep Daily: 7
    
- Keep Weekly: 4
    
- Keep Monthly: 3
    

These settings were not independently reconfirmed during the October 2026 investigation.

Effective retention depends on actual backup creation, pruning, datastore state, and the current retention configuration.

Do not infer available recovery dates from the retention policy alone.

### PBS VM Exclusion

**VM 103 — pbs-prod-01 is explicitly excluded from the scheduled PBS backup job.**

Its operating-system recovery and PBS datastore recovery requirements must be considered separately.

The existing backup job does not provide a recovery point for VM 103.

No independently verified PBS VM recovery procedure has been established during this documentation update.

---

## Restore — AI Observer

**Guest:** observer-prod-01  
**VMID:** 107  
**Proxmox node:** pve-prod-01  
**IP:** `192.168.30.17`

AI Observer has additional recovery considerations because it hosts the Proxmox MCP integration and OpenAI Secure MCP Tunnel client.

**Current status:** VM 107 is included in the configured PBS backup selection, but its latest backup failed and no successful archive was visible through the live integration.

### Special Precautions

Before restoring observer-prod-01:

- Verify a usable VM 107 archive actually exists.
    
- Confirm required credentials are available through approved secure storage.
    
- Determine whether the original observer instance is still running.
    
- Avoid two active VMs using `192.168.30.17`.
    
- Avoid simultaneous active instances using the same production tunnel identity.
    
- Do not start a test-restored observer directly on the production network without isolating it first.
    
- Preserve the observer's read-only Proxmox permissions.
    

### Recovery Requirements

A restored observer must be checked for:

- Proxmox MCP Compose configuration.
    
- MCP environment and authentication configuration.
    
- OpenAI tunnel-client binaries.
    
- Tunnel systemd service.
    
- Protected tunnel configuration and credentials.
    
- Network configuration.
    
- Proxmox API access.
    
- ChatGPT plugin connectivity.
    

A restored VM may contain all files stored on its included virtual disks, but successful recovery still depends on a complete, usable backup and valid credentials.

### Verification

After an approved production restoration:

1. Confirm the original observer is no longer using the production identity.
    
2. Check the restored VM's network configuration.
    
3. Verify Proxmox MCP container health.
    
4. Verify tunnel-client service status.
    
5. Verify local health and readiness endpoints.
    
6. Perform a successful live read-only Proxmox query through ChatGPT.
    
7. Confirm no unintended privilege changes were introduced.
    

The full AI Observer recovery procedure is maintained in:

`runbooks/ai-observer-operations.md`

**Do not duplicate or independently maintain the detailed observer recovery workflow in this general runbook.**

---

## Restore — Docker Stacks from rsync Backup

Use this when a Compose file or Git-excluded `.env` file is lost or corrupted on a participating Docker host.

### Available Source

The documented backup directories on nas-prod-01 are:

```
/mnt/user/backups/stacks/<hostname>/
/mnt/user/backups/appdata/<hostname>/
```

The documented backup transport uses SSH and rsync to:

`192.168.30.16`

The backup script uses the SSH key:

`/root/.ssh/backup_rsa`

### Step 1 — Identify the Affected Host

Confirm whether recovery concerns:

- docker-prod-01
    
- auth-prod-01
    
- immich-prod-01
    

These hosts have separate stack configurations and application data.

Do not restore one host's files into another host merely because they share the `homelab-infra` Git repository.

### Step 2 — Inspect the Source Backup

From the authorized host or NAS administration session, inspect the intended backup.

On nas-prod-01:

```
ls -la /mnt/user/backups/stacks/<hostname>/
ls -la /mnt/user/backups/appdata/<hostname>/
```

Confirm:

- The expected directory exists.
    
- The intended files are present.
    
- File timestamps are suitable for recovery.
    
- The backup is not obviously incomplete or corrupted.
    

The rsync implementation is a rolling mirror, not a versioned backup.

### Step 3 — Preserve the Existing Configuration

Before overwriting any file:

1. Review the current file.
    
2. Determine whether it contains newer valid configuration.
    
3. Preserve a copy of the current file where appropriate.
    
4. Confirm the backup copy is the intended replacement.
    
5. Stop affected applications if required for consistency.
    

Do not use recursive replacement or deletion commands without first determining their impact.

### Step 4 — Restore a Single Compose File

From the affected Docker host, using docker-prod-01 as the example:

```
sudo scp -i /root/.ssh/backup_rsa \
  root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/<stack>/compose.yaml \
  /opt/stacks/<stack>/compose.yaml
```

Replace `<stack>` with the actual stack directory.

Verify the restored file:

```
cd /opt/stacks/<stack>

docker compose config --quiet
```

Review its content and Git status before deploying changes.

A valid Compose file does not guarantee that its referenced images, volumes, networks, or secrets are available.

### Step 5 — Restore a Single .env File

From the affected Docker host:

```
sudo scp -i /root/.ssh/backup_rsa \
  root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/<stack>/.env \
  /opt/stacks/<stack>/.env
```

An `.env` file may contain credentials.

After restoration:

- Apply the host's documented file ownership and permissions.
    
- Ensure the file is not world-readable.
    
- Verify it remains excluded from Git.
    
- Do not display its secret values in logs or documentation.
    

For docker-prod-01, the established Dockman-compatible permissions are:

```
sudo chown gio:service /opt/stacks/<stack>/compose.yaml
sudo chmod 664 /opt/stacks/<stack>/compose.yaml

sudo chown gio:service /opt/stacks/<stack>/.env
sudo chmod 660 /opt/stacks/<stack>/.env
```

**These permissions are specific to docker-prod-01.**

Do not apply them automatically to auth-prod-01, immich-prod-01, or observer-prod-01.

For example, auth-prod-01 has a documented `gio:gio` ownership requirement for its Authentik `.env`.

### Full Stacks Restore to a Fresh Host

If a participating Docker host must be rebuilt, first confirm that the source backup exists and the target environment is ready.

The documented recovery commands for docker-prod-01 are:

```
sudo rsync -av \
  -e "ssh -i /root/.ssh/backup_rsa" \
  root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/ \
  /opt/stacks/
```

For application data:

```
sudo rsync -av \
  -e "ssh -i /root/.ssh/backup_rsa" \
  root@192.168.30.16:/mnt/user/backups/appdata/docker-prod-01/ \
  /opt/appdata/
```

These commands restore files without using `--delete`.

They can still overwrite files with matching paths. Review the destination and source before executing.

After restoration:

- Validate expected stack directories.
    
- Confirm ownership and permissions.
    
- Check Docker and Compose availability.
    
- Verify external mounts and networks.
    
- Restore any necessary credentials.
    
- Start services through their documented procedures.
    
- Verify each application.
    

Do not apply a blanket recursive ownership change across all restored hosts.

### rsync Backup Limitations

The documented rsync backup uses `--delete` during regular backup synchronization.

It maintains a rolling mirror that reflects the source's latest synchronized state.

If a file was deleted or corrupted before the last successful backup, the mirror may already contain the damaged state.

The planned Synology ABB implementation is intended to add historical recovery points, but has not yet been implemented.

---

## Restore — Appdata from rsync Backup

Use this when persistent application data under `/opt/appdata` is lost or corrupted.

### Step 1 — Identify the Application

Before restoring:

- Confirm the affected host.
    
- Confirm the application or stack.
    
- Identify the exact appdata directory.
    
- Determine whether the application uses a database.
    
- Determine whether other containers share the same data.
    
- Verify the available backup's suitability.
    

Do not restore unrelated application directories.

### Step 2 — Stop the Affected Service

Stop the affected service through Dockman or Docker Compose when required for a consistent restore.

For a Compose service that can be stopped independently:

```
cd /opt/stacks/<stack>

docker compose stop <service-name>
```

Use the exact Compose service name.

Some applications require coordinated shutdown of additional components, especially database-backed applications.

Follow their service-specific runbooks.

### Step 3 — Restore Application Data

The original documented full-directory recovery command is:

```
sudo rsync -av \
  -e "ssh -i /root/.ssh/backup_rsa" \
  root@192.168.30.16:/mnt/user/backups/appdata/docker-prod-01/ \
  /opt/appdata/
```

For a single affected application, prefer restoring only its relevant directory after confirming the actual backup layout.

Do not use `--delete` for recovery unless the consequences have been explicitly reviewed.

Preserve the current appdata before overwriting it where appropriate.

### Step 4 — Verify Permissions

Confirm the application's expected UID/GID and filesystem access.

The established service UID/GID on docker-prod-01 is generally `2000:2000`.

However, individual applications and hosts may require different ownership or permission settings.

Apply only the permissions appropriate to the affected service.

### Step 5 — Restart and Verify

Once restoration is complete:

```
cd /opt/stacks/<stack>

docker compose up -d <service-name>
```

Use the service-specific startup procedure if additional dependencies must be restored or started in a particular order.

Check:

```
docker compose ps

docker compose logs --tail 100 <service-name>
```

Verify the application's functionality, not just container status.

For databases, confirm the application can read and use its recovered data.

---

## Restore — Plex Media Server

**Status:** Built-in database backups available; dedicated full-appdata recovery pending.

Plex runs directly on nas-prod-01 under Unraid's Docker management.

It is not managed by Dockman on docker-prod-01 and is not included in the documented Docker-host rsync jobs.

### Available Recovery Source — Built-in Plex Database Backups

Plex creates dated backups of the library and blobs databases every three days during scheduled maintenance.

**Directory:**

```
/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases/
```

**Backup file patterns:**

```
com.plexapp.plugins.library.db-YYYY-MM-DD
com.plexapp.plugins.library.blobs.db-YYYY-MM-DD
```

Backup copies were verified for:

- October 16, 2026
    
- October 19, 2026
    
- October 22, 2026
    
- October 25, 2026
    

These files may provide database recovery points.

They do not constitute complete backups of Plex artwork, metadata directories, preferences, or other application state.

### Restore Precautions

- Do not overwrite the running Plex database while the container is active.
    
- Preserve existing database files before restoration, including relevant SQLite journal/WAL files.
    
- Use the Plex-supported database restoration procedure appropriate to the installed version.
    
- Verify ownership and permissions after restoration.
    
- The documented Plex UID/GID is `2000:2000`.
    
- Verify library availability, metadata, and watch history after recovery.
    

A complete, tested command-by-command Plex database restoration procedure has not yet been established.

These precautions should not be treated as a verified recovery procedure.

### Dedicated Full-Appdata Restore — Pending Phase 5 Wave 8

The planned custom full-appdata backup has not been implemented.

During Wave 8:

1. Implement and verify the dedicated Plex backup.
    
2. Define the restore procedure for its archive format and directory structure.
    
3. Verify handling of the running Plex container and SQLite databases.
    
4. Validate ownership and application startup.
    
5. Test library access and user state.
    
6. Perform a safe restoration test.
    
7. Document the tested commands and results.
    

See `services/media/plex.md` and `runbooks/backup-schedule.md`.

---

## Verify Backup Health

### Check rsync Logs

On each participating Docker host:

```
sudo tail -n 50 /var/log/backup-appdata.log
```

Look for the most recent successful completion.

If a failure is recorded, inspect the exit status and related error messages.

A successful rsync log demonstrates the script reported completion. It does not establish that every backed-up application can be restored successfully.

### Check Healthchecks.io

Review the documented checks:

- `backup-appdata-docker-prod-01`
    
- `backup-appdata-auth-prod-01`
    
- `backup-appdata-immich-prod-01`
    

Confirm that each has received a recent successful heartbeat.

Correlate heartbeat status with the actual backup logs.

### Check PBS Backup Status

Open PBS:

`https://192.168.30.12:8007`

Navigate to the `nas-backups` datastore.

Verify:

1. Expected backup groups are visible.
    
2. Recent archive timestamps exist.
    
3. Backup tasks completed successfully.
    
4. Required VM/LXC disks are included.
    
5. Archive verification reports no unresolved errors.
    
6. A safe restoration test has been completed where appropriate.
    

**Current documented status:** Scheduled PBS backups were failing on both Proxmox nodes as of the October 29–30, 2026 investigation.

Do not report the PBS backup system as healthy until the failure is resolved and the recovery evidence is verified.

### Check AI Observer Recovery Readiness

For observer-prod-01:

- Confirm a successful backup archive exists for VM 107.
    
- Verify the selected backup is accessible.
    
- Confirm the required configuration is included.
    
- Confirm credentials remain recoverable.
    
- Review the production tunnel identity and IP conflict precautions.
    
- Complete a safe restoration test.
    
- Verify a read-only Proxmox query through ChatGPT.
    

See `runbooks/ai-observer-operations.md`.

### Check Plex Built-in Database Backups

On nas-prod-01:

```
find "/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases" \
  -maxdepth 1 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM | %s bytes | %f\n' \
  | sort -r | head -30
```

Verify recent dated backup copies exist for both Plex databases.

This confirms backup-file creation and schedule consistency, not database integrity or recoverability.

**Last verified:** 2026-10-26.

### Check Dedicated Plex Full-Appdata Backup

**Not yet applicable.**

The dedicated backup remains pending Phase 5 Wave 8.

After implementation, add procedures for backup verification, logging, integrity checks, and restoration testing.

---

## Troubleshooting

### PBS Backups Failing with Connection Reset

**Known issue — unresolved.**

The October 29 backup tasks on both Proxmox nodes reported connection resets while communicating with PBS.

Representative errors:

```
HTTP/2.0 connection failed
Error: connection reset
backup register image failed
```

The failures affected multiple guests across both nodes.

This suggests a shared dependency may be involved, but the root cause has not been confirmed.

Areas for investigation include:

- PBS service availability.
    
- Datastore and NAS mount health.
    
- Network connectivity between Proxmox nodes and PBS.
    
- PBS resource utilization and logs.
    
- TLS or protocol-level failures.
    

The investigation should begin with read-only checks.

Do not restart PBS, modify storage, delete archives, or adjust firewall rules without establishing the cause and reviewing the impact.

**Status:** Reserved for separate troubleshooting work.

See `runbooks/backup-schedule.md` for verified findings.

### rsync Script Not Running

1. Check the root crontab:
    
    ```
    sudo crontab -l
    ```
    
    Expected documented entry:
    
    ```
    0 3 * * * /usr/local/bin/backup-appdata.sh
    ```
    
2. Confirm the script exists:
    
    ```
    ls -la /usr/local/bin/backup-appdata.sh
    ```
    
3. Review the existing backup log:
    
    ```
    sudo tail -n 100 /var/log/backup-appdata.log
    ```
    
4. Investigate any reported errors.
    
5. Run the script manually only after reviewing what it will synchronize, including the consequences of its `--delete` behavior.
    

Manual execution is a state-changing backup operation and can replace the destination's existing mirror state.

### rsync SSH Connection Fails

Check whether the configured SSH key exists:

```
sudo ls -la /root/.ssh/backup_rsa
```

Test connectivity only from an authorized backup host:

```
sudo ssh -i /root/.ssh/backup_rsa \
  root@192.168.30.16 \
  echo "test"
```

If authentication fails:

- Check network connectivity.
    
- Verify the intended SSH account is available.
    
- Confirm the authorized public key is installed.
    
- Review key permissions.
    
- Review the relevant SSH logs.
    

Do not print private keys or other secret material into logs or documentation.

### PBS Garbage Collection Fails with Permission Error

A previously documented failure scenario involved incorrect ownership or unexpected files within the PBS datastore directory.

However, the cause of a new GC failure must be verified rather than assumed.

**Datastore path:** `/mnt/backups/pbs`  
**NAS path:** `/mnt/user/backups/pbs`

Initial investigation:

1. Review the PBS GC task log.
    
2. Confirm the datastore is mounted and accessible.
    
3. Inspect datastore directory ownership and permissions.
    
4. Identify unexpected files or directories.
    
5. Determine whether the issue involves ownership, filesystem access, or datastore integrity.
    

On nas-prod-01, a read-only inspection may include:

```
ls -la /mnt/user/backups/pbs/
```

**Do not manually delete, move, or change ownership of PBS datastore files merely because they appear unfamiliar.**

PBS manages its own datastore structures, indexes, manifests, and chunk data.

Improper filesystem changes can damage recovery data.

Any corrective action must be based on the confirmed problem and appropriate PBS procedures.

### Plex Dedicated Backup Missing

The dedicated Plex full-appdata backup had not been implemented as of the October 26, 2026 verification.

Do not assume Docker-host rsync backups include Plex because Plex runs directly on Unraid.

Built-in Plex database backups were verified, but they remain inside the Plex appdata directory and do not provide an independent full-appdata recovery copy.

Refer to Phase 5 Wave 8 before creating or troubleshooting the planned dedicated backup.

### AI Observer Recovery Unavailable

If a usable VM 107 backup cannot be found:

1. Verify the PBS datastore and archive availability.
    
2. Confirm whether an older valid recovery point exists.
    
3. Review the available `homelab-infra` configuration.
    
4. Confirm required credentials are stored securely.
    
5. Determine whether the observer can be repaired in place.
    
6. Consider a controlled rebuild only if full-VM recovery is unavailable.
    

The rebuild process is documented as a contingency in `runbooks/ai-observer-operations.md`.

It has not yet been tested as a complete disaster recovery procedure.

Do not assume the observer can be fully rebuilt from Git alone.

---

## Recovery Safety Principles

The following principles apply to every restoration procedure:

1. **Verify the recovery point first.** Never restore from an assumed backup.
    
2. **Preserve existing data when practical.** Avoid destructive replacement without assessing the current state.
    
3. **Isolate test restorations.** Prevent duplicate IP addresses, identities, and externally connected services.
    
4. **Use application-specific recovery procedures.** Database-backed services may require coordinated restoration.
    
5. **Avoid unnecessary full-stack restarts.** Restore and restart only the affected components where supported.
    
6. **Preserve credentials securely.** Do not expose tokens, API keys, or private keys during recovery.
    
7. **Validate after restoration.** Confirm the application works, not just that the VM or container started.
    
8. **Document verified outcomes.** Record what was restored, which backup was used, and whether the recovery succeeded.
    

A documented recovery procedure is not automatically a tested recovery procedure.

---

## Known Recovery Gaps

|Area|Status|Required Follow-Up|
|---|---|---|
|Proxmox PBS backups|Failing|Investigate and restore successful backup execution|
|Historical PBS recovery points|Not fully verified|Inspect PBS datastore directly|
|pbs-prod-01 VM 103|Excluded from job|Establish separate PBS VM recovery strategy|
|AI Observer VM 107|No successful archive visible|Verify backup and perform safe restoration test|
|Docker-host rsync recovery|Documented; latest executions not reverified|Check current jobs and test appropriate file restoration|
|Plex built-in DB backups|Files last verified 2026-10-26|Establish and test database restore procedure|
|Plex full-appdata backup|Pending|Implement Phase 5 Wave 8|
|Synology ABB|Pending|Implement and verify versioned backups|

These items should remain open until the corresponding work has been completed and verified.

---

## Related Documentation

- `runbooks/backup-schedule.md` — Backup schedules, storage destinations, verification, and the open PBS incident.
    
- `runbooks/ai-observer-operations.md` — AI Observer health checks, maintenance, troubleshooting, and disaster recovery.
    
- `services/infra/ai-observer.md` — AI Observer architecture, configuration, security, and credential handling.
    
- `runbooks/git-workflow.md` — Git repositories, multi-host synchronization, and credential handling.
    
- `runbooks/updating-containers.md` — Container updates and maintenance procedures.
    
- `services/infra/dockman.md` — Docker management on docker-prod-01.
    
- `services/media/plex.md` — Plex deployment, persistent storage, and backup requirements.
    
- `phases/phase-5-hardening.md` — Planned backup and operational hardening work.
    

---

## Verification Summary — October 2026

|Item|Result|
|---|---|
|Nightly PBS backup job configured|Verified|
|Nightly PBS backup job enabled|Verified|
|October 29 PBS backups — pve-prod-01|Failed|
|October 29 PBS backups — pve-prod-02|Failed|
|PBS VM 103 excluded|Verified|
|AI Observer VM 107 included in backup selection|Verified|
|AI Observer successful archive|Not visible through live integration|
|AI Observer restoration test|Not performed|
|Docker rsync latest successful execution|Not reverified|
|Plex built-in database backup files|Last verified 2026-10-26|
|Dedicated Plex full-appdata backup|Not implemented|
|Synology ABB|Not implemented|

**Backup and recovery readiness must be determined from verified recovery evidence, not backup-job configuration alone.**

The outstanding PBS incident and associated recovery testing remain separate operational work.