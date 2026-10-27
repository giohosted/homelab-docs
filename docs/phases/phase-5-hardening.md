# Phase 5 — Hardening & Operational Readiness

**Status:** In Progress  
**Started:** 2026-03-17  
**Last Updated:** 2026-10-26

---

## Objectives

Harden the v3 infrastructure, establish operational tooling, configure monitoring, set up automated backups, and tune all services for optimal operation. This phase runs after all services are migrated and validated in Phase 4.

---

## Entry Criteria

- ✅ Phase 4 complete — all services running on v3 infrastructure
    
- ✅ Immich deployed and photos loading correctly
    
- ✅ Books stack deployed with hardlink workflows validated
    
- ✅ All DNS rewrites pointing at v3
    

---

## Wave 1 — Monitoring Stack ✅

### Beszel

Deployed Beszel agents on all 7 hosts — hub runs as a systemd binary service on pi-prod-01.

**Hosts with agents:**

- pve-prod-01
    
- pve-prod-02
    
- docker-prod-01
    
- auth-prod-01
    
- immich-prod-01
    
- nas-prod-01
    
- pi-prod-01 (local agent)
    

**Firewall rule added:**

- Source: VLAN 30 (192.168.30.0/24)
    
- Destination: 192.168.10.20 TCP port 45876
    
- Action: Allow
    

**OIDC:** Configured via Authentik — requires `email_verified: true` custom scope.

---

### Uptime Kuma

Running on pi-prod-01. 30 monitors configured (10 ping, 20 HTTP) covering all v3 hosts and services.

---

## Wave 2 — Backup Automation ✅

### Appdata rsync Script

Nightly rsync script backing up `/opt/appdata` and `/opt/stacks` from docker-prod-01, auth-prod-01, and immich-prod-01 to the NAS backups share.

**Implementation details:**

- Script location: `/usr/local/bin/backup-appdata.sh` on each host
    
- Log location: `/var/log/backup-appdata.log` on each host
    
- SSH key location: `/root/.ssh/backup_rsa` on each host (dedicated backup key, not interactive key)
    
