# Authentik

**Role:** Identity Provider (IdP) — SSO and OIDC for internal services  
**Host:** auth-prod-01 (192.168.30.13)  
**Proxmox Node:** pve-prod-03 — Lenovo ThinkCentre M90q  
**Version:** 2026.5.4  
**Compose:** `/opt/stacks/authentik/compose.yaml`  
**Appdata:** `/opt/appdata/authentik/`  
**URL:** https://auth.giohosted.com  
**Last Updated:** 2026-11-14

---

## Overview

Authentik is the single Identity Provider for homelab-v3. It handles SSO via OIDC for all services that support it. Carried forward from v2 with full config restored from backup — all providers, applications, users, and policies intact.

Authentik runs on a dedicated VM (`auth-prod-01`) rather than as containers on docker-prod-01. This is intentional — LXC support was officially removed from Proxmox community scripts in May 2025, and isolating Authentik on its own VM prevents resource contention with the media stack and other containers.

As of November 14, 2026, VM 104 runs on `pve-prod-03` after a verified offline migration from `pve-prod-01`. The VM retains its original service IP, VLAN, CPU/RAM allocation, and application configuration.

---

## VM — auth-prod-01

| Component | Detail |
|-----------|--------|
| **VM ID** | 104 |
| **Host** | pve-prod-03 — Lenovo ThinkCentre M90q |
| **OS** | Ubuntu Server 24.04 LTS |
| **CPU** | 2 cores (x86-64-v2-AES) |
| **RAM** | 4 GiB (verified November 14, 2026) |
| **Memory ballooning** | Disabled |
| **Disk** | 32 GB (local-zfs, no cache) |
| **System** | q35, OVMF UEFI, VirtIO SCSI |
| **IP** | 192.168.30.13/24 |
| **Gateway** | 192.168.30.1 |
| **DNS** | 192.168.30.10 (dns-prod-01) |
| **VLAN** | 30 — Services |
| **Docker** | Installed via official script |
| **QEMU Agent** | Installed and running |
| **Start at boot** | Enabled; Proxmox startup `order=4,up=60` |

### SSH Access

```bash
ssh gio@192.168.30.13
```

### Git

- homelab-infra repo cloned at `/opt/stacks`
- Credentials cached via `git config --global credential.helper store`
- Always run `git pull` before making changes — see `runbooks/git-workflow.md`

### VM Memory Right-Sizing — November 13, 2026

The Authentik VM was reviewed as part of a Homelab v3 Proxmox resource-sizing audit.

Proxmox initially reported nearly full VM memory utilization, while guest-level diagnostics showed substantial reclaimable Linux filesystem cache and low actual application demand.

**Completed change:**

- Reduced VM 104 memory allocation from 5 GiB to 4 GiB.
- Retained 2 vCPUs.
- Retained disabled memory ballooning.
- Rebooted the VM to apply the new memory allocation.
- Confirmed the 4 GiB configuration was active in Proxmox, with no pending memory changes.

**Post-resize verification — November 13:**

| Metric | Observed value |
|---|---|
| Configured RAM | 4 GiB |
| Guest-reported total RAM | 3.8 GiB |
| Guest RAM used | 1.4 GiB |
| Guest RAM available | 2.4 GiB |
| Buffer/cache | 2.5 GiB |
| Swap capacity | 3.8 GiB |
| Swap occupied | 268 KiB |
| Memory PSI avg10/avg60/avg300 | 0.00 / 0.00 / 0.00 |
| Active swap-in/out during five-second sample | None |
| Komodo server connection | Ok |
| Docker containers | 6 of 6 running |

The verified running containers included `authentik-server`, `authentik-worker`, `authentik-postgres`, `authentik-redis`, `beszel-agent`, and `komodo-periphery`.

**Result:** The 4 GiB allocation passed initial post-reboot verification, with approximately 2.4 GiB of available guest memory, negligible swap occupancy and no observed recent memory pressure.

