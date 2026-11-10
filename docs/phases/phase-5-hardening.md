# Phase 5 — Hardening & Operational Readiness

**Status:** In Progress  
**Started:** 2026-03-17  
**Last Updated:** 2026-11-09

---

## Objectives

Harden the v3 infrastructure, establish operational tooling, configure monitoring, set up automated backups, and tune all services for optimal operation. This phase runs after all services are migrated and validated in Phase 4.

**Documentation convention:** A completed configuration task is not necessarily equivalent to verified end-to-end functionality. Implemented settings, observed operational behavior, and outstanding verification are distinguished throughout this document.

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

| Host | Healthchecks.io Check |
|------|----------------------|
| docker-prod-01 | `https://hc-ping.com/132bdc56-2f0e-45b8-85a8-c07dc1c049ab` |
| auth-prod-01 | `https://hc-ping.com/7c625181-5347-4ce5-ab5a-385b72201d91` |
| immich-prod-01 | `https://hc-ping.com/1329d9f0-9416-4d81-935c-23ce2969c1a6` |

**Security note:** Healthchecks.io ping URLs function as monitoring credentials. Keep this documentation private and do not publish the URLs.

---

### PBS Backup

PBS 4.1.4 on pbs-prod-01 (192.168.30.12) at the time of the original Phase 5 setup. Datastore `nas-backups` mounted via NFS at `/mnt/backups/pbs` (NAS path: `/mnt/user/backups/pbs`).

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

```text
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

**Verification boundary:** This section preserves the originally documented backup implementation and schedules. The November 2026 Qbitrr audit did not independently revalidate PBS backup completion, datastore integrity, or restore capability. Those must be evaluated separately using current PBS operational evidence.

---

## Wave 3 — OIDC Rollout ✅

| Service | Status | Notes |
|---------|--------|-------|
| Proxmox (both nodes + PBS) | ✅ Done | OIDC via Authentik — both pve-prod-01 and pve-prod-02 |
| Immich | ✅ Done | Carried forward from v2 |
| Beszel | ✅ Done | Requires `email_verified: true` custom scope in Authentik |
| Synology DSM | ✅ Done | Local admin retained as break-glass |
| Audiobookshelf | ✅ Done | Carried forward from v2 |
| Calibre-Web-Automated | ✅ Done | Carried forward from v2 |
| Shelfmark | ✅ Done | New provider created in v3 |
| qBitrr | ✅ Done | New provider created in v3 |

---

## Wave 4 — External Access & CF Access ✅

### Immich — Temporary Tunnel for Wedding (October)

The original October 2026 plan was to keep Immich LAN-only by default and expose `photos.giohosted.com` temporarily for wedding guests.

The planned temporary route would be added the day before the wedding and removed several days afterward. No Cloudflare Access policy would be required for guests; they would access a shared album through its QR code, protected by Immich's built-in album password.

**Traefik file provider route:** `/opt/appdata/traefik/config/immich.yml`

The route had already been configured and verified on LAN.

**Original October checklist (day before wedding):**

1. Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Add public hostname:
   - Subdomain: `photos`
   - Domain: `giohosted.com`
   - Type: `HTTPS`
   - URL: `192.168.30.11`
   - Additional settings → TLS → No TLS Verify: enabled
2. Test QR code link from phone on LTE.
3. Share QR code with guests.

**Original post-wedding plan:**

1. Remove the `photos.giohosted.com` route from the tunnel.
2. Return Immich to LAN-only operation.

Immich's server external domain was documented as `https://photos.giohosted.com` in Administration → Server Settings.

**November 2026 documentation note:** The wedding access procedure above is retained as a historical operational plan. This Qbitrr audit did not establish whether the temporary Cloudflare Tunnel route was subsequently removed. Verify current Cloudflare Tunnel configuration before assuming Immich is LAN-only.

---

### ABS Mobile App — CF Access Fix ✅

Fixed using a CF Access service token and registering `audiobooth://oauth` in both Authentik and ABS.

**What was done:**

