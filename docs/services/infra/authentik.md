# Authentik

**Role:** Identity Provider (IdP) — SSO and OIDC for all internal services  
**Host:** auth-prod-01 (192.168.30.13)  
**Version:** 2026.5.4  
**Compose:** `/opt/stacks/authentik/compose.yaml`  
**Appdata:** `/opt/appdata/authentik/`  
**URL:** https://auth.giohosted.com  
**Last Updated:** 2026-11-05

---

## Overview

Authentik is the single Identity Provider for homelab-v3. It handles SSO via OIDC for all services that support it. Carried forward from v2 with full config restored from backup — all providers, applications, users, and policies intact.

Authentik runs on a dedicated VM (`auth-prod-01`) rather than as containers on docker-prod-01. This is intentional — LXC support was officially removed from Proxmox community scripts in May 2025, and isolating Authentik on its own VM prevents resource contention with the media stack and other containers.

---

## VM — auth-prod-01

| Component | Detail |
|-----------|--------|
| **VM ID** | 104 |
| **Host** | pve-prod-01 |
| **OS** | Ubuntu Server 24.04 LTS |
| **CPU** | 2 cores (x86-64-v3) |
| **RAM** | 5 GiB (reported by live Proxmox VM inventory on November 1, 2026; formerly documented as 4 GB) |
| **Disk** | 32GB (local-zfs, no cache) |
| **System** | q35, OVMF UEFI, VirtIO SCSI |
| **IP** | 192.168.30.13/24 |
| **Gateway** | 192.168.30.1 |
| **DNS** | 192.168.30.10 (dns-prod-01) |
| **VLAN** | 30 — Services |
| **Docker** | Installed via official script |
| **QEMU Agent** | Installed and running |

### SSH Access
```bash
ssh gio@192.168.30.13
```

### Git
- homelab-infra repo cloned at `/opt/stacks`
- Credentials cached via `git config --global credential.helper store`
- Always run `git pull` before making changes — see `runbooks/git-workflow.md`

---

## Containers

| Container          | Image                               | Role                    |
| ------------------ | ----------------------------------- | ----------------------- |
| authentik-server   | ghcr.io/goauthentik/server:2026.5.4 | Main application server |
| authentik-worker   | ghcr.io/goauthentik/server:2026.5.4 | Background task worker  |
| authentik-postgres | postgres:16-alpine                  | Database                |
| authentik-redis    | redis:alpine                        | Cache / session store   |

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

**Live verification:** All three Komodo hosts returned state `Ok` at Periphery `2.3.4`; `authentik` stack was `running` and the four Authentik application containers were listed as running through read-only MCP. The server and worker were also shown healthy by the local Docker CLI before enrollment. The live Proxmox VM inventory showed VM 104 running with **5 GiB** assigned memory, differing from the older 4 GB documentation; no RAM change was made as part of Komodo onboarding.

See `services/infra/ai-observer.md` and `runbooks/ai-observer-operations.md` for full security and operational procedures.

---

## Network & Routing

Authentik is on a dedicated VM — Traefik on docker-prod-01 cannot discover it via the Docker socket. Routing is handled via Traefik's file provider:

- **Traefik static route:** `auth.giohosted.com` → `http://192.168.30.13:9000`
- **Config file:** `/opt/appdata/traefik/config/authentik.yml` on docker-prod-01
- **AdGuard DNS rewrite:** `auth.giohosted.com` → `192.168.30.11` (Traefik IP)

Traffic path:
```
Browser → AdGuard (192.168.30.10) → 192.168.30.11 (Traefik) → 192.168.30.13:9000 (authentik-server)
```

---

## Configuration

### Environment Variables (.env)

| Variable                       | Notes                                                            |
| ------------------------------ | ---------------------------------------------------------------- |
| `PG_PASS`                      | Postgres password — carried forward from v2, must match database |
| `PG_USER`                      | `authentik`                                                      |
| `PG_DB`                        | `authentik`                                                      |
| `AUTHENTIK_SECRET_KEY`         | Carried forward from v2 — changing this invalidates all sessions |
| `AUTHENTIK_TAG`                | `2026.5.4`                                                       |
| `AUTHENTIK_BOOTSTRAP_EMAIL`    | Used on first boot only                                          |
| `AUTHENTIK_BOOTSTRAP_PASSWORD` | Used on first boot only                                          |

> **Critical:** `PG_PASS` and `AUTHENTIK_SECRET_KEY` must match the values used when the database was originally created. Changing either will break the existing database.

### Volumes

| Host Path | Container Path | Notes |
|-----------|---------------|-------|
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

---

## SSO-Enabled Services