The previous 5 GiB allocation was not required by the workload observed during this audit. However, the 4 GiB allocation is not an independently established minimum for every possible Authentik workload.

**Operational considerations:**

- Monitor available memory, memory pressure, swap activity and service health during normal operation.
- Retain sufficient headroom for Authentik upgrades, migrations, background tasks and PostgreSQL maintenance.
- Reevaluate the allocation if sustained memory pressure, active swapping, OOM events or service degradation occurs.
- Reboot-related reduction in swap occupancy should not be interpreted as proof that historical memory pressure has been eliminated.
- No CPU, application configuration, Docker resource limit or authentication-policy changes were made as part of this optimization.

### Proxmox Host Migration — November 14, 2026

**Status:** Completed and post-migration verification passed.

**Objective:** Relocate the Authentik VM from the primary Minisforum Proxmox node to the Lenovo third node, reducing the number of critical services affected by maintenance and reboots on `pve-prod-01`.

| Item | Value |
|---|---|
| VM | `auth-prod-01` (104) |
| Source | `pve-prod-01` (AMD Ryzen 9 7945HX) |
| Destination | `pve-prod-03` (Intel Core i7-10700) |
| Migration method | Offline, with local disk migration |
| Source storage | `local-zfs` |
| Destination storage | `local-zfs` |
| VM disks | 32 GiB primary disk plus UEFI disk |
| CPU | 2 vCPU, `x86-64-v2-AES` |
| RAM | 4 GiB |
| Network | VLAN 30, `192.168.30.13/24` |
| MAC | `BC:24:11:CA:5D:1B` |

**Procedure completed:**

1. Reviewed the health of all three Proxmox nodes, cluster quorum, ZFS mirrors, and shared PBS storage.
2. Verified the scheduled November 14 PBS backup of VM 104 succeeded while it was still on `pve-prod-01`.
3. Gracefully shut down VM 104.
4. Performed the offline migration using:

   ```bash
   qm migrate 104 pve-prod-03 --with-local-disks --targetstorage local-zfs --online 0
   ```

5. Confirmed VM 104 was registered on `pve-prod-03` with its virtual disks, network configuration, and resource assignments preserved.
6. Started VM 104 on the Lenovo.
7. Validated the guest, Docker application services, network routing, SSO, UPS monitoring, and PBS backup.

**Why offline migration was selected:**

- The source uses AMD and the destination uses Intel hardware.
- The VM disks reside on node-local ZFS storage rather than shared guest storage.
- An offline migration avoids live CPU-state compatibility and live disk-transfer concerns.
- A controlled Authentik service interruption was acceptable for the maintenance window.

**Post-migration guest verification:**

| Check | Result |
|---|---|
| VM status | Running on `pve-prod-03` |
| Guest address | `192.168.30.13` |
| Default gateway | `192.168.30.1` |
| Guest RAM total | 3.8 GiB |
| Guest RAM used | 1.3 GiB |
| Guest RAM available | 2.4 GiB |
| Swap | 0 B occupied |
| PostgreSQL | Running and healthy |
| Authentik server | Running and healthy |
| Authentik worker | Running and healthy |
| Redis | Running |
| Beszel agent | Running |
| Komodo Periphery | Running |
| Komodo connection | `Ok` |
| Local Authentik health endpoint | HTTP 200 |
| Reverse-proxy backend | HTTP 302 |
| Public Authentik URL | HTTP 302 |
| Internal DNS resolution | `auth.giohosted.com` → `192.168.30.11` |
| SSO functional test | Passed, operator-confirmed |
| Lenovo ZFS pool | `ONLINE`, no known data errors |

The reverse-proxy HTTP 302 responses are expected redirects and did not indicate a failed application health check.

The VM's Docker bridge addresses (`172.17.0.1`, `172.18.0.1`, and `172.19.0.1`) were present alongside the primary Services VLAN address. These did not replace the guest's production IP.

