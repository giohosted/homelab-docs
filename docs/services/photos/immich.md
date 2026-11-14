# Immich

**Role:** Photo and video library — backup, organization, ML-powered search and face detection  
**Host:** immich-prod-01 (192.168.30.14)  
**Version:** v3.3.5 (last confirmed from `immich_server` startup logs on 2026-11-01; version not re-read after node migration)  
**Compose:** `/opt/stacks/immich/compose.yaml`  
**Appdata:** `/opt/appdata/immich/`  
**URL:** `https://photos.giohosted.com`  
**Last Updated:** 2026-11-13

---

## Overview

Immich is the self-hosted photo and video library for homelab-v3. It replaces Google Photos as the primary photo backup and browsing solution. Immich runs on dedicated VM 105 (`immich-prod-01`) on the **new Dell OptiPlex 5000 SFF at `pve-prod-02`**, isolated from the media stack to contain ML worker CPU spikes. The former OptiPlex 3070 Micro was replaced on November 13, 2026. During cutover the VM was temporarily hosted on `pve-prod-03`; it is now back on node 02.

Photo files live on the NAS (`/mnt/user/photos`) and are NFS-mounted into the VM at `/data/photos`. The Postgres database lives on the VM's local disk — NFS is not supported for Postgres.

---

## VM — immich-prod-01

|Component|Detail|
|---|---|
|**VM ID**|105|
|**Host**|pve-prod-02 — Dell OptiPlex 5000 SFF (i7-12700 / 48 GB)|
|**OS**|Ubuntu Server 24.04 LTS|
|**CPU**|2 cores (x86-64-v2-AES)|
|**RAM**|6GB|
|**Disk**|32 GB on `local-zfs` (`rpool/data`) with EFI disk also on `local-zfs`|
|**System**|q35, OVMF UEFI, VirtIO SCSI|
|**IP**|192.168.30.14/24|
|**Gateway**|192.168.30.1|
|**DNS**|192.168.30.10 (dns-prod-01)|
|**VLAN**|30 — Services|
|**Docker**|Installed via official script|
|**QEMU Agent**|Enabled; VM running and Komodo healthy on 2026-11-13|

### Verified Host Migration — November 13, 2026

- Old `pve-prod-02` (OptiPlex 3070 Micro) was evacuated before replacement. Immich was temporarily hosted on the new `pve-prod-03` (Lenovo ThinkCentre M90q).
- After the new Dell joined `homelab-v3` and `local-zfs` was enabled across all three nodes, VM 105 was gracefully shut down and migrated offline with `qm migrate 105 pve-prod-02 --with-local-disks --targetstorage local-zfs`.
- Proxmox migration task completed `OK` (3 minutes, 32 seconds). VM settings remained **2 vCPU, 6 GiB RAM, 32 GB system disk, VLAN tag 30**, original virtual MAC, and UEFI EFI disk.
- VM 105 started successfully on the replacement node; Komodo confirmed all Immich application containers and Periphery running.
- Operator verified the Immich web UI and the NFS photos share at `192.168.30.16:/mnt/user/photos` mounted under `/data/photos` (`systemd` automount + NFSv3). Existing `backups`, `encoded-video`, `library`, `profile`, `thumbs`, and `upload` directories were visible.
- A manual snapshot backup to PBS from `pve-prod-02` finished `TASK OK` on November 13 at **17:16 CST**. This verified a backup operation, **not** a full restore test. NAS photo files are not included in the VM system-disk backup; they depend on their separate NAS protection.
- CPU/RAM allocations and Immich application settings were **not** changed as part of the migration. Improved host capacity is available for later performance tuning.

### SSH Access

```
ssh gio@192.168.30.14
```

### Git

- homelab-infra repo cloned at `/opt/stacks`
- Credentials cached via `git config --global credential.helper store`
- Always run `git pull` before making changes — see `runbooks/git-workflow.md`

---

## Containers

|Container|Image|Role|
|---|---|---|
|immich_server|ghcr.io/immich-app/immich-server:release|Main application server + API|
|immich_machine_learning|ghcr.io/immich-app/immich-machine-learning:release|Face detection, CLIP indexing, smart search|
|immich_postgres|ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0|Database|
|immich_redis|docker.io/valkey/valkey:9|Cache / job queue|