1. Created CF Access service token `abs-mobile` under Access controls → Service credentials.
2. Added `abs-mobile-token` policy (action: SERVICE AUTH) to the Audiobookshelf CF Access application — order 1, evaluated before other policies.
3. Added `audiobooth://oauth` to ABS server settings → Authentication → OpenID Connect → Mobile Redirect URIs.
4. Configured the ABS mobile app with two custom headers:
   - `CF-Access-Client-Id`
   - `CF-Access-Client-Secret`

See `services/books/audiobookshelf.md` for full details.

---

## Wave 5 — MAM Seeding Rules ✅ Configured

**Configuration status:** Complete  
**End-to-end MAM verification:** Pending

qBitrr manages the `ebooks` and `audiobooks` qBittorrent categories and applies a tracker-specific policy intended to enforce MyAnonamouse requirements.

### Configuration History

The original Phase 5 implementation documented MAM seeding rules with:

```text
[BitTorrent.CategorySeeding.Trackers.myanonamouse]
```

That reference is obsolete for the current deployed configuration.

During the November 2026 Qbitrr audit, the tracker configuration was migrated to the `[[qBit.Trackers]]` structure.

**Current configuration file:**

```text
/opt/appdata/qbitrr/config.toml
```

### Managed Book Categories

The active configuration contains:

```toml
ManagedCategories = ["audiobooks", "ebooks"]
```

The category-level default policy is:

```toml
[qBit.CategorySeeding]
MaxUploadRatio = -1
MaxSeedingTime = 3600
RemoveTorrent = 2
```

This configures time-based cleanup after approximately one hour of seeding for managed public book torrents.

The previous indefinite-seeding defaults were replaced during the November audit.

### MyAnonamouse Tracker Configuration

The following settings were verified directly from `config.toml`:

```toml
[[qBit.Trackers]]
Name = "MyAnonamouse"
URI = "myanonamouse.net"
Priority = 10
MaxUploadRatio = 1.0
MaxSeedingTime = 1209600
HitAndRunMode = "and"
MinSeedRatio = 1.0
MinSeedingTimeDays = 14
TrackerUpdateBuffer = 3600
AddTrackerIfMissing = false
RemoveIfExists = false
```

**Intended policy:**

- Match torrents associated with `myanonamouse.net`.
- Require at least 14 days of seeding.
- Require a minimum upload ratio of 1.0.
- Use `HitAndRunMode = "and"` so both requirements must be satisfied.
- Account for possible tracker reporting delays using `TrackerUpdateBuffer = 3600`.
- Avoid automatically adding or removing tracker URLs.

The configured 14-day duration is expressed as 1,209,600 seconds.

### qBittorrent Global Seeding Limits

The old native qBittorrent global 60-minute seeding stop rule was disabled during the November audit.

This prevents a global stop condition from interfering with the intended tracker-specific MAM protection.

Seeding policy management is centralized in qBitrr.

### Verification Status

| Item | Status | Evidence |
|------|--------|----------|
| `config.toml` parses successfully | ✅ Verified | TOML validation |
| MAM tracker definition exists | ✅ Verified | Direct configuration inspection |
| Qbitrr workers load after restart | ✅ Verified | Successful worker initialization |
| Public audiobook cleanup | ✅ Verified | Real AudiobookBay torrent |
| Library media retained after cleanup | ✅ Verified | ABS playback after torrent removal |
| MAM tracker matches a real torrent | ⬜ Pending | No suitable live MAM torrent observed |
| MAM protected beyond one hour | ⬜ Pending | Requires live tracker test |
| MAM cleared only after time AND ratio conditions | ⬜ Pending | Requires real-world confirmation |

**Important:** Configuration validity does not prove private-tracker compliance.

Until a live MAM torrent is used to confirm matching and retention behavior, the policy must be considered configured but not fully verified.

Do not manually remove MAM torrents before confirming the applicable tracker obligations have been satisfied.

### Future Validation

When a suitable MAM torrent is available:

1. Confirm that qBittorrent identifies the expected MAM tracker URL.
2. Confirm that qBitrr recognizes the tracker override.
3. Verify that the torrent remains protected beyond the default 60-minute public-torrent threshold.
4. Monitor seeding duration and ratio as reported by both qBittorrent and MAM.
5. Confirm that removal is not permitted before both configured minimum requirements have been satisfied.

A full 14-day test does not need to block other Phase 5 activities.