**UPS monitoring:**

The destination Proxmox node's NUT client was checked after reboot:

```text
systemctl is-enabled nut-monitor
enabled

systemctl is-active nut-monitor
active

upsc ups@192.168.10.10 ups.status
OL
```

The `Init SSL without certificate database` notice was nonfatal. The UPS status `OL` indicates the server reported utility power at the time of the check.

The guest itself does not require a separate NUT network client for the established hypervisor-managed shutdown model. Actual UPS-triggered guest shutdown sequencing remains untested.

**Fresh post-migration PBS backup:**

A manual snapshot backup was executed on `pve-prod-03`:

```bash
vzdump 104 --storage pbs-prod-01 --mode snapshot
```

The job completed successfully on **November 14, 2026, at 14:09:45 CST**.

| Metric | Result |
|---|---|
| Backup mode | Snapshot |
| Source host | `pve-prod-03` |
| Target | `pbs-prod-01` (`nas-backups`) |
| Virtual disk size processed | 32.00 GiB |
| Sparse zero data | 11.59 GiB (36%) |
| Reused backup data | 30.52 GiB (95%) |
| Duration | 44 seconds |
| Reported transfer throughput | 744.7 MiB/s |
| Result | Backup job finished successfully |

The message `notification was not matched by any matcher` indicates that the completed event did not match a configured notification rule. It did not indicate a backup failure.

**Validation boundaries:**

- A fresh backup operation succeeded, but a full isolated VM restore was not performed.
- The migration preserved the operating system and application configuration; no new Authentik deployment or database restoration was required.
- The migration did not create automatic HA or shared guest storage.
- If node 03 fails, recovering its local-disk guests still requires deliberate restore or recovery.
- Cluster quorum remains independent of any single node, but the IdP is still a single VM instance.

---

## Containers

| Container | Image | Role |
|---|---|---|
| authentik-server | ghcr.io/goauthentik/server:2026.5.4 | Main application server |
| authentik-worker | ghcr.io/goauthentik/server:2026.5.4 | Background task worker |
| authentik-postgres | postgres:16-alpine | Database |
| authentik-redis | redis:alpine | Cache / session store |

All **Authentik application** containers use the internal `authentik-internal` Docker network — not the proxy network. Only the Authentik server publishes port 9000 to the host. The independently deployed Komodo Periphery agent is not an Authentik application service and publishes no host port.

---

## Komodo Periphery and AI Observer — Phase 3 (Verified November 1, 2026 CST)

`auth-prod-01` is enrolled as an additional Docker server in Komodo Core on `docker-prod-01` (`https://komodo.giohosted.com`). The existing **Homelab Komodo MCP** connection on `observer-prod-01` can list the host, Authentik containers and stack through its **read-only** API identity; authorized Komodo administrators can use the Komodo UI to manage containers and open terminals. No extra MCP container or OpenAI tunnel is necessary.

| Item | Verified configuration |
|---|---|
| Registered server | `auth-prod-01` (`192.168.30.13`), state `Ok` |
| Periphery image / version | `ghcr.io/moghtech/komodo-periphery:2`, running `2.3.4` |
| Periphery Compose | `/opt/stacks/komodo-periphery/compose.yaml`, independent of `/opt/stacks/authentik/compose.yaml` |
| Permanent agent keys | `/opt/appdata/komodo/periphery/keys/` (`core.pub`, `periphery.pub`, `periphery.key`), root-owned/private |
| Core connection | Outbound `wss://komodo.giohosted.com`, verified TLS |
| Identity | `PERIPHERY_CONNECT_AS=auth-prod-01` |
| Periphery listener | Disabled (`PERIPHERY_SERVER_ENABLED=false`); no TCP 8120 published on host |
| Host access | Docker socket mounted locally and `/opt/stacks` available through root directory `/opt` |
| Terminal policy | Host and container terminals enabled for authorized Komodo administrators |
| Adopted Compose stack | `authentik`: **Files on Server**, run directory `/opt/stacks/authentik`, file `compose.yaml`, project name `authentik` |

