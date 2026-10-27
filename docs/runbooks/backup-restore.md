# Backup & Restore Runbook

**Last Updated:** 2026-10-26

Procedures for restoring from backup in the event of host failure, data loss, or corruption. See `backup-schedule.md` for what runs, when, and where backups are stored.

---

## Backup Types

|Type|Tool|What It Covers|Location|
|---|---|---|---|
|VM & CT snapshots|PBS|Full Proxmox VMs and LXCs|nas-prod-01 `/mnt/user/backups/pbs`|
|Stacks & appdata|rsync|Docker compose files, `.env` files, container appdata on Docker hosts|nas-prod-01 `/mnt/user/backups/stacks` + `appdata`|
|Plex built-in DB backup|Plex Scheduled Tasks|Plex library and blobs databases only|nas-prod-01 `/mnt/user/appdata/plex`|
|Plex full-appdata backup (pending)|Custom script planned|Complete Plex persistent appdata|NAS `backups` share — destination TBD|
|Versioned backup|Synology ABB (pending)|Entire `backups` share with point-in-time recovery|Synology NAS|

The dedicated Plex full-appdata backup and Synology ABB remain pending Phase 5 Wave 8 implementation.

---

## Restore — VM or LXC from PBS

Use this when a VM or LXC is lost, corrupted, or needs to be rolled back.

1. Log into Proxmox UI on pve-prod-01 or pve-prod-02 at `https://192.168.10.11:8006` or `https://192.168.10.12:8006`
    
2. In the left sidebar, select the node the VM/CT should be restored to
    
3. Click **Storage → pbs-prod-01 → nas-backups**
    
4. Find the backup you want to restore — select by date
    
5. Click **Restore**
    
6. Choose target node, storage, and VM/CT ID
    
7. Check **Start after restore** if you want it to boot
    
8. Click **Restore** and monitor the task log
    

> **Note:** PBS retention keeps Last 3, Daily 7, Weekly 4, Monthly 3. If the corruption went unnoticed for more than 7 days you may only have weekly snapshots available.

---

## Restore — Docker Stacks from rsync Backup

Use this when a compose file or `.env` file is lost or corrupted on a Docker host.

### Access the backup on the NAS

SSH into nas-prod-01:

```
ssh root@192.168.30.16
```

Navigate to the relevant backup:

```
ls /mnt/user/backups/stacks/<hostname>/
ls /mnt/user/backups/appdata/<hostname>/
```

### Copy a single file back to the host

From the Docker host (e.g. docker-prod-01), pull the file from the NAS:

```
sudo scp -i /root/.ssh/backup_rsa root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/<stack>/compose.yaml /opt/stacks/<stack>/compose.yaml
```

Or for an `.env` file:

```
sudo scp -i /root/.ssh/backup_rsa root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/<stack>/.env /opt/stacks/<stack>/.env
```

### Full stacks restore to a fresh host

If the entire host needs to be rebuilt:

```
sudo rsync -av -e "ssh -i /root/.ssh/backup_rsa" root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/ /opt/stacks/
sudo rsync -av -e "ssh -i /root/.ssh/backup_rsa" root@192.168.30.16:/mnt/user/backups/appdata/docker-prod-01/ /opt/appdata/
```

Then fix ownership:

```
sudo chown -R gio:service /opt/stacks
sudo find /opt/stacks -name "compose.yaml" -exec chmod 664 {} \;
sudo find /opt/stacks -name ".env" -exec chmod 660 {} \;
```

> **Note:** rsync backups are a rolling mirror — only the most recent state is stored. If files were deleted or corrupted before the last backup ran, the corrupted state is what's on the NAS. For point-in-time recovery, use Synology ABB once implemented in Wave 8.

---

## Restore — appdata from rsync Backup

Use this when container config or data under `/opt/appdata` is lost or corrupted.

Same procedure as stacks restore above but targeting the `appdata` path:

```
sudo rsync -av -e "ssh -i /root/.ssh/backup_rsa" root@192.168.30.16:/mnt/user/backups/appdata/docker-prod-01/ /opt/appdata/
```

Stop the affected container before restoring its appdata, then restart after:

```
# In Dockman or via CLI
docker compose -f /opt/stacks/<stack>/compose.yaml down
# restore appdata
docker compose -f /opt/stacks/<stack>/compose.yaml up -d
```

---

## Restore — Plex Media Server

**Status:** Database backups available; dedicated full-appdata restore procedure pending.