**Primary reference:** `services/torrent/qbitrr.md`

**Related references:**

- `services/torrent/qbittorrent.md`
- `services/books/shelfmark.md`

---

## Wave 6 — Service Settings Review ✅

All priority services were reviewed during the original Phase 5 configuration review.

The November 2026 Qbitrr audit added further configuration validation, troubleshooting, performance assessment, and operational verification.

### Original Service Review

| Service | Changes |
|---------|---------|
| qBitrr | `ReSearchStalled = true` configured on all four ARR instances; initial MAM tracker seeding block added |
| Immich | Facial recognition model changed `buffalo_l` → `buffalo_s`; storage template enabled `{{y}}/{{MMMM}}/{{filename}}` |
| AdGuard | Hagezi Threat Intelligence Feeds blocklist added |
| Traefik | No changes needed during the original review |
| ARR stack | No changes needed during the original review — Profilarr manages quality profiles, naming, and custom formats from TRaSH |
| Plex | Built-in database backup every three days enabled in Scheduled Tasks; verified operational 2026-10-26 |
| PBS | Covered in Wave 2 — no additional settings changes documented during the original review |

### Plex Verification — October 26, 2026

Dated library and blobs database backup pairs were found for October 16, 19, 22, and 25, created during Plex's scheduled maintenance window.

This confirmed the built-in database backup was operating at the time of inspection.

It did not mean a dedicated full-appdata backup had been implemented.

The dedicated full-appdata backup remains a separate Wave 8 task.

---

### November 2026 — qBitrr Configuration and Reliability Audit

**Audit period:** November 7–9, 2026  
**Host:** docker-prod-01  
**Configuration:** `/opt/appdata/qbitrr/config.toml`

**Verified software versions:**

| Service | Version |
|---------|---------|
| qBitrr | 5.14.9-1 |
| qBittorrent | 5.3.0 |

The review covered:

1. Automated searching and monitoring
2. Torrent health and recovery
3. Seeding policies and tracker requirements
4. Incorrect and orphaned torrent records
5. Performance and reliability
6. Configuration and documentation consistency

#### Automated Searching and Monitoring

All four ARR instances were confirmed with the following search settings:

| Setting | Value |
|---------|-------|
| `SearchMissing` | `true` |
| `SearchLimit` | `5` |
| `SearchRequestsEvery` | `300` |
| `SearchAgainOnSearchCompletion` | `true` |
| `SearchByYear` | `true` |
| `SearchInReverse` | `false` |
| `Unmonitored` | `false` |
| `DoUpgradeSearch` | `false` |
| `QualityUnmetSearch` | `false` |
| `CustomFormatUnmetSearch` | `false` |
| `RefreshDownloadsTimer` | `1` |
| `ReSearch` | `true` |

Sonarr-specific settings:

- `SearchBySeries = "smart"`
- `PrioritizeTodaysReleases = true`

Global settings:

- `LoopSleepTimer = 5`
- `SearchLoopDelay = -1`

The observed search-loop logs included 30-second pacing messages, which were consistent with the configured default search delay.

The review did not establish that excessive repeated indexer searches were occurring.

**Decision:** Retain existing search and polling settings. Automated searching is intentional, and no performance evidence justified reducing its frequency.

#### Sonarr-Anime Monitoring

Earlier unwanted anime downloads were traced to monitoring settings.

Sonarr-Anime had content that was not intended to remain monitored, including individual episodes whose monitoring state required attention.

Affected content was unmonitored as appropriate.

`SearchMissing = true` remained enabled for other legitimately monitored anime.

**Decision:** Preserve automated searching while controlling eligibility through Sonarr's monitoring state.

#### Torrent Health and Recovery

The configuration was reviewed across all four ARR instances.

Key settings included:

```toml
ReSearchStalled = true
StalledDelay = 15
DoNotRemoveSlow = true
IgnoreTorrentsYoungerThan = 180
MaximumDeletablePercentage = 0.99
AutoDelete = false
```

A stalled Radarr-1080 torrent for *Dracula Untold* was repeatedly detected but could not be removed under the previous `RemoveTorrent = -1` setting.

The logs explicitly identified the removal policy as the blocker.

