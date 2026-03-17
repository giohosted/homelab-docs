# Phase 5 — Hardening & Operational Readiness

**Status:** Pending  
**Started:** —  
**Last Updated:** 2026-03-16

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

## Wave 1 — Monitoring Stack

### Beszel

Deploy Beszel agent on all hosts — metrics server already running on pi-prod-01.

**Hosts to deploy agents on:**
- pve-prod-01
- pve-prod-02
- docker-prod-01
- auth-prod-01
- immich-prod-01
- nas-prod-01
- pi-prod-01 (local agent)

**Prerequisite firewall rule (add before deploying):**
- Source: VLAN 30 (192.168.30.0/24)
- Destination: 192.168.10.20 TCP port 45876
- Action: Allow
- Reason: Beszel agents on VLAN 30 hosts must reach the Beszel server on VLAN 10

**OIDC:** Configure Authentik OIDC for Beszel — requires `email_verified: true` custom scope.

---

### Uptime Kuma

Already running on pi-prod-01. Configure monitors for all v3 services:

| Service | Monitor Type | URL/Host |
|---------|-------------|---------|
| Traefik | HTTP | `https://traefik.giohosted.com` |
| Authentik | HTTP | `https://auth.giohosted.com` |
| AdGuard primary | DNS | 192.168.30.10 |
| AdGuard secondary | DNS | 192.168.30.15 |
| docker-prod-01 | Ping | 192.168.30.11 |
| auth-prod-01 | Ping | 192.168.30.13 |
| immich-prod-01 | Ping | 192.168.30.14 |
| pbs-prod-01 | Ping | 192.168.30.12 |
| Plex | HTTP | `http://192.168.30.2:32400/web` |
| Immich | HTTP | `https://photos.giohosted.com` |
| Audiobookshelf | HTTP | `https://audiobooks.giohosted.com` |

---

## Wave 2 — Backup Automation

### Appdata rsync Script

Write a hardened rsync script to back up `/opt/appdata` and `/opt/stacks` from docker-prod-01, auth-prod-01, and immich-prod-01 to the NAS backups share nightly.

**Script requirements:**
- rsync `/opt/appdata` → `/mnt/user/backups/appdata/<hostname>/`
- rsync `/opt/stacks` → `/mnt/user/backups/stacks/<hostname>/`
- Exclude `.env` files from stacks backup (already in git, secrets stay off NAS)
- Healthchecks.io heartbeat ping on success — silent failure detection
- Run nightly via cron at 03:00 (after PBS backup job at 02:00)
- Log output to `/var/log/backup-appdata.log`

**Hosts:**
- docker-prod-01 → NFS mount backups share temporarily or use SSH rsync to NAS
- auth-prod-01 → same
- immich-prod-01 → same

### PBS Backup Review

Revisit PBS settings:
- Retention policy — review keep last/daily/weekly/monthly values
- Schedule timing — currently 02:00 daily, verify no conflicts
- Prune schedule — confirm auto-prune is running
- Garbage collection — configure GC schedule
- Verify jobs — enable backup verification if supported on PBS 4.1.0
- Namespace organization — consider per-node namespaces for cleaner UI

### Healthchecks.io

Set up Healthchecks.io monitors for:
- Appdata rsync script (per host)
- PBS backup job completion
- Any other cron jobs added in Phase 5

---

## Wave 3 — OIDC Rollout

Configure Authentik OIDC for remaining services that support it:

| Service | Status | Notes |
|---------|--------|-------|
| Proxmox (both nodes) | ⬜ Pending | OIDC via Authentik — both pve-prod-01 and pve-prod-02 |
| Immich | ✅ Done | Carried forward from v2 |
| Beszel | ⬜ Pending | Requires `email_verified: true` custom scope in Authentik |
| Synology DSM | ⬜ Pending | Local admin retained as break-glass |
| Audiobookshelf | ✅ Done | Carried forward from v2 |
| Calibre-Web-Automated | ✅ Done | Carried forward from v2 |
| Shelfmark | ✅ Done | New provider created in v3 |
| qBitrr | ✅ Done | New provider created in v3 |

---

## Wave 4 — External Access & CF Access

### Immich — Cloudflare Tunnel + CF Access

Add `photos.giohosted.com` to the homelab-v3 CF Tunnel and configure Cloudflare Access policy.