---

## Komodo Periphery and AI Observer — Phase 3 (Verified November 1, 2026 CST)

`immich-prod-01` is enrolled as an additional Docker server in Komodo Core on `docker-prod-01` (`https://komodo.giohosted.com`). The existing **Homelab Komodo MCP** integration on `observer-prod-01` sees the server, containers, registered stack and logs with **read-only permissions**; authorized Komodo administrators retain terminal/container-terminal operations through the Komodo UI. No additional MCP service or OpenAI tunnel was installed.

| Item | Verified configuration |
|---|---|
| Registered server | `immich-prod-01` (`192.168.30.14`), state `Ok` |
| Periphery version / image | `2.3.4`; `ghcr.io/moghtech/komodo-periphery:2` |
| Periphery Compose | `/opt/stacks/komodo-periphery/compose.yaml` (independent of Immich Compose) |
| Periphery persistent identity keys | `/opt/appdata/komodo/periphery/keys/` (`core.pub`, `periphery.pub`, `periphery.key`), root-owned/private |
| Connection to Core | Outbound `wss://komodo.giohosted.com`, with verified TLS |
| Periphery identity | `PERIPHERY_CONNECT_AS=immich-prod-01` |
| Inbound Periphery server / published TCP 8120 | Disabled (`PERIPHERY_SERVER_ENABLED=false`); no host port published |
| Compose access | `/opt/stacks` mounted; root directory `/opt` |
| Terminal behavior | Host and container terminals enabled for authorized Komodo administrators |
| Registered application stack | `immich` — **Files on Server**, run directory `/opt/stacks/immich`, file `compose.yaml`, project name `immich` |

The `immich` stack was registered against its **existing** Compose project labels; no Komodo Deploy action was used to import or recreate the application containers. The new Periphery container has privileged local Docker socket access. This does **not** grant terminal/exec/write privileges to the restricted read-only ChatGPT integration.

Onboarding used a temporary non-privileged key. It was deleted from Periphery Compose after successful registration; the agent then reconnected with `onboarding_key: None`, the temporary host secret was removed and the Komodo onboarding key was revoked. Do not put keys in the `homelab-infra` repository, Obsidian or chat. Permanent Periphery identity keys must be retained and protected by host backup procedures; coverage is not independently verified.

**Live validation:** The Komodo MCP listed this server `Ok`, Periphery `2.3.4`, and all four Immich containers running at the final check. The `immich` stack was reported running. A separate incident during rollout briefly left `immich_server` in `created` state and the stack unhealthy; that incident was addressed in a different troubleshooting conversation. The current running state was verified, but this document does not assert an unverified cause or repair sequence. `immich_server` startup logs identify application version **3.3.5** and show a successful API startup.

See `services/infra/ai-observer.md` and `runbooks/ai-observer-operations.md` for architecture, MCP security boundaries and Periphery operational checks.

---

## Network & Routing

Immich is on a dedicated VM — Traefik on docker-prod-01 routes via the file provider:

- **Traefik static route:** `photos.giohosted.com` → `http://192.168.30.14:2283`
- **Config file:** `/opt/appdata/traefik/config/immich.yml` on docker-prod-01
- **AdGuard DNS rewrite:** `photos.giohosted.com` → `192.168.30.11` (Traefik IP)
- **Port 2283** exposed on host via ports mapping — required for cross-host file provider routing
- **Traefik `websecure` responding timeouts:** `readTimeout=600s`, `idleTimeout=600s` in `/opt/stacks/traefik/compose.yaml` on docker-prod-01; verified active 2026-10-31

Traffic path:

```
Browser → AdGuard (192.168.30.10) → 192.168.30.11 (Traefik) → 192.168.30.14:2283 (immich_server)
```

The Immich iOS app uses the same Traefik HTTPS entrypoint when connected on the LAN via `https://photos.giohosted.com`. The responding-timeout flags are **Traefik static entrypoint options**, not settings in the Immich file-provider route or the Immich container.

---

## External Access

Immich is LAN-only by default. A temporary CF Tunnel route is added for the wedding event in October and removed afterwards. No CF Access policy — guests access the shared album directly via QR code, protected by Immich's built-in album password.

See `cloudflare-tunnel.md` for the October checklist.