The seeding configuration was corrected to use `RemoveTorrent = 2`.

Separately, invalid or unavailable Sonarr-TV episode torrents were observed being removed and blocklisted.

**Verified:**

- Stalled-torrent detection occurred.
- The original removal-policy restriction was identified.
- The configuration was corrected.
- Invalid torrent removal and blocklisting occurred.

**Pending verification:**

- A newly stalled torrent successfully completing the full detection, removal, and replacement-download workflow under the corrected policy.

#### File Filtering and Regex Correction

All four ARR configurations use an extension allowlist to restrict eligible file types.

The verified allowlist includes:

```toml
FileExtensionAllowlist = [".mp4", ".mkv", ".sub", ".ass", ".srt", ".!qB", ".parts"]
```

Executable files such as `.exe` are excluded from the allowlist.

Folder and filename exclusions cover samples and unwanted extras.

During the audit, 12 incorrectly escaped regular expressions were identified and corrected across the four ARR instances.

The affected expressions included numbered anime opening and ending extras such as:

- `NCOP12`
- `NCED05`

The corrected expressions were validated with 24 tests.

**Result:** All 24 validation tests passed.

The configuration was then loaded through a Qbitrr container restart, and worker startup was verified.

A backup of the pre-correction configuration was retained under:

```text
/opt/appdata/qbitrr/config.toml.pre-regex-fix-20261110-014955
```

#### Database Orphan Cleanup

The audit identified 384 stale Sonarr-Anime episode records in Qbitrr's SQLite database.

These included records associated with previously removed or no-longer-valid media entries.

A targeted cleanup was performed after taking a database backup:

```text
/opt/appdata/qbitrr/backups/qbitrr-pre-cleanup-20261108T233539Z.db
```

Following cleanup:

- The targeted 384 stale entries were removed.
- The Sonarr-Anime orphan count was reduced to zero.
- Existing Sonarr-TV records were preserved.
- SQLite integrity was verified.

Sixteen leftover Bleach media files, totaling approximately 17 GB, were also identified and removed after explicit authorization.

**Limitation:** The underlying cause of the stale search or tracking records was not conclusively established. Monitor for recurrence.

#### Public Audiobook Seeding Verification

A public AudiobookBay torrent for *Threshing Day* was used to verify the audiobook workflow.

The test confirmed:

1. Shelfmark successfully submitted the torrent.
2. qBittorrent downloaded and seeded the audiobook.
3. Shelfmark imported the audiobook into Audiobookshelf.
4. Audiobookshelf successfully played the file.
5. qBitrr automatically removed the torrent after the configured one-hour seeding period.
6. The imported audiobook remained present and playable after torrent removal.

**Result:** Public audiobook importing, torrent cleanup, and library retention were verified end-to-end.

This test does not verify MAM private-tracker compliance.

#### Shelfmark NFS Incident

During the audit, Shelfmark encountered failed ebook and audiobook downloads.

Investigation found that Shelfmark's mounted directories returned:

```text
Stale file handle
```

The host's `/data` NFS mount remained accessible, and other containers continued to access their directories successfully.

The issue was isolated to Shelfmark's container-visible directory handles.

Restarting only Shelfmark restored access.

Subsequent tests confirmed:

- Direct EPUB download into CWA succeeded.
- A public audiobook torrent downloaded successfully.
- ABS imported and played the audiobook.
- qBitrr cleanup did not remove the imported audiobook.

**Decision:** Treat the incident as resolved. Do not introduce additional NFS architecture changes or automatic restart mechanisms based on a single occurrence.

The exact initiating cause remains unknown.

See `services/books/shelfmark.md` for details.

#### Performance and Reliability

The following Qbitrr measurements were collected during the audit:

| Metric | Observed Value |
|--------|----------------|
| CPU usage | 3.45% |
| Memory usage | 368.5 MiB |
| Memory limit reported by Docker | 7.736 GiB |
| Process/thread count | 36 |
| Restart policy | `unless-stopped` |
| Container status | Running |
| OOM killed | `false` |
| Docker healthcheck | Not configured |
| Docker log driver | `json-file` |
| Explicit log rotation options | None |
| Docker log file size | Approximately 7.3 MB |