The application stack was adopted by matching its Docker Compose project labels. Registration did **not** entail a new Komodo deployment, a recreation of Authentik/PostgreSQL or a change to the identity-provider configuration. The Periphery Docker-socket mount is privileged and must be protected through Komodo administrative access controls; it does **not** change the read-only capabilities of the ChatGPT MCP identity.

The separate non-privileged onboarding key used for initial registration was removed from the Compose file, the Periphery agent recreated and verified logged in with `onboarding_key: None`, the root-only temporary credential file removed and the onboarding key revoked in Komodo Settings. Retain and securely back up the persistent key directory. Host backup coverage of these new keys has not yet been independently verified.

**Live verification (November 1):** All three Komodo hosts returned state `Ok` at Periphery `2.3.4`; the `authentik` stack was `running` and the four Authentik application containers were listed as running through read-only MCP. The server and worker were also shown healthy by the local Docker CLI before enrollment. The live Proxmox VM inventory showed VM 104 running with **5 GiB** assigned memory, differing from the older 4 GB documentation; no RAM change was made as part of Komodo onboarding.

**Subsequent resource update (November 13):** VM 104 was intentionally reduced from 5 GiB to 4 GiB during the resource-sizing audit. The new configuration and initial guest memory checks were verified after reboot, and all six containers were running. See the VM Memory Right-Sizing section above.

**Subsequent host migration (November 14):** The existing VM moved from `pve-prod-01` to `pve-prod-03`. The Komodo server identity and connection were preserved. After VM startup on node 03, Komodo again returned state `Ok` and listed all six containers as running.

See `services/infra/ai-observer.md` and `runbooks/ai-observer-operations.md` for full security and operational procedures.

---

## Network & Routing

Authentik is on a dedicated VM — Traefik on docker-prod-01 cannot discover it via the Docker socket. Routing is handled via Traefik's file provider:

- **Traefik static route:** `auth.giohosted.com` → `http://192.168.30.13:9000`
- **Config file:** `/opt/appdata/traefik/config/authentik.yml` on docker-prod-01
- **AdGuard DNS rewrite:** `auth.giohosted.com` → `192.168.30.11` (Traefik IP)

Traffic path:

```text
Browser → AdGuard (192.168.30.10) → 192.168.30.11 (Traefik) → 192.168.30.13:9000 (authentik-server)
```

The VM's move to `pve-prod-03` did not require changing the internal DNS rewrite, Traefik route, or Authentik service IP. The Services VLAN remained VLAN 30 on the destination's VLAN-aware Proxmox bridge.

---

## Configuration

### Environment Variables (.env)

| Variable | Notes |
|---|---|
| `PG_PASS` | Postgres password — carried forward from v2, must match database |
| `PG_USER` | `authentik` |
| `PG_DB` | `authentik` |
| `AUTHENTIK_SECRET_KEY` | Carried forward from v2 — changing this invalidates all sessions |
| `AUTHENTIK_TAG` | `2026.5.4` |
| `AUTHENTIK_BOOTSTRAP_EMAIL` | Used on first boot only |
| `AUTHENTIK_BOOTSTRAP_PASSWORD` | Used on first boot only |

> **Critical:** `PG_PASS` and `AUTHENTIK_SECRET_KEY` must match the values used when the database was originally created. Changing either will break the existing database.

### Volumes

| Host Path | Container Path | Notes |
|---|---|---|
| `/opt/appdata/authentik/postgres` | `/var/lib/postgresql/data` | Database files |
| `/opt/appdata/authentik/redis` | `/data` | Redis persistence |
| `/opt/appdata/authentik/data` | `/data` | Authentik application data |
| `/opt/appdata/authentik/media` | `/media` | Uploaded media |
| `/opt/appdata/authentik/certs` | `/certs` | Certificates |
| `/opt/appdata/authentik/custom-templates` | `/templates` | Custom UI templates |