---

## Storage

### NFS Mount

|NAS Export|Mount Point|Purpose|
|---|---|---|
|`192.168.30.16:/mnt/user/photos`|`/data/photos`|All Immich upload data — photos, thumbs, backups, encoded video|

fstab entry:

```
192.168.30.16:/mnt/user/photos /data/photos nfs defaults,_netdev,x-systemd.automount,x-systemd.mount-timeout=30 0 0
```

### Volumes

|Host Path|Container Path|Purpose|
|---|---|---|
|`/data/photos`|`/data`|Upload location — photos, thumbs, backups, encoded video|
|`/opt/appdata/immich/postgres`|`/var/lib/postgresql/data`|Postgres database — local disk only, NFS not supported|

### Immich Upload Folder Structure

Immich manages its own folder structure inside `/data/photos` (= `/mnt/user/photos` on NAS):

```
/data/photos/
├── backups/          ← Immich auto-generated DB backups (nightly)
├── encoded-video/    ← Transcoded video versions
├── library/          ← External library assets (if used)
├── profile/          ← User profile photos
├── thumbs/           ← Generated thumbnails
└── upload/           ← User uploaded photos and videos (organized by user UUID)
```

> **Important:** Immich DB auto-backups are stored at `/data/photos/backups/` — these are the authoritative backup source for the Immich database. They are on the NAS ZFS mirror pool and are retained automatically by Immich (last 7 days by default).

---

## Environment Variables (.env)

|Variable|Value|Notes|
|---|---|---|
|`UPLOAD_LOCATION`|`/data/photos`|NFS mount point on host|
|`DB_DATA_LOCATION`|`/opt/appdata/immich/postgres`|Local disk — never NFS|
|`TZ`|`America/Chicago`|Timezone|
|`IMMICH_VERSION`|`release`|Always pulls latest stable|
|`DB_PASSWORD`|see `.env`|Gitignored — never commit|
|`DB_USERNAME`|`postgres`|Postgres user|
|`DB_DATABASE_NAME`|`immich`|Postgres database name|

---

## Authentication

- **Method:** OAuth via Authentik — carried forward from v2 backup
- **Authentik application:** `Immich`
- **Authentik provider:** Carried forward from v2 — all users and sessions intact on restore
- **Local admin:** Retained as break-glass — do not delete

---

## Machine Learning Settings

Configured in Administration → Settings → Machine Learning.

|Setting|Value|Notes|
|---|---|---|
|Smart Search (CLIP)|Enabled|Model: `ViT-B-32__openai`|
|Facial Recognition|Enabled|Model: `buffalo_s`|
|OCR|Enabled|Model: `PP-OCRv5_mobile`|

**Why `buffalo_s` over `buffalo_l`:** immich-prod-01 has 2 vCPUs and 6GB RAM — ML inference runs on CPU only, no GPU passthrough. The small model is meaningfully faster with minimal accuracy difference for a personal library of ~10k assets.

---

## Job Concurrency Settings

Configured in Administration → Settings → Jobs.

|Job|Concurrency|Notes|
|---|---|---|
|Thumbnail Generation|5|Fine for 2 vCPUs — only active during imports|
|Metadata Extraction|5|Same|
|Smart Search|2|ML inference — 2 workers is appropriate|
|Face Detection|2|ML inference — 2 workers is appropriate|
|Storage Template Migration|1|Fine as-is|

> Concurrency values are intentionally left at defaults. Jobs only run during imports — idle CPU usage is 7-8% normally. Adding more vCPUs is the right lever if throughput becomes a problem, not throttling concurrency.

---

## Storage Template

Enabled in Administration → Settings → Storage Template.

**Template string:**

```
{{y}}/{{MMMM}}/{{filename}}
```

**Output example:** `2024/January/IMG_1234.jpg`

This organizes photos by year and month using the original filename — readable if browsing the NAS directly outside of Immich.

> Enabling storage template triggers a migration job that moves all existing files to the new path structure. This is safe but generates significant disk I/O on first run.

---

## Other Settings

|Setting|Value|Notes|
|---|---|---|
|Trash retention|30 days|Fine as-is|
|Video preview thumbnails|Never|Storage and CPU intensive — not worth it|
|Chapter image thumbnails|As scheduled task + on add|Fine|
|Library scan schedule|N/A|Only applies to external libraries — not used|