The reported CPU and memory usage were reasonable for the current workload.

All 13 Qbitrr worker processes started successfully after the final configuration restart.

The separate qBit category manager was also initialized.

**Decision:** No CPU, memory, worker-count, or polling changes were needed.

#### SQLite Maintenance

The active database is located at:

```text
/opt/appdata/qbitrr/qBitManager/qbitrr.db
```

The database files observed during the review were approximately:

| File | Size |
|------|------|
| `qbitrr.db` | 790,528 bytes |
| `qbitrr.db-wal` | 144,232 bytes |
| `qbitrr.db-shm` | 32,768 bytes |

Qbitrr runs periodic WAL checkpoint maintenance every five minutes.

The supplied logs showed successful checkpoints without reported checkpoint errors.

A final read-only SQLite integrity check returned:

```text
SQLite integrity: ok
```

**Decision:** Leave database maintenance intervals unchanged. Do not manually delete live WAL or SHM files.

#### Docker Log Rotation

Qbitrr currently uses Docker's `json-file` logging driver without explicit per-container log rotation.

The log measured approximately 7.3 MB.

No immediate disk-space concern was identified.

**Optional future improvement:** Consider a bounded log policy, such as:

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

This is a proposed example, not an implemented setting.

Apply only during planned container maintenance after evaluating the broader Docker logging configuration.

#### November Audit Outcome

| Area | Result |
|------|--------|
| Automated search and monitoring | Reviewed; unnecessary monitoring corrected |
| Torrent health and recovery | Configuration corrected; new stalled-recovery test pending |
| Seeding policies | Configured; public audiobook verified |
| MAM private-tracker protection | Configuration verified; live test pending |
| Orphan database records | Cleaned and verified |
| File-filtering patterns | Corrected; 24/24 tests passed |
| Shelfmark NFS issue | Resolved; downstream workflows verified |
| Container performance | Healthy snapshot; no tuning needed |
| Worker startup | Verified |
| SQLite integrity | Verified |
| Database checkpoint maintenance | Operating normally |
| Docker log rotation | Optional future improvement |
| Documentation reconciliation | Updated November 2026 |

**Primary operational documentation:**

- `services/torrent/qbitrr.md`
- `services/torrent/qbittorrent.md`
- `services/books/shelfmark.md`

The phase document summarizes the review. Detailed configuration values, commands, and troubleshooting procedures are maintained in the relevant service documentation to avoid unnecessary duplication.

---

## Wave 7 — Security Hardening

### SSH Key-Based Authentication

Set up SSH key-based authentication on all hosts and disable password authentication after verified key-based access and recovery procedures are available.

| Host | Status |
|------|--------|
| docker-prod-01 | ⬜ Pending |
| auth-prod-01 | ⬜ Pending |
| immich-prod-01 | ⬜ Pending |
| pve-prod-01 | ⬜ Pending |
| pve-prod-02 | ⬜ Pending |
| pi-prod-01 | ⬜ Pending |
| nas-prod-01 | ⬜ Pending |

These statuses are retained from the existing Phase 5 documentation and were not independently reassessed during the November Qbitrr audit.

**Security precaution:** Verify an alternative administrative access path before disabling password-based SSH authentication on any host.

### NUT Client on auth-prod-01 and immich-prod-01

NUT clients were not configured on auth-prod-01 or immich-prod-01 during Phase 4.

Planned Phase 5 work:

- Install `nut-client` on both VMs.
- Configure `/etc/nut/nut.conf` and `/etc/nut/upsmon.conf`.
- Monitor `ups@192.168.10.10`.
- Verify behavior and shutdown dependencies before enabling automated responses.

### Firewall Rule Review

Review all UDM-SE firewall rules.

Confirm no overly permissive rules remain from the migration period.

Consider service dependencies, management access, VLAN isolation, and any explicitly required exceptions before modifying rules.

### v2 Decommission

Decommission remaining v2 infrastructure when no longer needed:

- Confirm all DNS rewrites point to v3.
- Shut down v2 Docker host (192.168.20.10).
- Shut down v2 Proxmox host (192.168.20.2).
- Reclaim VLAN 20 IPs.

Do not mark decommissioning complete without confirming the actual state of these systems and any remaining dependencies.

