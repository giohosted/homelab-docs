# Backup & Restore Runbook

**Last Updated:** 2026-11-14  
**Status:** Docker-based PBS active. Scheduled backups verified on all three Proxmox nodes November 14, 2026. Authentik VM 104 successfully backed up from its new host, `pve-prod-03`, following migration. Targeted CT 100 integrity verification and file-level recovery previously passed. Full VM/CT restoration, PBS maintenance-job execution, and long-term recovery reliability remain unverified.

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

### Backup Types

| Type | Tool | What It Covers | Location | Status |
|---|---|---|---|---|
| VM & CT backups | PBS Docker on Unraid | Selected Proxmox VM/LXC disks and configurations | nas-prod-01 `/mnt/zfs-mirror/backups/pbs` (direct bind mount) | November 14 scheduled backups successful on all three nodes; VM 104 post-migration backup successful; full guest recovery not tested |
| Stacks & appdata | rsync | Docker Compose files, `.env` files, and persistent appdata on participating Docker hosts | nas-prod-01 `/mnt/user/backups/stacks` and `/mnt/user/backups/appdata` | Documented operational; latest runs not reverified |
| Plex built-in DB backup | Plex Scheduled Tasks | Plex library and blobs databases only | nas-prod-01 `/mnt/user/appdata/plex` | Last verified 2026-10-26 |
| Plex full-appdata backup | Custom script planned | Complete Plex persistent appdata | NAS backups share — destination TBD | Pending |
| Versioned backup | Synology ABB planned | NAS backups share with historical recovery points | Synology NAS | Pending |

### Current Proxmox Architecture

**Cluster:** `homelab-v3`  
**Topology:** Three voting Proxmox nodes  
**Quorum:** Two of three votes required  
**External QDevice:** Removed  
**Proxmox HA:** Disabled  
**Last live verification:** November 14, 2026

| Node | Management IP | Hardware | Current Guests |
|---|---|---|---|
| pve-prod-01 | 192.168.10.11 | Minisforum MS-A2 | CT 100, VM 101, VM 107; VM 106 stopped |
| pve-prod-02 | 192.168.10.12 | Dell OptiPlex 5000 SFF | VM 105 |
| pve-prod-03 | 192.168.10.13 | Lenovo ThinkCentre M90q Tiny | CT 102, VM 104 |

The cluster was confirmed quorate with all three nodes online November 14.

**Important recovery distinction:** Three-node quorum does not provide automatic guest failover. Guest disks reside on node-local ZFS storage, and Proxmox HA policies are not enabled.

Unexpected node failure therefore requires an appropriate manual recovery or restore procedure.

### Scope Limitations

PBS backs up the virtual disks and other resources included in a guest's backup configuration. It does not necessarily include external NAS shares, independently mounted remote filesystems, or guest disks explicitly excluded from the backup.

The November 14 live backup-job inspection confirmed that all seven registered guests are included in the job.

| VMID | Guest | Host | Included Volumes |
|---|---|---|---|
| 100 | dns-prod-01 | pve-prod-01 | LXC rootfs |
| 101 | docker-prod-01 | pve-prod-01 | scsi0, efidisk0 |
| 102 | dns-prod-02 | pve-prod-03 | LXC rootfs |
| 104 | auth-prod-01 | pve-prod-03 | scsi0, efidisk0 |
| 105 | immich-prod-01 | pve-prod-02 | scsi0, efidisk0 |
| 106 | dragonwilds-server | pve-prod-01 | scsi0, efidisk0 |
| 107 | observer-prod-01 | pve-prod-01 | scsi0, efidisk0 |

**External data considerations:**

- Immich VM 105 mounts photos from Unraid. The NAS photo library is not included merely because the VM's virtual disks are backed up.
- Docker workloads using NAS-mounted media or application data may have external dependencies requiring separate recovery.
- PBS running as a Docker container on Unraid is not itself protected by the Proxmox guest backup job.
- A Proxmox guest backup does not independently protect the underlying NAS, its ZFS pool, or the PBS datastore against complete NAS loss.

Docker rsync jobs cover the documented participating hosts:

- docker-prod-01
- auth-prod-01
- immich-prod-01

They do not automatically protect observer-prod-01 or Plex appdata running directly on Unraid.

The dedicated Plex full-appdata backup and Synology ABB remain pending Phase 5 Wave 8 implementation.

---

## PBS Backup Incident — Operational Recovery, October 2026 (Historical)

**Status at October 30:** Immediate failure mechanism identified and temporarily cleared; failures recurred October 31–November 2. The original VM NFS path was replaced by Docker-based PBS November 3–5, while the original invalidation trigger remains unconfirmed.

**Identified:** October 29, 2026  
**Affected:** Both then-existing PVE nodes and the PBS datastore accessed over Unraid NFS

The October 29 scheduled jobs failed for all seven selected guests on the two PVE nodes with `connection reset` and `HTTP/2.0 connection failed`.