---

## v2 → v3 Migration

Authentik was restored from a v2 backup rather than set up fresh — all OIDC providers, applications, users, groups, and policies carried forward intact.

### Restore Process

1. Appdata backed up from v2 host to Synology via Active Backup for Business
2. `.env` file recovered from Synology (not in NAS rsync backup — rsync excluded secrets)
3. Appdata copied to auth-prod-01: `/opt/appdata/authentik/`
4. Compose started with original `PG_PASS` and `AUTHENTIK_SECRET_KEY` values
5. Authentik came up with full v2 config intact — no reconfiguration needed

This historical v2-to-v3 application restore is separate from the November 14 Proxmox host migration, which copied the existing VM and did not require reinstalling or restoring Authentik.

---

## SSO-Enabled Services

| Service | Method | Provider Slug | Notes |
|---|---|---|---|
| Proxmox (previously verified on pve-prod-01 and pve-prod-02) | OIDC | `proxmox` | Realm: `authentik`. Username claim: `username`. Previously confirmed strict redirect URIs: `https://192.168.10.11:8006`, `https://192.168.10.12:8006`. Shared provider with PBS. OIDC configuration for new node 03 has not been independently documented. |
| PBS Docker on nas-prod-01 | OIDC | `proxmox` | Shares `OIDC - Proxmox` provider and `App - Proxmox` with PVE. Strict redirect URI: `https://192.168.30.16:8007`. Former VM 103 URI retired. |
| Beszel | OIDC | `beszel` | Redirect URI: `http://192.168.10.20:8090/api/oauth2-redirect`. |
| Synology DSM | OIDC | `synology` | Well-known URL must use app-specific path: `https://auth.giohosted.com/application/o/synology/.well-known/openid-configuration` — NOT the generic path. |
| Audiobookshelf | OIDC | — | Carried forward from v2 backup |
| Calibre-Web-Automated | OAuth2 | — | Carried forward from v2 backup |
| Shelfmark | OIDC | — | New provider created in v3 — callback `https://shelf.giohosted.com/api/auth/oidc/callback` |
| qBitrr | OIDC | — | New provider created in v3 — callback `/signin-oidc`. Both `http://` and `https://` redirect URIs registered. |
| Immich | OIDC | — | Carried forward from v2 backup |
| ARR stack, Uptime Kuma | None | — | LAN-only, low risk — no SSO overhead |

> **OIDC admin group:** Shelfmark and ABS use the existing `admins` group for admin authorization. No service-specific admin groups were created — reusing `admins` keeps Authentik group management simple.

> **Proxmox/PBS shared provider:** Proxmox nodes 01/02 and Docker PBS were previously verified using the same Authentik provider (`OIDC - Proxmox`) and application (`App - Proxmox`). The three documented active strict redirect URIs were `https://192.168.10.11:8006`, `https://192.168.10.12:8006`, and `https://192.168.30.16:8007`. The old `192.168.30.12:8007` redirect was replaced. The present provider configuration should be read back before asserting that node 03 has been added. Permissions remain service-specific: `gio@authentik` has Administrator on the previously verified PVE nodes and Admin on PBS. The shared OIDC secret was rotated during PBS migration and fresh login tests passed on those three services November 5; never include the secret in docs.

**November 14:** A fresh functional SSO test passed after migrating Authentik to node 03. The specific OIDC client endpoints tested were not itemized in the operator's confirmation, so that result does not independently establish node 03's redirect configuration.

---

## PBS Docker Migration — OIDC Verification (November 5, 2026)

The former PBS VM `pbs-prod-01` at `192.168.30.12:8007` was replaced with a Docker-based PBS instance on `nas-prod-01` at `https://192.168.30.16:8007`. The operator chose to **retain the shared OIDC provider** rather than create a new provider for PBS.