---

## Wave 8 — Synology Active Backup for Business & Plex Backup ⬜

**Status:** Pending

Wave 8 will establish versioned off-host backups and complete dedicated backup protection for Plex running directly on Unraid.

### Synology Active Backup for Business

Configure Synology DSM Active Backup for Business (ABB) to pull versioned backups from nas-prod-01.

**Goal:** Provide versioned, point-in-time recovery on top of the rolling rsync mirror.

The existing rsync scripts maintain a same-day mirror. ABB is intended to add historical recovery versions.

**Scope:**

- Source: nas-prod-01 `backups` share (`/mnt/user/backups`)
- Destination: Synology NAS dedicated backup volume
- Schedule: TBD — nightly after source backups complete
- Retention: TBD

**Status:** Pending implementation and verification.

### Dedicated Plex Full-Appdata Backup

Plex runs directly on nas-prod-01 and is not covered by the rsync jobs implemented on docker-prod-01, auth-prod-01, and immich-prod-01.

Plex's built-in database backup is already enabled and was verified operational in October 2026, but it remains within the live appdata directory.

A separate backup solution is required for full-appdata recovery.

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

The Plex service documentation previously stated that a dedicated stop–backup–restart script had been implemented.

Live investigation did not support that statement.

**Findings:**

- Plex appdata size: 6.8G
- ZFS mirror: 3.62T total, 3.41T free, ONLINE
- Built-in Plex database backups: verified every three days
- Documented dedicated backup path `/mnt/user/backups/plex/db/`: absent
- Root crontab: no Plex-specific entries found
- Unraid User Scripts directory: absent
- Dedicated Plex backup script: not found in searched common script locations
- Plex had been continuously running since September 5 before its October 26 update, inconsistent with the documented periodic stop–backup–restart procedure

**Conclusion:** The dedicated Plex full-appdata backup remained unimplemented at the time of investigation. The prior service documentation incorrectly described planned work as completed.

**Decision:** Defer implementation until Wave 8 and use a custom script.

No new backup automation was deployed during this investigation.

### Wave 8 Completion Criteria

- ⬜ Configure Synology ABB for versioned backups.
- ⬜ Verify ABB scheduling, retention, and restore capability.
- ⬜ Implement dedicated Plex full-appdata backup script.
- ⬜ Configure and verify Plex backup scheduling.
- ⬜ Confirm full-appdata backup integrity.
- ⬜ Establish and test Plex restoration procedure.
- ⬜ Update Plex service documentation and backup runbooks with final verified configuration.

---

## Exit Criteria

- ✅ Beszel agents deployed on all hosts — metrics visible
- ✅ Uptime Kuma monitors configured for all services
- ✅ Appdata rsync script implemented with nightly schedule and Healthchecks.io heartbeats
- ✅ PBS backup settings reviewed and configured during original Phase 5 work
- ✅ OIDC configured for Proxmox, Beszel, Synology
- ✅ Immich temporary external access procedure planned for October wedding
- ✅ ABS mobile app CF Access issue resolved — service token + `audiobooth://oauth` redirect URI
- ✅ MAM seeding rules configured in qBitrr
- ⬜ MAM seeding protection and clearance verified end-to-end
- ✅ Priority service settings reviewed
- ✅ Qbitrr performance and reliability audit completed — November 2026
- ✅ Public audiobook seeding cleanup and retained library playback verified
- ✅ Qbitrr database cleanup and SQLite integrity verified
- ✅ Plex built-in database backups verified operational — October 26, 2026
- ⬜ SSH key-based authentication on all hosts
- ⬜ NUT clients on auth-prod-01 and immich-prod-01
- ⬜ v2 infrastructure decommissioned
- ⬜ Firewall rules reviewed and cleaned up
- ⬜ Synology ABB configured for versioned backups
- ⬜ Dedicated Plex full-appdata backup implemented and restore-tested

**Phase status remains In Progress.**

The November 2026 Qbitrr audit closed the identified configuration and reliability review items, with live MAM compliance and new stalled-torrent recovery remaining to be verified as suitable events occur.

The outstanding security-hardening and backup work in Waves 7 and 8 is not affected by completion of the Qbitrr audit.