Repeated task errors were visible through at least September 30–October 29 on pve-prod-01; the exact first failure date is not established.

PBS proxy logs recorded the server-side error:

```text
TASK ERROR: unable to get shared lock - ESTALE: Stale file handle
```

A manual CT 100 backup reproduced this exact error at 00:31:10 CDT on October 30.

PBS proxy PID 594 (running since August 11) retained an open `/mnt/backups/pbs/.lock` descriptor that returned `ESTALE` when accessed via `/proc/594/fd/21`.

The current `.lock` pathname remained accessible, and a newly opened POSIX shared lock succeeded.

This confirms a stale, long-lived proxy file handle as the immediate failure mechanism; the upstream event that invalidated it is not confirmed.

The datastore was on Unraid's healthy `zfs-mirror` pool at `/mnt/zfs-mirror/backups/pbs`, but the former PBS VM connected to it through an NFSv3 export of `/mnt/user/backups` (`fuse.shfs`).

FUSE/NFS file-handle behavior is a leading hypothesis for the original invalidation. No FUSE/NFS setting or ZFS export was changed.

**This describes the retired VM-based PBS deployment. The current Docker-based PBS accesses the datastore directly and does not use this NFS mount.**

### Corrective Action and Verification

- On October 30 at 00:55:42 CDT, the operator restarted `proxmox-backup-proxy` after checking that no PBS tasks were running. The service came back active/running under PID 1372698.
- Manual CT 100 backup `ct/100/2026-10-30T05:57:23Z` completed with `TASK OK` and was visible in PBS with its manifest, configuration, catalog, and filesystem index.
- That CT 100 snapshot passed targeted PBS integrity verification at 01:02 CDT, reporting zero errors.
- The October 30 02:00 CDT scheduled backup jobs completed with `TASK OK` on both then-existing nodes: pve-prod-01 (100, 101, 104, 106, 107) and pve-prod-02 (102, 105).
- Existing historical snapshots were found in PBS, including older snapshots with previous `Verify OK` status. Earlier empty results from the read-only Proxmox archive API were not a reliable inventory of the datastore.

### Recovery Impact and Outstanding Checks (Historical October 30 Assessment)

Successful backup execution and one verified CT 100 snapshot demonstrated operational recovery, but did not prove full disaster recovery readiness.

The other newly created archives had not all been independently integrity-verified, and no isolated restore test had been performed.

At the time, PBS VM 103 was excluded from the scheduled backup job.

The operator elected to monitor Discord backup alerts and leave NFS/Unraid configuration unchanged.

`Permit exclusive shares = No` remained in place; proposed FUSE-bypass changes were not implemented.

Confirm notification delivery separately rather than assuming a task log saying notified proves receipt.

**Subsequent disposition:** The October 30 recovery did not permanently resolve the issue. Failures recurred October 31–November 2, and the VM-based PBS deployment was replaced by Docker-based PBS on Unraid during November 3–5.

The former PBS VM 103 was subsequently **destroyed before the November 13 Proxmox host replacement**.

See `runbooks/backup-schedule.md` for complete incident evidence and chronology.

---

## PBS Docker Deployment — Current Recovery Endpoint

**Deployment date:** November 3–5, 2026  
**Last backup execution verification:** November 14, 2026  
**Status:** Active

The original PBS VM 103 (`pbs-prod-01`, `192.168.30.12`) was powered off following the October 31–November 2 recurrence of `ESTALE`.

At the November 5 verification, it remained stopped as a temporary rollback artifact.

**That historical state is no longer current. VM 103 was destroyed before the November 13 replacement of the original pve-prod-02.**

The active service is the `proxmox-backup-server` container on Unraid `nas-prod-01`, using the unofficial image `ayufan/proxmox-backup-server:latest`.

The installed PBS UI version was 4.2.0 at the last direct verification.

### Active Configuration

| Setting | Active Configuration |
|---|---|
| PVE storage ID | `pbs-prod-01` (unchanged) |
| PBS UI and API | `https://192.168.30.16:8007` |
| PBS host | `nas-prod-01` |
| PBS datastore | `nas-backups` |
| Container mount | `/backups` |
| Direct ZFS host mount | `/mnt/zfs-mirror/backups/pbs` |
| Persistent PBS config | `/mnt/user/appdata/pbs/etc` |
| Persistent PBS logs | `/mnt/user/appdata/pbs/logs` |
| Persistent PBS runtime lib | `/mnt/user/appdata/pbs/lib` |
| Authentication for backup clients | `pve-backup@pbs!pve-cluster` API token; `DatastoreBackup` on `/datastore/nas-backups` |
| Interactive admin access | `gio@authentik` (Admin `/`), fallback `admin@pbs` (local PBS realm) |
| Retention | Keep-last 3, keep-daily 7, keep-weekly 4, keep-monthly 3 |
| Garbage collection | Saturday 04:00 |
| Verification | Saturday 05:00 |

The image runs with:

- Host networking.
- 2 GiB RAM limit.
- `/run` tmpfs.
- `--stop-signal=SIGHUP`.
- `--restart unless-stopped`.
- Unraid Docker template with Privileged disabled.

Host-network binding may expose TCP 8007 on the NAS management IP as well as Services VLAN; firewall reachability has not been independently reviewed.

Keep the ZFS datastore available before PBS starts. Actual automatic start-after-ZFS-mount ordering has not been separately tested.

The underlying `zfs-mirror/backups` dataset also contains non-PBS rsync data.

### Verified Backup and Recovery Capabilities

**November 5:**

- Manual CT 100 backup successful.
- Targeted snapshot integrity verification passed.
- Scheduled backups successful on both then-existing PVE nodes.
- File-level recovery from CT 100 performed by browsing and downloading `/etc/hostname` and checking its contents.

**November 14:**

- Scheduled 02:00 PBS backup completed successfully on pve-prod-01.
- Scheduled 02:00 PBS backup completed successfully on replacement pve-prod-02.
- Scheduled 02:00 PBS backup completed successfully on pve-prod-03.
- All seven current guests were included in the cluster-wide backup job.
- After VM 104 migrated to pve-prod-03, a fresh manual snapshot backup succeeded from the destination node.

**Authentik post-migration backup:**

| Item | Result |
|---|---|
| Guest | VM 104 — auth-prod-01 |
| Proxmox host | pve-prod-03 |
| Date | November 14, 2026 |
| Completion | 14:09:45 CST |
| Mode | Snapshot |
| Included disks | 32 GiB scsi0 and EFI disk |
| Reused backup data | 30.52 GiB (95%) |
| Job duration | Approximately 44 seconds |
| Status | `TASK OK` |

These results establish successful backup execution from the three-node architecture.

**They do not establish full disaster-recovery readiness.** No complete VM or LXC has been independently restored in an isolated test.

### Authentication and Alerts

The shared Authentik OIDC provider redirect was updated from:

`https://192.168.30.12:8007`

to:

`https://192.168.30.16:8007`

Its shared client secret was rotated, and fresh SSO logins succeeded to PBS and both then-existing PVE nodes on November 5.

A fresh Authentik SSO functional test also passed following the VM 104 migration on November 14. The exact Proxmox client endpoints tested were not itemized, so this does not independently establish OIDC redirect configuration for node 03.

The Discord target sent a successful test message. A warning/error matcher was configured but not independently exercised by generating a failure.

Do not include secret values or webhook URLs in this runbook.

### Updates and Subscription

The Docker image is unofficial and described as unmaintained by its maintainer.

Docker image updates, if published, should be evaluated and deployed through Unraid's Docker update workflow with a tested recovery path.

Do not assume an `apt upgrade` in the ephemeral container updates the deployed image.

As of November 5, Unraid reported no available image update, while official Proxmox Backup Server documentation covered newer PBS 4.2.8.

PBS does not require a paid subscription for its core features; the GUI no-subscription warning is expected.

### Outstanding Recovery Work

- Verify successful scheduled prune, garbage collection, and snapshot verification operations.
- Continue checking daily backup execution on all three nodes.
- Confirm the first scheduled backup of VM 104 from pve-prod-03.
- Test isolated full guest restoration.
- Review network exposure of TCP 8007.
- Validate PBS startup ordering relative to ZFS availability.
- Establish a protected recovery strategy for the Docker configuration and datastore appropriate to the accepted risk.

The operator intentionally declined a migration-time ZFS snapshot and accepted datastore recreation if older archives became unusable.

---

## Restore — VM or LXC from PBS

Use this procedure when a VM or LXC has been lost, corrupted, or must be restored to an earlier recovery point.

**Status:** Backup execution verified on all three nodes November 14. CT 100 previously passed integrity verification and `/etc/hostname` was successfully restored at file level. No complete VM/CT restoration test has been performed.

### Backup Infrastructure

| Component | Value |
|---|---|
| Proxmox storage ID | `pbs-prod-01` |
| PBS host | `192.168.30.16` (nas-prod-01 Docker Host networking) |
| PBS datastore | `nas-backups` |
| PBS datastore mount | `/backups` inside container |
| NAS destination | `/mnt/zfs-mirror/backups/pbs` (direct ZFS) |
| Primary Proxmox host | `192.168.10.11` |
| Secondary Proxmox host | `192.168.10.12` |
| Third Proxmox host | `192.168.10.13` |

All three nodes currently have access to the shared PBS storage definition.

The VM or LXC's current host is not necessarily the only possible restore destination. Actual restore feasibility depends on storage, resources, network configuration, backup format, and any service-specific requirements.

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
- pve-prod-03: `https://192.168.10.13:8006`

From the Proxmox interface:

1. Select a node with access to the `pbs-prod-01` storage.
2. Select the `pbs-prod-01` storage entry.
3. Open its backup-content view.
4. Locate the intended guest.
5. Review available backup timestamps.

Alternatively, inspect PBS directly:

`https://192.168.30.16:8007`

Navigate to the `nas-backups` datastore and review its backup content.

If no usable archive is available, stop.

Do not attempt to restore from an assumed or nonexistent backup.

**Authentication dependency:** If Authentik is unavailable, use the appropriate local emergency administrator rather than relying solely on OIDC SSO. Preserve and protect the configured local emergency access methods.

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

**Special cases:**

- **VM 104 — Authentik:** Confirm the archive contains its primary disk and EFI disk. Recovery requires preventing conflicts with the production IdP, including its IP and authentication services.
- **VM 105 — Immich:** Its system disk backup does not include the external Unraid photo-library mount. Confirm photo and database recovery requirements separately.
- **VM 107 — AI Observer:** Confirm the selected recovery point includes the currently deployed four integrations, their protected runtime configuration, and tunnel clients.
- **CT 102 — Secondary DNS:** Confirm the intended AdGuard configuration and synchronization behavior before reintroducing a restored instance.

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

Do not enable automatic startup for a test restoration when the restored guest could conflict with an active production service.

This is especially important for services using static IP addresses, authentication credentials, cluster membership, or externally connected tunnels.

**Three-node architecture consideration:** VM backups can be restored to a suitable different node if the destination meets the workload requirements. However, all current `local-zfs` guest storage is node-local; a failed host's guest disks are not automatically available on another node.

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

Do not overwrite an existing production guest without an approved recovery plan.

For a test restoration, use an alternate VMID and ensure that network connectivity, startup settings, and service identities will not conflict with production.

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

Current PBS retention settings, visually verified in the Docker PBS instance on November 5:

- Keep Last: 3
- Keep Daily: 7
- Keep Weekly: 4
- Keep Monthly: 3

Effective retention depends on actual backup creation, pruning, datastore state, and the current retention configuration.

Do not infer available recovery dates from the retention policy alone.

### Retired PBS VM 103 — Exclusion and Recovery Implications

**Current status:** VM 103 was destroyed before November 13, 2026.

The original PBS VM remains listed as an exclusion in the cluster backup job:

`backup-54015f90-26e5`

That exclusion is harmless because VM 103 no longer exists.

**The retired VM is not a rollback option.**

Do not attempt to start or restore the historical PBS VM against the active Docker PBS datastore without a separately designed and approved recovery plan.

The existing backup job does not protect the current PBS Docker instance, its configuration, or its NAS-hosted datastore.

Recovery requirements for these components must be considered separately.

The obsolete VM 103 exclusion may be removed during a future backup-job housekeeping review after confirming coverage remains correct.

---

## Restore — AI Observer

**Guest:** observer-prod-01  
**VMID:** 107  
**Proxmox node:** pve-prod-01  
**IP:** 192.168.30.17

AI Observer has additional recovery considerations because it hosts four independent read-only MCP integrations (Proxmox, Unraid, Komodo, and UniFi) and four OpenAI Secure MCP Tunnel clients.

The private `homelab-infra` repository tracks the four MCP deployment definitions, the pinned UniFi source submodule, and inert tunnel YAML/systemd recovery templates (verified Git commits `6502c59` and `9887296`, November 6, 2026).

This does not cover active credentials, installed binaries, a tested runtime rebuild, or backup integrity.

**Updated backup status:**

VM 107 completed its scheduled October 30, 2026 backup (`TASK OK`) on pve-prod-01. That backup predated the November 1 Komodo and November 5 UniFi additions.

A newer scheduled backup completed successfully on **November 14, 2026**. Its primary 20 GiB disk and EFI disk were included.

This provides current backup-execution evidence after the four integrations were deployed.

However, independent snapshot integrity verification and a safe restoration test have not been completed.

The operator previously opted to skip Phase 4 reboot/recovery testing.

### Special Precautions

Before restoring observer-prod-01:

- Verify a usable VM 107 archive actually exists.
- Confirm required credentials are available through approved secure storage.
- Determine whether the original observer instance is still running.
- Avoid two active VMs using `192.168.30.17`.
- Avoid simultaneous active instances using any of the four production tunnel identities.
- Do not start a test-restored observer directly on the production network without isolating it first.
- Preserve the four integrations' approved MCP read-only boundaries; the UniFi upstream API key itself is not independently verified read-only.

### Recovery Requirements

A restored observer must be checked for:

- All four MCP Compose configurations and excluded `.env` files; initialize the pinned UniFi `unifi-mcp/source` Git submodule and restore its locally built Docker image if required.
- Four protected tunnel configurations and runtime keys; Proxmox/Unraid additionally use local MCP authorization files.
- All four installed systemd tunnel units, shared OpenAI tunnel-client release, and protected credential ownership/permissions; Git copies under `observer/tunnel-client/` are not automatically installed.
- Network configuration and all four upstream API connections.
- All four ChatGPT plugin integrations, maintaining the recorded 84 + 26 + 40 + 55 = 205 Read / 0 Write tool inventory unless an authorized change has been verified.