---

## Large iOS Video Upload Failure — Resolved 2026-10-31

**Symptoms:** Immich iOS backups generally worked, but certain videos larger than approximately 500 MB failed with a `502 Bad Gateway` response after about 60 seconds. The app accessed `https://photos.giohosted.com` over the internal Traefik reverse proxy; Immich was not publicly exposed.

**Diagnosis:** Review of `/opt/stacks/traefik/compose.yaml` and the running Traefik container showed no responding-timeout overrides on `websecure`. Traefik's default request-body `readTimeout` is 60 seconds, matching the failure timing. The reported 500 MB threshold was observational: the issue was consistent with upload duration rather than a fixed file-size cap.

**Fix applied on docker-prod-01:** Added the following flags to the `traefik` service's `command:` list in `/opt/stacks/traefik/compose.yaml`:

```
- --entrypoints.websecure.transport.respondingtimeouts.readtimeout=600s
- --entrypoints.websecure.transport.respondingtimeouts.idletimeout=600s
```

**Validation:**

1. `docker compose -f /opt/stacks/traefik/compose.yaml config --quiet` passed without output.
2. `docker compose -f /opt/stacks/traefik/compose.yaml up -d --no-deps traefik` recreated and started Traefik without restarting unrelated services.
3. `docker inspect` of the running Traefik container returned both `readtimeout=600s` and `idletimeout=600s` flags.
4. Previously failing large videos subsequently uploaded successfully from the Immich iOS app.

**Result:** Resolved and functionally verified. No changes were required to the Immich stack, NFS mounts, NAS share, or Immich file-provider route. As with all `websecure` entrypoint settings, the timeout change affects other HTTPS routes sharing that entrypoint. The `homelab-infra` Git commit/push remains a separate step until verified.