Steps:
1. Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Add route: `photos.giohosted.com` → `https://192.168.30.11`, No TLS Verify
2. Cloudflare Zero Trust → Access → Applications → Add: `photos.giohosted.com`
3. Configure Access policy — Authentik OIDC login required

### ABS Mobile App — CF Access Fix

Cloudflare Access blocks API calls from the ABS mobile app because the app cannot complete the browser-based auth challenge.

**Fix options to evaluate:**
- Add a CF Access service token for the ABS API endpoints
- Configure bypass rules for specific ABS API paths (`/api/*`)
- Use Cloudflare Access → Service Auth → create a service token and configure in ABS mobile app settings

---

## Wave 5 — MAM Seeding Rules

Configure qBitrr MAM-specific seeding rules for ebooks and audiobooks.

**Requirements:**
- 14-day minimum seed time for MAM torrents only
- Rule must be tracker-scoped — applies to MAM only, NOT AudiobookBay or other sources
- ebooks category: `ebooks`
- audiobooks category: `audiobooks`

**Configuration location:** `/opt/appdata/qbitrr/config.toml` — `[qBit.CategorySeeding]` section with tracker-specific overrides.

---

## Wave 6 — Service Settings Review

Review and tune settings for all deployed services. Goal is to move from default/restored configs to intentionally configured optimal settings.

### Priority services to review:

**qBitrr** — currently mostly default settings. Review:
- Search concurrency limits
- Stalled torrent handling thresholds
- Re-search behavior
- Loop timing
- Upgrade search settings

**PBS** — review retention, schedule, GC, verify jobs (covered in Wave 2 but settings review is separate)

**Immich** — review:
- ML concurrency settings (face detection, CLIP indexing workers)
- Thumbnail quality settings
- Storage template
- Library scan schedule
- Trash retention period

**AdGuard** — review:
- Blocklist sources
- Query log retention
- Client-specific rules if needed

**Traefik** — review:
- Access log configuration
- Rate limiting if needed
- Any missing middleware

**ARR stack** — review quality profiles, indexer priorities, download client settings post-migration

**Plex** — review:
- Scheduled scan timing
- Thumbnail generation settings
- Transcoding quality settings

---

## Wave 7 — Security Hardening

### SSH Key-Based Authentication

Set up SSH key-based auth on all hosts and disable password auth:

| Host | Status |
|------|--------|
| docker-prod-01 | ⬜ Pending |
| auth-prod-01 | ⬜ Pending |
| immich-prod-01 | ⬜ Pending |
| pve-prod-01 | ⬜ Pending |
| pve-prod-02 | ⬜ Pending |
| pi-prod-01 | ⬜ Pending |
| nas-prod-01 | ⬜ Pending |

### NUT Client on auth-prod-01 and immich-prod-01

NUT clients not configured on auth-prod-01 or immich-prod-01 during Phase 4. Add during Phase 5:
- Install `nut-client` on both VMs
- Configure `/etc/nut/nut.conf` and `/etc/nut/upsmon.conf`
- Monitor `ups@192.168.10.10`

### Firewall Rule Review

Review all UDM-SE firewall rules — confirm no overly permissive rules remain from migration period. Add Beszel rule (VLAN 30 → 192.168.10.20 TCP 45876).

### v2 Decommission

Decommission v2 infrastructure:
- Confirm all DNS rewrites pointing at v3
- Shut down v2 Docker host (192.168.20.10)
- Shut down v2 Proxmox host (192.168.20.2)
- Reclaim VLAN 20 IPs

---

## Exit Criteria

- ⬜ Beszel agents deployed on all hosts — metrics visible
- ⬜ Uptime Kuma monitors configured for all services
- ⬜ Appdata rsync script running nightly with Healthchecks.io heartbeats
- ⬜ PBS backup settings reviewed and optimized
- ⬜ OIDC configured for Proxmox, Beszel, Synology
- ⬜ Immich CF Tunnel + CF Access configured
- ⬜ ABS mobile app CF Access issue resolved
- ⬜ MAM seeding rules configured in qBitrr
- ⬜ Service settings review complete for all priority services
- ⬜ SSH key-based auth on all hosts
- ⬜ NUT clients on auth-prod-01 and immich-prod-01
- ⬜ v2 infrastructure decommissioned
- ⬜ Firewall rules reviewed and cleaned up