| Service                             | Method | Provider Slug | Notes                                                                                                                                                   |
| ----------------------------------- | ------ | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Proxmox (pve-prod-01 + pve-prod-02) | OIDC | `proxmox` | Realm: `authentik`. Username claim: `username`. Strict redirect URIs: `https://192.168.10.11:8006`, `https://192.168.10.12:8006`. Shared provider with PBS. |
| PBS Docker on nas-prod-01 | OIDC | `proxmox` | Shares `OIDC - Proxmox` provider and `App - Proxmox` with PVE. Strict redirect URI: `https://192.168.30.16:8007`. Former VM 103 URI retired. |
| Beszel                              | OIDC   | `beszel`      | Redirect URI: `http://192.168.10.20:8090/api/oauth2-redirect`.                                                                                          |
| Synology DSM                        | OIDC   | `synology`    | Well-known URL must use app-specific path: `https://auth.giohosted.com/application/o/synology/.well-known/openid-configuration` — NOT the generic path. |
| Audiobookshelf                      | OIDC   | —             | Carried forward from v2 backup                                                                                                                          |
| Calibre-Web-Automated               | OAuth2 | —             | Carried forward from v2 backup                                                                                                                          |
| Shelfmark                           | OIDC   | —             | New provider created in v3 — callback `https://shelf.giohosted.com/api/auth/oidc/callback`                                                              |
| qBitrr                              | OIDC   | —             | New provider created in v3 — callback `/signin-oidc`. Both `http://` and `https://` redirect URIs registered.                                           |
| Immich                              | OIDC   | —             | Carried forward from v2 backup                                                                                                                          |
| ARR stack, Uptime Kuma              | None   | —             | LAN-only, low risk — no SSO overhead                                                                                                                    |

> **OIDC admin group:** Shelfmark and ABS use the existing `admins` group for admin authorization. No service-specific admin groups were created — reusing `admins` keeps Authentik group management simple.

> **Proxmox/PBS shared provider:** Both Proxmox nodes and Docker PBS use the same Authentik provider (`OIDC - Proxmox`) and application (`App - Proxmox`). The three **active** strict redirect URIs are `https://192.168.10.11:8006`, `https://192.168.10.12:8006`, and `https://192.168.30.16:8007`. The old `192.168.30.12:8007` redirect was replaced. Permissions remain service-specific: `gio@authentik` has Administrator on PVE and Admin on PBS. The shared OIDC secret was rotated during PBS migration and fresh login tests passed on all three services November 5; never include the secret in docs.

---

## PBS Docker Migration — OIDC Verification (November 5, 2026)

The former PBS VM `pbs-prod-01` at `192.168.30.12:8007` was replaced with a Docker-based PBS instance on `nas-prod-01` at `https://192.168.30.16:8007`. The operator chose to **retain the shared OIDC provider** rather than create a new provider for PBS.

- **Application / provider:** `App - Proxmox` / `OIDC - Proxmox` (unchanged).
- **Issuer:** `https://auth.giohosted.com/application/o/proxmox/`.
- **PBS realm:** `authentik`; username claim `username`, auto-create enabled. `gio@authentik` was granted PBS `Admin` on `/` with propagation.
- **Redirect:** Replaced the retired PBS URI with strict `https://192.168.30.16:8007`; existing PVE redirect URIs retained.
- **Secret rotation:** Shared client secret rotated by the operator; fresh private-browser OIDC logins to PBS and both PVE nodes succeeded afterward. No credential value recorded here.
- **Fallback:** The new PBS Docker image uses `admin@pbs` as a separate local emergency administrator; it does not rely on `root@pam` Linux PAM authentication in the container.

Because this provider is shared, any **future** secret rotation must update the provider clients on all three systems together. This completed rotation was independently validated by successful interactive logins, not by reading the configured secret values.

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Dedicated VM, not shared with docker-prod-01 | LXC officially removed from Proxmox community scripts May 2025. VM is the only stable option. Isolates IdP from media stack resource contention. |
| No disk cache | ZFS (local-zfs) has its own ARC caching. QEMU cache layer is unnecessary and can cause data integrity issues on ZFS storage. |
| Secrets carried forward from v2 | PG_PASS and AUTHENTIK_SECRET_KEY must match the existing database — changing them breaks the restore. |
| Proxmox and PBS share one provider | Permissions are identical and managing separate providers adds unnecessary complexity. |

---

## Notes

- Root SSH login: disabled (standard Ubuntu default)
- QEMU guest agent installed — Proxmox can cleanly shut down this VM on UPS event
- NUT client not yet configured — to be done in Phase 5 Wave 7
- Authentik worker mounts the Docker socket (`/var/run/docker.sock`) — required for embedded outpost functionality
- "No providers assigned to this outpost" warning in logs is normal — embedded outpost present but not configured for any providers
- Bootstrap email/password only apply on very first boot with empty database — ignored on subsequent starts
- Historical Phase 4 sizing check: 4 GB allocation, approximately 1.6 GB active usage, minimal swap; no bump needed **at that time**. Live Proxmox inventory on November 1, 2026 reported 5 GiB provisioned. Date/reason for the intervening adjustment was not established in this documentation update.
- Synology SSO well-known URL must point at the app-specific path (`/application/o/synology/.well-known/openid-configuration`) — the generic path (`/application/o/.well-known/openid-configuration`) causes "Unable to get the URL for SSO authentication" error
