# Backup & Restore Runbook

**Last Updated:** 2026-03-18

Procedures for restoring from backup in the event of host failure, data loss, or corruption. See `backup-schedule.md` for what runs, when, and where backups are stored.

---

## Backup Types

| Type | Tool | What It Covers | Location |
|------|------|---------------|----------|
| VM & CT snapshots | PBS | Full Proxmox VMs and LXCs | nas-prod-01 `/mnt/user/backups/pbs` |
| Stacks & appdata | rsync | Docker compose files, `.env` files, container appdata | nas-prod-01 `/mnt/user/backups/stacks` + `appdata` |
| Versioned backup | Synology ABB (pending) | Entire `backups` share with point-in-time recovery | Synology NAS |

---

## Restore — VM or LXC from PBS

Use this when a VM or LXC is lost, corrupted, or needs to be rolled back.

1. Log into Proxmox UI on pve-prod-01 or pve-prod-02 at `https://192.168.10.11:8006` or `https://192.168.10.12:8006`
2. In the left sidebar, select the node the VM/CT should be restored to
3. Click **Storage → pbs-prod-01 → nas-backups**
4. Find the backup you want to restore — select by date
5. Click **Restore**
6. Choose target node, storage, and VM/CT ID
7. Check **Start after restore** if you want it to boot immediately
8. Click **Restore** and monitor the task log

> **Note:** PBS retention keeps Last 3, Daily 7, Weekly 4, Monthly 3. If the corruption went unnoticed for more than 7 days you may only have weekly snapshots available.

---

## Restore — Docker Stacks from rsync Backup

Use this when a compose file or `.env` file is lost or corrupted on a Docker host.

### Access the backup on the NAS

SSH into nas-prod-01:
```bash
ssh root@192.168.30.16
```

Navigate to the relevant backup:
```bash
ls /mnt/user/backups/stacks/<hostname>/
ls /mnt/user/backups/appdata/<hostname>/
```

### Copy a single file back to the host

From the Docker host (e.g. docker-prod-01), pull the file from the NAS:
```bash
sudo scp -i /root/.ssh/backup_rsa root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/<stack>/compose.yaml /opt/stacks/<stack>/compose.yaml
```

Or for an `.env` file:
```bash
sudo scp -i /root/.ssh/backup_rsa root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/<stack>/.env /opt/stacks/<stack>/.env
```

### Full stacks restore to a fresh host

If the entire host needs to be rebuilt:
```bash
sudo rsync -av -e "ssh -i /root/.ssh/backup_rsa" root@192.168.30.16:/mnt/user/backups/stacks/docker-prod-01/ /opt/stacks/
sudo rsync -av -e "ssh -i /root/.ssh/backup_rsa" root@192.168.30.16:/mnt/user/backups/appdata/docker-prod-01/ /opt/appdata/
```

Then fix ownership:
```bash
sudo chown -R gio:service /opt/stacks
sudo find /opt/stacks -name "compose.yaml" -exec chmod 664 {} \;
sudo find /opt/stacks -name ".env" -exec chmod 660 {} \;
```

> **Note:** rsync backups are a rolling mirror — only the most recent state is stored. If files were deleted or corrupted before the last backup ran, the corrupted state is what's on the NAS. For point-in-time recovery, use Synology ABB (Wave 8).

---

## Restore — appdata from rsync Backup

Use this when container config or data under `/opt/appdata` is lost or corrupted.

Same procedure as stacks restore above but targeting the `appdata` path:
```bash
sudo rsync -av -e "ssh -i /root/.ssh/backup_rsa" root@192.168.30.16:/mnt/user/backups/appdata/docker-prod-01/ /opt/appdata/
```

Stop the affected container before restoring its appdata, then restart after:
```bash
# In Dockman or via CLI
docker compose -f /opt/stacks/<stack>/compose.yaml down
# restore appdata
docker compose -f /opt/stacks/<stack>/compose.yaml up -d
```

---

## Verify Backup Health

### Check rsync logs
On each Docker host:
```bash
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