Plex runs directly on `nas-prod-01` under the Unraid Docker UI, not through Dockman or the Docker host rsync backup jobs.

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

Backup copies were verified for October 16, 19, 22, and 25, 2026.

These files can provide database recovery points, but do not constitute a full copy of Plex application data, artwork, metadata directories, and configuration.

**Restore precautions:**

- Do not overwrite the running Plex database while the container is active.
    
- Preserve the existing database files, including SQLite journal/WAL-related files, before attempting recovery.
    
- Use the Plex-supported database restoration procedure appropriate to the installed server version.
    
- Verify file ownership and permissions after restoration. Plex uses UID/GID `2000:2000` in this deployment.
    
- Verify library availability and watch history after any restore.
    

A complete, tested command-by-command database restoration procedure has not yet been established in this runbook. Do not treat the precautions above as a tested restoration procedure.

### Dedicated Full-Appdata Restore — Pending Phase 5 Wave 8

The custom full-appdata backup script has not been implemented.

During Wave 8:

1. Implement and verify the full-appdata backup.
    
2. Define an appropriate restore procedure for the selected archive format and backup layout.
    
3. Validate ownership, database integrity, Plex startup, library access, and user state.
    
4. Test restoration using a safe procedure that does not overwrite the production configuration unexpectedly.
    
5. Record the verified commands and results in this runbook.
    

See `services/media/plex.md` and `runbooks/backup-schedule.md` for current backup status.

---

## Verify Backup Health

### Check rsync logs

On each Docker host:

```
sudo cat /var/log/backup-appdata.log
```

Look for `Backup completed successfully` at the last entry. If you see `Backup FAILED` check the exit codes logged on the same line.

### Check Healthchecks.io

Log into healthchecks.io and verify all three checks are green:

- `backup-appdata-docker-prod-01`
    
- `backup-appdata-auth-prod-01`
    
- `backup-appdata-immich-prod-01`
    

### Check PBS backup status

Log into PBS at `https://192.168.30.12:8007` → **Datastore → nas-backups → Content** — confirm recent backups exist for all VMs and CTs.

### Check Plex built-in database backups

On nas-prod-01:

```
find "/mnt/user/appdata/plex/Library/Application Support/Plex Media Server/Plug-in Support/Databases" \
  -maxdepth 1 -type f \
  -printf '%TY-%Tm-%Td %TH:%TM | %s bytes | %f\n' \
  | sort -r | head -30
```

Verify that recent dated backup copies exist for both Plex databases.

This confirms file creation and schedule consistency, not database integrity or recoverability.

**Last verified:** 2026-10-26.

### Check Dedicated Plex Full-Appdata Backup

**Not yet applicable.** The dedicated backup is pending Phase 5 Wave 8.

Add backup-job logging, verification, and recovery validation procedures here after implementation.

---

## Troubleshooting

### rsync script not running

1. Check cron is configured: `sudo crontab -l` — should show `0 3 * * * /usr/local/bin/backup-appdata.sh`
    
2. Check the script exists and is executable: `ls -la /usr/local/bin/backup-appdata.sh`
    
3. Run manually and check output: `sudo /usr/local/bin/backup-appdata.sh`
    
4. Check the log: `sudo cat /var/log/backup-appdata.log`
    

### rsync SSH connection fails

1. Verify SSH key exists: `sudo ls -la /root/.ssh/backup_rsa`
    
2. Test SSH manually: `sudo ssh -i /root/.ssh/backup_rsa root@192.168.30.16 echo "test"`
    
3. Verify NAS authorized_keys: on nas-prod-01 run `cat /root/.ssh/authorized_keys` — confirm the host's public key is present
    

### PBS GC fails with permission error

Most likely cause: a folder owned by root exists in the PBS datastore path (`/mnt/backups/pbs`). PBS uid 34 cannot read root-owned files and aborts GC.

1. SSH into nas-prod-01
    
2. Run `ls -la /mnt/user/backups/pbs/` — check for any root-owned entries that don't belong
    
3. Remove or move any non-PBS files out of the `pbs/` subdirectory
    
4. Re-run GC from PBS UI
    

### Plex dedicated backup missing

The dedicated Plex full-appdata backup has not been implemented as of 2026-10-26.

Do not assume that the regular Docker host rsync backups include Plex, since Plex runs directly on Unraid.

The built-in Plex database backups are available, but they are stored inside `/mnt/user/appdata/plex` and are not independent full-appdata recovery copies.

Refer to Phase 5 Wave 8 before creating or troubleshooting the dedicated backup script.