**Reference:** [Immich reverse proxy documentation](https://docs.immich.app/administration/reverse-proxy/) and `services/infra/traefik.md`.

---

## v2 → v3 Migration

Immich was restored using Immich's own auto-generated database backups rather than a manual appdata backup (the v2 appdata backup folder was empty).

### Restore Process

1. Deployed fresh Immich stack — containers came up healthy
2. Immich entered maintenance mode on first boot — no database schema present
3. Navigated to maintenance URL from server logs: `https://photos.giohosted.com/maintenance?token=<token>`
4. Cancelled the restore wizard (no backup to upload via UI) — created fresh admin user
5. Found Immich auto-generated DB backups at `/data/photos/backups/` — 14 days of backups present
6. Restored most recent backup (2026-03-10) directly via psql:

```
gunzip -c /data/photos/backups/immich-db-backup-20260310T020000-v2.5.6-pg14.19.sql.gz | docker exec -i immich_postgres psql -U postgres -d immich
```

7. Asset paths in database were stored as `/data/upload/...` — matched the official compose volume mapping (`:/data`). Photos loaded correctly after restart.
8. Thumbnail regeneration queued via Administration → Job Queues → Generate Thumbnails → All (~7,400 assets)
9. OAuth login working — user "Gio" authenticated via Authentik

### Issues Encountered

|Issue|Resolution|
|---|---|
|Immich maintenance mode on first boot|Fresh DB had no schema. Bypassed by navigating to maintenance URL from server logs, cancelling restore wizard, creating admin user.|
|UI restore failed silently|Immich maintenance mode UI restore was interrupted mid-migration, leaving empty tables. Fixed by running restore directly via psql CLI.|
|Port 2283 not accessible from Traefik|Immich compose had no ports mapping — required for cross-host file provider routing. Added `ports: - "2283:2283"` to immich-server.|
|nas-isos NFS broken on original pve-prod-02 (historical)|Original DAC-only `nas-isos` used `10.0.0.2`, unreachable from nodes 02/03. `nas-isos-p2` uses `192.168.30.16`; as of November 13 it is assigned to both nodes 02 and 03.|
|docker-prod-01 RAM pressure|docker-prod-01 had 1GB active swap. Bumped from 6GB to 8GB before creating immich-prod-01 VM.|
|Non-standard compose volume path|Initially used `/usr/src/app/upload` as internal mount path — caused confusion during path troubleshooting. Corrected to official `:/data` mapping.|

---

## Job Queues

After a fresh DB restore, run these jobs in order to fully re-index the library:

1. **Extract Metadata** → All
2. **Generate Thumbnails** → All
3. **Smart Search** → All (CLIP indexing — runs on ML worker)
4. **Face Detection** → Missing (runs on ML worker — slow on first run)
5. **Facial Recognition** → Missing (runs after face detection)

> ML jobs (Smart Search, Face Detection) run on the `immich_machine_learning` container and are CPU-heavy on first run. On a 2-core VM they will take several hours for a large library. This is expected — they run in the background and don't affect photo browsing.

---

## Database Backups

Immich automatically backs up its database nightly to `/data/photos/backups/` (= `/mnt/user/photos/backups/` on NAS). These are retained for 7 days by default.

These backups are on the NAS ZFS mirror pool — protected by ZFS redundancy and ZnapZend snapshots.

**To restore from an Immich auto-backup:**

```
# Stop the server to prevent writes during restore
docker stop immich_server

# Restore the backup
gunzip -c /data/photos/backups/<backup-filename>.sql.gz | docker exec -i immich_postgres psql -U postgres -d immich

# Start the server
docker start immich_server
```

> Always use the psql CLI method — the Immich UI restore is unreliable and can fail silently mid-migration.

---

## Directory Structure

```
/opt/stacks/immich/
  compose.yaml              ← in git
  .env                      ← gitignored (DB_PASSWORD and other secrets)

/opt/appdata/immich/
  postgres/                 ← Postgres data — local disk, never backed up via rsync (Immich auto-backups to NAS are the DB backup source)

/data/photos/               ← NFS mount from nas-prod-01:/mnt/user/photos
  backups/                  ← Immich auto-generated DB backups
  encoded-video/
  library/
  profile/
  thumbs/
  upload/                   ← Photo and video files by user UUID
```

---

## Traefik File Provider Config

`/opt/appdata/traefik/config/immich.yml` on docker-prod-01:

```
http:
  routers:
    immich:
      rule: "Host(`photos.giohosted.com`)"
      entrypoints:
        - websecure
      tls:
        certResolver: cloudflare
      service: immich

  services:
    immich:
      loadBalancer:
        servers:
          - url: "http://192.168.30.14:2283"
```

For upload timeout settings, see the separate **static** Traefik Compose flags described under "Large iOS Video Upload Failure"; no change to this file-provider route was needed.

---

## Design Decisions

|Decision|Rationale|
|---|---|
|Dedicated VM on new pve-prod-02|ML worker CPU spikes remain isolated from the media stack; replacement i7-12700/48 GB node provides additional compute/memory headroom. Historical placement decisions are in `architecture/decisions-log.md`.|
|Postgres on local VM disk|Immich explicitly does not support NFS for Postgres. Local disk avoids NFS latency and locking issues.|
|DB restore via psql CLI|UI restore failed silently mid-migration. CLI restore is reliable and provides clear output.|
|Official compose volume mapping `:/data`|Non-standard paths cause DB asset path mismatches on future migrations. Official mapping ensures consistency with Immich tooling.|
|LAN-only by default|No need for permanent external exposure. Temporary tunnel added for wedding event only — see cloudflare-tunnel.md.|
|buffalo_s over buffalo_l|Faster on CPU-only VM with negligible accuracy difference for a personal library.|
|2 vCPUs retained after hardware move|Allocation was not changed during migration. Historical 7–8% idle CPU observation predates the Dell replacement; reassess before increasing cores.|

---

## Notes

- Root SSH login: disabled (standard Ubuntu default)
- QEMU guest agent installed — Proxmox can cleanly shut down this VM on UPS event
- NUT client not yet configured — to be done in Phase 5 Wave 7
- `gio` user added to `service` group (GID 2000) on immich-prod-01 for file permission consistency
- git identity configured: `delgadogiovanny@gmail.com` / `giohosted`
- The previous v3.0.2 version note was superseded by a live `immich_server` startup log confirming v3.3.5 on November 1, 2026 (CST). The application version was not independently rechecked after the November 13 host migration.
- Related host documents: `architecture/compute/pve-prod-02.md`, `architecture/compute/pve-prod-03.md`.