A restored VM may contain all files stored on its included virtual disks, but successful recovery still depends on a complete, usable backup and valid credentials.

### Verification

After an approved production restoration:

1. Confirm the original observer is no longer using the production identity.
2. Check the restored VM's network configuration.
3. Verify all four MCP containers and their approved local-only endpoint bindings.
4. Verify all four tunnel systemd services and separate readiness ports (8081, 8083, 8085, 8087).
5. Verify local MCP health/protocol and protected upstream authentication without exposing secrets.
6. Perform one successful read-only ChatGPT query through each Homelab Proxmox, Unraid, Komodo, and UniFi integration.
7. Confirm no unintended privilege, tool-registration, or network exposure changes were introduced.

The full AI Observer recovery procedure is maintained in:

`runbooks/ai-observer-operations.md`

Do not duplicate or independently maintain the detailed observer recovery workflow in this general runbook.

---

## Restore — Docker Stacks from rsync Backup

Use this when a Compose file or Git-excluded `.env` file is lost or corrupted on a participating Docker host.

### Available Source

The documented backup directories on nas-prod-01 are:

```text
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

**Authentik migration note:** `auth-prod-01` now resides on `pve-prod-03`, but its guest hostname, IP, and documented rsync locations remain unchanged.

Its latest post-migration rsync execution and heartbeat have not been independently verified.

### Step 2 — Inspect the Source Backup

From the authorized host or NAS administration session, inspect the intended backup.

On nas-prod-01:

```bash
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

From the affected Docker host, using `docker-prod-01` as the example:

```bash
sudo scp -i /root/.ssh/backup_rsa \
  root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/<stack>/compose.yaml \
  /opt/stacks/<stack>/compose.yaml
```

Replace `<stack>` with the actual stack directory.

Verify the restored file:

```bash
cd /opt/stacks/<stack>
docker compose config --quiet
```

Review its content and Git status before deploying changes.

A valid Compose file does not guarantee that its referenced images, volumes, networks, or secrets are available.

### Step 5 — Restore a Single .env File

From the affected Docker host:

```bash
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

```bash
sudo chown gio:service /opt/stacks/<stack>/compose.yaml
sudo chmod 664 /opt/stacks/<stack>/compose.yaml

sudo chown gio:service /opt/stacks/<stack>/.env
sudo chmod 660 /opt/stacks/<stack>/.env
```

These permissions are specific to docker-prod-01.

Do not apply them automatically to auth-prod-01, immich-prod-01, or observer-prod-01.

For example, auth-prod-01 has a documented `gio:gio` ownership requirement for its Authentik `.env`.

### Full Stacks Restore to a Fresh Host

If a participating Docker host must be rebuilt, first confirm that the source backup exists and the target environment is ready.

The documented recovery commands for docker-prod-01 are:

```bash
sudo rsync -av \
  -e "ssh -i /root/.ssh/backup_rsa" \
  root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/ \
  /opt/stacks/
```

For application data:

```bash
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

```bash
cd /opt/stacks/<stack>
docker compose stop <service-name>
```

Use the exact Compose service name.

Some applications require coordinated shutdown of additional components, especially database-backed applications.

Follow their service-specific runbooks.

### Step 3 — Restore Application Data

The original documented full-directory recovery command is:

```bash
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

```bash
cd /opt/stacks/<stack>
docker compose up -d <service-name>
```

Use the service-specific startup procedure if additional dependencies must be restored or started in a particular order.

Check:

```bash
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

Directory:

```text
/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases/
```

Backup file patterns:

```text
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

```bash
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

`https://192.168.30.16:8007`

Navigate to the `nas-backups` datastore.

Verify:

1. Expected backup groups are visible.
2. Recent archive timestamps exist.
3. Backup tasks completed successfully.
4. Required VM/LXC disks are included.
5. Archive verification reports no unresolved errors.
6. A safe restoration test has been completed where appropriate.

**Latest verified execution state (November 14):**

- Scheduled backups completed successfully on all three Proxmox nodes.
- CT 100 and VMs 101, 104, 106, and 107 were backed up on node 01 before the Authentik migration.
- VM 105 was backed up on replacement node 02.
- CT 102 was backed up on node 03.
- A separate manual snapshot of VM 104 completed successfully on node 03 after migration.

The earlier CT 100 targeted integrity verification and file-level `/etc/hostname` recovery passed.

Continue checking archive integrity and perform isolated full restoration before declaring comprehensive recovery readiness.

### Check AI Observer Recovery Readiness

For observer-prod-01:

- Confirm a successful backup archive exists for VM 107.
- Verify the selected backup is accessible.
- Confirm the required configuration is included.
- Confirm credentials remain recoverable.
- Review production tunnel identity and IP conflict precautions.
- Complete a safe restoration test.
- Verify read-only queries through all four ChatGPT MCP integrations.

See `runbooks/ai-observer-operations.md`.

### Check Plex Built-in Database Backups

On nas-prod-01:

```bash
find "/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases" \
  -maxdepth 1 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM | %s bytes | %f\n' \
  | sort -r | head -30
```

Verify recent dated backup copies exist for both Plex databases.

This confirms backup-file creation and schedule consistency, not database integrity or recoverability.

Last verified: 2026-10-26.

### Check Dedicated Plex Full-Appdata Backup

Not yet applicable.

The dedicated backup remains pending Phase 5 Wave 8.

After implementation, add procedures for backup verification, logging, integrity checks, and restoration testing.

---

## Troubleshooting

### PBS Backups Failing with Connection Reset or ESTALE

**Current deployment:** Docker PBS on `nas-prod-01`, using direct-ZFS datastore access.

**Historical deployment:** PBS VM 103, using NFSv3 through Unraid `fuse.shfs`. That VM was deleted and is not an active recovery endpoint.

Do not assume a new PBS backup failure has the same cause as the October 2026 incident.

#### Read-Only First Response — Current Docker PBS

1. Identify failed Proxmox tasks and record their UPIDs, timestamps, affected guests, and error messages.
2. Determine whether failures affect one node or all three.
3. Check that `pbs-prod-01` storage is available through Proxmox.
4. Confirm whether the PBS Docker container is running on `nas-prod-01`.
5. Review the PBS container's logs through authorized Unraid Docker management.
6. Confirm the ZFS datastore is available and healthy.
7. Check storage capacity and relevant filesystem permissions.
8. Identify whether errors involve networking, authentication, datastore access, locking, or container startup.

**Current datastore paths:**

| Location | Path |
|---|---|
| Unraid host | `/mnt/zfs-mirror/backups/pbs` |
| Docker container | `/backups` |
| Proxmox storage | `pbs-prod-01` |
| PBS datastore | `nas-backups` |

The former VM's `/mnt/backups/pbs` mount is not the current Docker datastore path.

Do not restart services, modify the datastore, or change NFS exports until the actual failure mechanism is identified and the potential impact reviewed.

#### Historical NFS Failure — October 2026

The October 30 proxy restart was only a temporary recovery; the same error recurred October 31–November 2.

The active PBS Docker path bypasses the former NFS/FUSE datastore mount.

The commands below preserve the historical investigation procedure. They are **not instructions to execute against the current Docker PBS deployment**.

Known symptoms:

- Proxmox tasks showed `HTTP/2.0 connection failed` or `connection reset`.
- PBS proxy journal recorded `unable to get shared lock - ESTALE: Stale file handle`.
- The PBS proxy held a stale, long-lived `.lock` descriptor.

Historical read-only investigation steps:

1. Check failures on the then-existing Proxmox nodes and record task IDs, timestamps, affected guests, and error strings.

2. Inspect PBS proxy logs around the failure time (use UTC consistently):

   ```bash
   TZ=UTC journalctl --utc -u proxmox-backup-proxy -u proxmox-backup \
     --since "YYYY-MM-DD HH:MM:SS" --until "YYYY-MM-DD HH:MM:SS" \
     --no-pager -o short-iso
   ```

3. Inspect the former NFS mount and datastore directory:

   ```bash
   findmnt --kernel -T /mnt/backups/pbs -o TARGET,SOURCE,FSTYPE,OPTIONS
   ls -la /mnt/backups/pbs
   ```

4. Inspect the proxy PID and its open lock descriptor. Do not assume historical PID 594 would recur:

   ```bash
   pid=$(systemctl show -P MainPID proxmox-backup-proxy)
   echo "PBS proxy PID: $pid"
   ls -l "/proc/$pid/fd" 2>&1 | grep -E 'backups|\.lock|deleted' || true
   ```

   If a relevant descriptor is found, inspect only that descriptor (using its actual FD number):

   ```bash
   readlink "/proc/$pid/fd/FD_NUMBER"
   stat -L "/proc/$pid/fd/FD_NUMBER"
   ```

   An `ESTALE` result on the open descriptor while the current pathname works is strong evidence of a stale in-process handle.

   The label `(deleted)` is not by itself proof that PBS files were deleted.

5. From Unraid, historical read-only checks included `zpool status -v`, `exportfs -v`, and `findmnt --kernel -T /mnt/user/backups`.

   Do not attribute a new failure to FUSE without evidence.

#### Historical Corrective Action

In October 2026, after checking that PBS had no running tasks, the operator performed:

```bash
systemctl restart proxmox-backup-proxy
```

This was a service-changing recovery operation, not an automatic first step.

It interrupted the old PBS proxy and cleared the stale process descriptor but did not resolve the underlying recurrence.

**Do not treat this as an approved recovery command for the current container.** The previous PBS VM no longer exists, and the active container uses a different datastore access path.

After any authorized current-deployment recovery, verify the service, perform an appropriate controlled backup, check the resulting archive, and observe subsequent scheduled jobs.

Do not blindly:

- Run `exportfs -ra`.
- Change global Unraid exclusive-share settings.
- Remount unrelated NFS shares.
- Delete `.lock` files.
- Delete incomplete snapshot directories.
- Run garbage collection or pruning as a troubleshooting shortcut.
- Change dataset ownership recursively.
- Migrate the datastore without an approved plan.

The former direct-ZFS NFS export was only a proposal during the October incident. The implemented November change instead used Docker PBS with a direct bind mount.

### rsync Script Not Running

1. Check the root crontab:

   ```bash
   sudo crontab -l
   ```

   Expected documented entry:

   ```cron
   0 3 * * * /usr/local/bin/backup-appdata.sh
   ```

2. Confirm the script exists:

   ```bash
   ls -la /usr/local/bin/backup-appdata.sh
   ```

3. Review the existing backup log:

   ```bash
   sudo tail -n 100 /var/log/backup-appdata.log
   ```

4. Investigate any reported errors.

5. Run the script manually only after reviewing what it will synchronize, including the consequences of its `--delete` behavior.

Manual execution is a state-changing backup operation and can replace the destination's existing mirror state.

### rsync SSH Connection Fails

Check whether the configured SSH key exists:

```bash
sudo ls -la /root/.ssh/backup_rsa
```

Test connectivity only from an authorized backup host:

```bash
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

**Current Docker datastore paths:**

- Container: `/backups`
- Unraid host: `/mnt/zfs-mirror/backups/pbs`

**Historical VM datastore paths (retired):**

- Former PBS VM: `/mnt/backups/pbs`
- Former Unraid share alias: `/mnt/user/backups/pbs`

Initial investigation:

1. Review the PBS GC task log.
2. Confirm the active datastore is mounted and accessible.
3. Inspect datastore directory ownership and permissions.
4. Identify unexpected files or directories.
5. Determine whether the issue involves ownership, filesystem access, or datastore integrity.

On nas-prod-01, a read-only inspection may include:

```bash
ls -la /mnt/zfs-mirror/backups/pbs/
```

Do not manually delete, move, or change ownership of PBS datastore files merely because they appear unfamiliar.

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

| Area | Current Status | Required Follow-Up |
|---|---|---|
| Proxmox PBS backups | Docker deployment active; November 14 scheduled jobs passed on all three nodes | Monitor continued execution; verify prune, GC, and scheduled snapshot verification |
| Authentik VM 104 | Migrated to pve-prod-03; fresh November 14 manual backup succeeded | Confirm first scheduled backup from node 03; perform isolated restore test |
| Historical PBS recovery points | Older snapshots found; completeness varies | Verify required snapshots and preserve historical recovery options |
| PBS Docker container | Config/datastore not protected by PVE guest jobs; original VM 103 destroyed | Plan PBS Docker configuration and datastore recovery |
| AI Observer VM 107 | November 14 scheduled backup succeeded | Confirm snapshot integrity and perform safe isolated restoration |
| Immich VM 105 | November 14 scheduled backup on replacement node 02 succeeded | Validate VM recovery plus separate database and NAS photo-library protection |
| Docker-host rsync recovery | Documented; latest executions not reverified | Check current jobs and test appropriate file restoration |
| Plex built-in DB backups | Files last verified 2026-10-26 | Establish and test database restore procedure |
| Plex full-appdata backup | Pending | Implement Phase 5 Wave 8 |
| Synology ABB | Pending | Implement and verify versioned backups |
| PBS alerts | Discord target test passed | Verify warning/error routing and real notification delivery |
| UPS-assisted recovery | NUT monitoring active on all three hosts | Test the actual guest/host shutdown and post-power-restoration sequence under an approved plan |

These items should remain open until the corresponding work has been completed and verified.

---

## Related Documentation

- `runbooks/backup-schedule.md` — Backup schedules, storage destinations, verification, and historical PBS incident.
- `runbooks/ai-observer-operations.md` — AI Observer health checks, maintenance, troubleshooting, and disaster recovery.
- `services/infra/ai-observer.md` — AI Observer architecture, configuration, security, and credential handling.
- `services/infra/authentik.md` — Authentik configuration, SSO, and November 14 migration.
- `architecture/compute/pve-prod-01.md` — Primary Proxmox node, primary Docker, primary DNS, and AI Observer.
- `architecture/compute/pve-prod-02.md` — Replacement Dell node, Immich, and VM 103 retirement history.
- `architecture/compute/pve-prod-03.md` — Lenovo third node, secondary DNS, and Authentik.
- `architecture/hardware/inventory.md` — Current hardware and workload placement.
- `architecture/network/ip-addresses.md` — Current service IP and VM host assignments.
- `architecture/storage/nas-prod-01.md` — NAS storage, PBS datastore location, and UPS server.
- `runbooks/git-workflow.md` — Git repositories, multi-host synchronization, and credential handling.
- `runbooks/updating-containers.md` — Container updates and maintenance procedures.
- `services/infra/dockman.md` — Docker management on docker-prod-01.
- `services/media/plex.md` — Plex deployment, persistent storage, and backup requirements.
- `phases/phase-5-hardening.md` — Planned backup and operational hardening work.

---

## Verification Summary — October 2026 (Historical)

| Item | Result |
|---|---|
| Nightly PBS backup job configured and enabled | Verified |
| October 29 PBS backups — both then-existing nodes | Failed |
| Immediate failure mechanism | Stale PBS proxy datastore `.lock` file descriptor (`ESTALE`) |
| Original NFS handle invalidation trigger | Unconfirmed; FUSE/NFS behavior suspected |
| Proxy restart, October 30 00:55 CDT | Completed; service active under new PID |
| Manual CT 100 backup, October 30 00:57 CDT | `TASK OK`; visible in PBS |
| CT 100 integrity verification, October 30 01:02 CDT | Passed with 0 errors |
| October 30 02:00 scheduled backups — pve-prod-01 | `TASK OK` (100, 101, 104, 106, 107) |
| October 30 02:00 scheduled backups — original pve-prod-02 | `TASK OK` (102, 105) |
| Integrity verification of other new snapshots | Not independently completed |
| Safe restore test, including AI Observer | Not performed |
| PBS VM 103 | Excluded; separate recovery strategy outstanding at that time |
| Discord alerts | Configured per operator; actual failure delivery not independently verified |
| NFS/FUSE mount/export changes | Not performed; operator elected monitoring |
| Docker rsync latest successful execution | Not reverified |
| Plex built-in database backup files | Last verified 2026-10-26 |
| Dedicated Plex full-appdata backup / Synology ABB | Not implemented |

**Historical note:** This October 30 state was superseded by the November Docker deployment after repeated stale-handle errors October 31–November 2.

### Verification Summary — November 5, 2026

| Item | Result |
|---|---|
| Docker PBS service | Running on nas-prod-01 with direct ZFS datastore access |
| Both PVE 02:00 scheduled backup tasks | `OK` (November 5) |
| CT 100 manual backup | `TASK OK` |
| CT 100 targeted integrity verification | Passed |
| CT 100 `/etc/hostname` file restore | Downloaded; operator confirmed correct content |
| Existing group ownership | All seven groups reassigned to new PVE token |
| Fresh Authentik login after secret rotation | Verified on PBS and both then-existing PVE hosts |
| Discord target | Webhook test delivered |
| Prune / GC / scheduled verify | Configured; scheduled execution not yet independently verified |
| Full guest restore | Not yet performed |
| Original VM 103 | Stopped and retained at the time; subsequently destroyed before November 13 |
| Image updates | Unraid showed no newer image; third-party latest lagged upstream PBS |

**Historical note:** VM 103's stopped/retained status describes November 5 only, not the current environment.

### Verification Summary — November 13, 2026

| Item | Result |
|---|---|
| Cluster architecture | Three Proxmox voting nodes, no external QDevice |
| Original pve-prod-02 | OptiPlex 3070 Micro retired |
| Replacement pve-prod-02 | Dell OptiPlex 5000 SFF |
| Third node | Lenovo ThinkCentre M90q Tiny |
| Old PBS VM 103 | Destroyed |
| Scheduled backups on nodes 01 and 03 | Successful |
| VM 105 placement | Moved from temporary node 03 to replacement node 02 |
| VM 105 manual backup on replacement node 02 | `TASK OK` |
| Full guest restore | Not performed |

### Verification Summary — November 14, 2026 (Current)

| Item | Result |
|---|---|
| Cluster | Three nodes online, quorate |
| Proxmox HA | Not enabled |
| PBS storage | `pbs-prod-01` available on all three Proxmox nodes |
| Scheduled backup job | Enabled, 02:00 daily, snapshot mode, all guests except nonexistent VM 103 |
| Guest backup selection | All seven current guests included |
| Scheduled pve-prod-01 backup | `TASK OK`; 100, 101, 104, 106, 107 |
| Scheduled replacement pve-prod-02 backup | `TASK OK`; VM 105 |
| Scheduled pve-prod-03 backup | `TASK OK`; CT 102 |
| Authentik VM 104 migration | Completed from node 01 to node 03 |
| Authentik manual backup from node 03 | `TASK OK` at 14:09:45 CST |
| VM 104 backup disks | 32 GiB system disk plus EFI disk |
| VM 107 backup | `TASK OK`; system and EFI disks |
| Former PBS VM 103 | Deleted; not available as rollback |
| Full guest restoration | Not performed |
| PBS prune, GC, and scheduled verification | Execution not independently reverified |
| Authentik rsync after migration | Latest successful execution not independently confirmed |
| Warning/error notification delivery | Not failure-tested |
| PBS Docker/datastore recovery | No end-to-end restoration test |

**Current assessment:** Backup execution is verified across the three-node Proxmox architecture, and Authentik has been successfully backed up from its new host. The principal remaining gap is recovery assurance: independent integrity checks, isolated restores, PBS maintenance validation, and notification testing.