- Schedule: cron at 03:00 nightly — edit via `sudo crontab -e` (root's crontab)
    
- To edit script: `sudo nano /usr/local/bin/backup-appdata.sh`
    
- To run manually: `sudo /usr/local/bin/backup-appdata.sh`
    
- Transport: SSH rsync to nas-prod-01 at `192.168.30.16` (VLAN 30 data interface)
    
- Authorized keys on NAS: `/root/.ssh/authorized_keys` — one entry per host
    

**What is backed up:**

- `/opt/stacks` → `nas-prod-01:/mnt/user/backups/stacks/<hostname>/` (includes `.env` files)
    
- `/opt/appdata` → `nas-prod-01:/mnt/user/backups/appdata/<hostname>/`
    
- Behavior: rolling mirror with `--delete` — destination stays in sync with source, no versioning
    

**Hosts and Healthchecks.io URLs:**

|Host|Healthchecks.io Check|
|---|---|
|docker-prod-01|`https://hc-ping.com/132bdc56-2f0e-45b8-85a8-c07dc1c049ab`|
|auth-prod-01|`https://hc-ping.com/7c625181-5347-4ce5-ab5a-385b72201d91`|
|immich-prod-01|`https://hc-ping.com/1329d9f0-9416-4d81-935c-23ce2969c1a6`|

---

### PBS Backup

PBS 4.1.4 on pbs-prod-01 (192.168.30.12). Datastore `nas-backups` mounted via NFS at `/mnt/backups/pbs` (NAS path: `/mnt/user/backups/pbs`).

**Backup job:** Daily at 02:00, all VMs and CTs, both nodes.

**Retention (prune job):**

- Keep Last: 3
    
- Keep Daily: 7
    
- Keep Weekly: 4
    
- Keep Monthly: 3
    

**Schedules:**

- Prune: daily
    
- GC: every Saturday at 04:00
    
- Verify: every Saturday at 05:00
    

**NAS share structure:**

```
/mnt/user/backups/
├── pbs/                    ← PBS datastore (owned by uid 34)
├── appdata/                ← rsync appdata backups (owned by root)
│   ├── docker-prod-01/
│   ├── auth-prod-01/
│   └── immich-prod-01/
└── stacks/                 ← rsync stacks backups (owned by root)
    ├── docker-prod-01/
    ├── auth-prod-01/
    └── immich-prod-01/
```

> **Important:** PBS datastore must be scoped to `/mnt/backups/pbs` — not the share root `/mnt/backups`. PBS GC will fail with permission errors if it sees the `appdata` and `stacks` folders owned by root.

---

## Wave 3 — OIDC Rollout ✅

|Service|Status|Notes|
|---|---|---|
|Proxmox (both nodes + PBS)|✅ Done|OIDC via Authentik — both pve-prod-01 and pve-prod-02|
|Immich|✅ Done|Carried forward from v2|
|Beszel|✅ Done|Requires `email_verified: true` custom scope in Authentik|
|Synology DSM|✅ Done|Local admin retained as break-glass|
|Audiobookshelf|✅ Done|Carried forward from v2|
|Calibre-Web-Automated|✅ Done|Carried forward from v2|
|Shelfmark|✅ Done|New provider created in v3|
|qBitrr|✅ Done|New provider created in v3|

---

## Wave 4 — External Access & CF Access ✅

### Immich — Temporary Tunnel for Wedding (October)

Immich is LAN-only by default. The tunnel route for `photos.giohosted.com` will be added temporarily the day before the wedding and removed a few days after. No CF Access policy — guests access the shared album directly via QR code, protected by Immich's built-in album password.

**Traefik file provider route** already configured at `/opt/appdata/traefik/config/immich.yml` and verified working on LAN.

**October checklist (day before wedding):**

1. Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Add public hostname:
    
    - Subdomain: `photos` / Domain: `giohosted.com`
        
    - Type: `HTTPS` / URL: `192.168.30.11`
        
    - Additional settings → TLS → No TLS Verify: enabled
        
2. Test QR code link from phone on LTE
    
3. Share QR code with guests
    

**After the wedding (a few days later):**

1. Remove the `photos.giohosted.com` route from the tunnel
    
2. Immich returns to LAN-only
    

> Immich server external domain confirmed set to `https://photos.giohosted.com` in Administration → Server Settings.

---

### ABS Mobile App — CF Access Fix ✅

Fixed using a CF Access service token + registering `audiobooth://oauth` in both Authentik and ABS.

**What was done:**

1. Created CF Access service token `abs-mobile` (Access controls → Service credentials)
    
2. Added `abs-mobile-token` policy (action: SERVICE AUTH) to the Audiobookshelf CF Access application — order 1, evaluated before all other policies
    
3. Added `audiobooth://oauth` to ABS server settings → Authentication → OpenID Connect → Mobile Redirect URIs
    
4. Configured ABS mobile app with two custom headers: `CF-Access-Client-Id` and `CF-Access-Client-Secret`
    

See `audiobookshelf.md` for full details.

---

## Wave 5 — MAM Seeding Rules ✅

Configured qBitrr tracker-scoped seeding rules for MAM ebooks and audiobooks.

**Rules applied:** 14-day minimum seed time, 1.0 minimum ratio, auto-remove on completion. Scoped to `myanonamouse.net` tracker only — AudiobookBay and other sources unaffected.

**Configuration location:** `/opt/appdata/qbitrr/config.toml` — `[BitTorrent.CategorySeeding.Trackers.myanonamouse]`

See `qbitrr.md` for full details.

---

## Wave 6 — Service Settings Review ✅

All priority services reviewed and tuned. Summary of changes made:

|Service|Changes|
|---|---|
|qBitrr|`ReSearchStalled = true` on all 4 instances; MAM tracker seeding block added|
|Immich|Facial recognition model changed `buffalo_l` → `buffalo_s`; storage template enabled `{{y}}/{{MMMM}}/{{filename}}`|
|AdGuard|Hagezi Threat Intelligence Feeds blocklist added|
|Traefik|No changes needed — already solid|
|ARR stack|No changes needed — Profilarr manages quality profiles, naming, and custom formats from TRaSH|
|Plex|Built-in database backup every 3 days enabled in Scheduled Tasks; verified operational 2026-10-26|
|PBS|Covered in Wave 2 — no additional changes needed|

**Plex verification (2026-10-26):**

Dated library and blobs database backup pairs were found for October 16, 19, 22, and 25, created during Plex's scheduled maintenance window.

This confirms the built-in database backup is operating. It does not mean a dedicated full-appdata backup has been implemented.

The dedicated full-appdata backup remains a separate Wave 8 task.

---

## Wave 7 — Security Hardening

### SSH Key-Based Authentication

Set up SSH key-based auth on all hosts and disable password auth:

|Host|Status|
|---|---|
|docker-prod-01|⬜ Pending|
|auth-prod-01|⬜ Pending|
|immich-prod-01|⬜ Pending|
|pve-prod-01|⬜ Pending|
|pve-prod-02|⬜ Pending|
|pi-prod-01|⬜ Pending|
|nas-prod-01|⬜ Pending|

### NUT Client on auth-prod-01 and immich-prod-01

NUT clients not configured on auth-prod-01 or immich-prod-01 during Phase 4. Add during Phase 5:

- Install `nut-client` on both VMs
    
- Configure `/etc/nut/nut.conf` and `/etc/nut/upsmon.conf`
    
- Monitor `ups@192.168.10.10`
    

### Firewall Rule Review

Review all UDM-SE firewall rules — confirm no overly permissive rules remain from migration period.

### v2 Decommission

Decommission v2 infrastructure:

- Confirm all DNS rewrites pointing at v3
    
- Shut down v2 Docker host (192.168.20.10)
    
- Shut down v2 Proxmox host (192.168.20.2)
    
- Reclaim VLAN 20 IPs
    

---

## Wave 8 — Synology Active Backup for Business & Plex Backup ⬜

**Status:** Pending

Wave 8 will establish versioned off-host backups and complete dedicated backup protection for Plex running directly on Unraid.

### Synology Active Backup for Business

Configure Synology DSM Active Backup for Business (ABB) to pull versioned backups from nas-prod-01.

**Goal:** Provide versioned, point-in-time recovery on top of the rolling rsync mirror. The rsync scripts are a same-day mirror — ABB adds historical versions.

**Scope:**

- Source: nas-prod-01 `backups` share (`/mnt/user/backups`)
    
- Destination: Synology NAS dedicated backup volume
    
- Schedule: TBD — nightly after source backups complete
    
- Retention: TBD
    

**Status:** Pending implementation and verification.

### Dedicated Plex Full-Appdata Backup

Plex runs directly on nas-prod-01 and is not covered by the rsync jobs implemented on docker-prod-01, auth-prod-01, and immich-prod-01.

Plex's built-in database backup is already enabled and working, but it remains within the live appdata directory. A separate backup solution is required for full-appdata recovery.

**Implementation approach:**

- Use a custom backup script on nas-prod-01.
    
- Do not use the existing Unraid Appdata Backup plugin as the preferred solution.
    
- Protect the entire `/mnt/user/appdata/plex` persistent directory.
    
- Store backups on the NAS `backups` share, backed by the ZFS mirror.
    
- Ensure backups are consistent by safely handling the Plex container and databases.
    
- Restart Plex reliably after the backup, including on failure.
    
- Configure scheduling, retention, logging, and monitoring.
    
- Coordinate with existing ZFS snapshots and the planned Synology ABB backup workflow.
    
- Document and test restoration before marking the solution complete.
    

**Exact schedule, destination layout, archive format, retention, and script implementation:** TBD during Wave 8.

### Pre-Implementation Investigation — 2026-10-26

A discrepancy was identified between `services/media/plex.md` and this Phase 5 plan.

The Plex service documentation previously stated that a dedicated stop–backup–restart script had been implemented. Live investigation did not support that statement.

**Findings:**

- Plex appdata size: 6.8G
    
- ZFS mirror: 3.62T total, 3.41T free, ONLINE
    
- Built-in Plex database backups: verified every three days
    
- Documented dedicated backup path `/mnt/user/backups/plex/db/`: absent
    
- Root crontab: no Plex-specific entries found
    
- Unraid User Scripts directory: absent
    
- Dedicated Plex backup script: not found in searched common script locations
    
- Plex had been continuously running since September 5 before its October 26 update, inconsistent with the documented periodic stop–backup–restart procedure
    

**Conclusion:** The dedicated Plex full-appdata backup remains unimplemented. The prior service documentation incorrectly described planned work as completed.

**Decision:** Defer implementation until Wave 8 and use a custom script.

No new backup automation was deployed during this investigation.

### Wave 8 Completion Criteria

- ⬜ Configure Synology ABB for versioned backups
    
- ⬜ Verify ABB scheduling, retention, and restore capability
    
- ⬜ Implement dedicated Plex full-appdata backup script
    
- ⬜ Configure and verify Plex backup scheduling
    
- ⬜ Confirm full-appdata backup integrity
    
- ⬜ Establish and test Plex restoration procedure
    
- ⬜ Update Plex service documentation and backup runbooks with final verified configuration
    

---

## Exit Criteria

- ✅ Beszel agents deployed on all hosts — metrics visible
    
- ✅ Uptime Kuma monitors configured for all services
    
- ✅ Appdata rsync script running nightly with Healthchecks.io heartbeats
    
- ✅ PBS backup settings reviewed and optimized
    
- ✅ OIDC configured for Proxmox, Beszel, Synology
    
- ✅ Immich external access planned — temporary tunnel for wedding (October), no permanent exposure
    
- ✅ ABS mobile app CF Access issue resolved — service token + `audiobooth://oauth` redirect URI
    
- ✅ MAM seeding rules configured in qBitrr
    
- ✅ Service settings review complete for all priority services
    
- ✅ Plex built-in database backups verified operational
    
- ⬜ SSH key-based auth on all hosts
    
- ⬜ NUT clients on auth-prod-01 and immich-prod-01
    
- ⬜ v2 infrastructure decommissioned
    
- ⬜ Firewall rules reviewed and cleaned up
    
- ⬜ Synology ABB configured for versioned backups
    
- ⬜ Dedicated Plex full-appdata backup implemented and restore-tested