- **Application / provider:** `App - Proxmox` / `OIDC - Proxmox` (unchanged).
- **Issuer:** `https://auth.giohosted.com/application/o/proxmox/`.
- **PBS realm:** `authentik`; username claim `username`, auto-create enabled. `gio@authentik` was granted PBS `Admin` on `/` with propagation.
- **Redirect:** Replaced the retired PBS URI with strict `https://192.168.30.16:8007`; existing PVE redirect URIs retained.
- **Secret rotation:** Shared client secret rotated by the operator; fresh private-browser OIDC logins to PBS and both then-existing PVE nodes succeeded afterward. No credential value recorded here.
- **Fallback:** The new PBS Docker image uses `admin@pbs` as a separate local emergency administrator; it does not rely on `root@pam` Linux PAM authentication in the container.

Because this provider is shared, any **future** secret rotation must update all configured provider clients together. The completed November 5 rotation was independently validated by successful interactive logins, not by reading the configured secret values.

---

## Design Decisions

| Decision | Rationale |
|---|---|
| Dedicated VM, not shared with docker-prod-01 | LXC officially removed from Proxmox community scripts May 2025. VM is the only stable option. Isolates IdP from media stack resource contention. |
| Placement on pve-prod-03 | Separates Authentik from the primary Docker host and primary DNS on pve-prod-01, reducing the impact of planned primary-node maintenance. Offline migration validated November 14. |
| 4 GiB RAM allocation | Verified low steady-state memory demand and significant available RAM after right-sizing from 5 GiB. Retains headroom for normal operation while reclaiming 1 GiB of committed guest allocation. |
| No disk cache | ZFS (local-zfs) has its own ARC caching. QEMU cache layer is unnecessary and can cause data integrity issues on ZFS storage. |
| Secrets carried forward from v2 | PG_PASS and AUTHENTIK_SECRET_KEY must match the existing database — changing them breaks the restore. |
| Proxmox and PBS share one provider | Original decision retained for verified PVE/PBS clients. Any additional node must have its OIDC client and redirect configuration checked before being considered covered. |
| Offline migration with local ZFS disks | Preserves the VM's existing identity and application configuration while avoiding live CPU-state compatibility concerns between AMD and Intel hosts. |

---

## Notes

- Root SSH login: disabled (standard Ubuntu default)
- QEMU guest agent installed — Proxmox can cleanly shut down this VM on UPS event
- NUT client is not configured inside the Authentik guest. The destination hypervisor (`pve-prod-03`) runs an operational NUT client connected to `nas-prod-01`; end-to-end UPS shutdown testing remains outstanding
- Authentik worker mounts the Docker socket (`/var/run/docker.sock`) — required for embedded outpost functionality
- "No providers assigned to this outpost" warning in logs is normal — embedded outpost present but not configured for any providers
- Bootstrap email/password only apply on very first boot with empty database — ignored on subsequent starts
- Historical Phase 4 sizing check: 4 GB allocation, approximately 1.6 GB active usage, minimal swap; no bump needed **at that time**. Live Proxmox inventory on November 1, 2026 reported 5 GiB provisioned. Date/reason for the intervening adjustment was not established during the November 5 documentation review. The subsequent reduction to 4 GiB was an intentional, verified change on November 13, 2026.
- Synology SSO well-known URL must point at the app-specific path (`/application/o/synology/.well-known/openid-configuration`) — the generic path (`/application/o/.well-known/openid-configuration`) causes "Unable to get the URL for SSO authentication" error
- The `auth-prod-01` hostname and `192.168.30.13` address did not change during the November 14 migration
- VM 104 now resides on `pve-prod-03`; it should no longer be counted among `pve-prod-01`'s active guests
- Full VM restore testing and the complete UPS-triggered shutdown sequence remain open validation items
