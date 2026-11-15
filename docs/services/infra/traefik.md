# Traefik

**Role:** Reverse proxy, TLS termination, internal service routing  
**Host:** docker-prod-01 (192.168.30.11)  
**Image:** `traefik:latest`  
**Version:** v3.7.19 (verified 2026-11-14)  
**Compose:** `/opt/stacks/traefik/compose.yaml`  
**Appdata:** `/opt/appdata/traefik/`  
**Last Updated:** 2026-11-14

---

## Overview

Traefik is the v3 reverse proxy, replacing Nginx Proxy Manager from v2. It handles:

- Routing internal requests to the correct container by hostname
- TLS termination with a wildcard cert for `*.giohosted.com`
- Cross-host routing to services on other VMs (auth-prod-01, immich-prod-01) via the file provider
- HTTP → HTTPS redirect on all requests

Traefik is **not** exposed directly to the internet. Internal routing and external access through cloudflared (Cloudflare Tunnel) are separate mechanisms.

**Current deployment:** Traefik v3.7.19, upgraded and verified on 2026-11-14. The Compose configuration uses the moving `traefik:latest` tag. Version changes must therefore be reviewed before pulling future images.

---

## How It Works

When a request comes in for e.g. `https://sonarr-tv.giohosted.com`:

1. Your device asks AdGuard "what's the IP for sonarr-tv.giohosted.com?"
2. AdGuard returns `192.168.30.11` (DNS rewrite)
3. Your device connects to `192.168.30.11:443`
4. Traefik receives the request, matches the hostname to the sonarr-tv router, and forwards to the sonarr-tv container
5. Response comes back through Traefik to your device

Cloudflare is only involved in **certificate issuance** for this internal request path — not in the traffic path itself.

---

## TLS Certificate

- **Type:** Wildcard — covers `giohosted.com` and `*.giohosted.com`
- **Issuer:** Let's Encrypt (via ACME)
- **Challenge:** DNS-01 via Cloudflare API token
- **Storage:** `/opt/appdata/traefik/acme.json` (chmod 600)
- **Auto-renewal:** Yes — handled automatically by Traefik.
- **Why DNS-01:** Allows wildcard cert issuance without opening inbound ports to the internet. Traefik adds a temporary TXT record to Cloudflare DNS to prove domain ownership.

**Verified 2026-11-14:**

- `acme.json` existed and contained 16,050 bytes before the upgrade.
- File permissions were `600`, owned by `gio:gio`.
- Traefik initialized the ACME provider successfully after the upgrade.
- HTTPS tests for Authentik and Immich completed with TLS verification result `0`.

Do not commit `acme.json` or the Cloudflare API token to Git.

---

## Providers

### Docker Provider

Traefik watches the Docker socket on docker-prod-01 for containers. Containers opt in via labels:

```
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<n>.rule=Host(`service.giohosted.com`)"
  - "traefik.http.routers.<n>.entrypoints=websecure"
  - "traefik.http.routers.<n>.tls=true"
  - "traefik.http.services.<n>.loadbalancer.server.port=<container_port>"
```

`exposedbydefault=false` — containers without `traefik.enable=true` are ignored entirely.

The Docker socket is mounted read-only inside Traefik.

**Verified 2026-11-14:** Seventeen running containers, including Traefik, had `traefik.enable=true`.

The router inventory showed that most administrative services use `local-only@docker`. Shelfmark, Seerr, and Audiobookshelf did not have that middleware on their inspected routers.

This does not establish whether those three services are reachable from the internet. DNS, firewall rules, Cloudflare Tunnel configuration, and upstream access controls require separate verification.

**Pending:** Review whether the access policies for Shelfmark, Seerr, and Audiobookshelf match their intended exposure.

### File Provider

Used for routing to services on other VMs (cross-host). Traefik cannot discover containers on other hosts via its local Docker socket, so static routes are defined in configuration files.

- **Config directory:** `/opt/appdata/traefik/config/`
- **Watch:** enabled — changes to config files are picked up without restart

#### Current Static Routes

| File | Hostname | Backend |
|---|---|---|
| `authentik.yml` | `auth.giohosted.com` | `http://192.168.30.13:9000` |
| `immich.yml` | `photos.giohosted.com` | `http://192.168.30.14:2283` |

#### Retired Beszel Route — 2026-11-14

An existing `beszel.yml` file defined:

- **Hostname:** `beszel.giohosted.com`
- **Backend:** `http://192.168.10.20:8090`
- **Middleware:** `local-only@docker`

During the Traefik upgrade review, the hostname returned a gateway timeout from an authorized client.

Direct access from docker-prod-01 to `192.168.10.20:8090` also timed out. The network route traversed the VLAN 30 gateway.

The Beszel hub itself remained operational and was accessible from the operator's existing direct-IP bookmark.

Since hostname-based access was not required, the unused file-provider route was disabled rather than changing inter-VLAN firewall rules.

The configuration was moved from:

`/opt/appdata/traefik/config/beszel.yml`

To:

`/opt/appdata/traefik/beszel.yml.disabled`

The disabled file is outside Traefik's watched configuration directory.

**Verification:**

- `auth.giohosted.com` continued returning HTTP `302`.
- `photos.giohosted.com` continued returning HTTP `200`.
- `beszel.giohosted.com` returned HTTP `404`, confirming the route was no longer configured.
- The Beszel dashboard remained accessible through its direct IP address.

Current Beszel hub access is documented in `services/monitoring/beszel.md`.

---

## Entrypoints

| Name | Port | Notes |
|---|---|---|
| web | 80 | HTTP — immediately redirects to websecure |
| websecure | 443 | HTTPS — all traffic served here; 600-second request read and idle connection timeouts configured |

Both entrypoints also implement the HTTP header-alias and encoded-character restrictions described below.

### Large Uploads — Responding Timeouts

**Verified change (2026-10-31):** Certain Immich iOS video uploads over approximately 500 MB failed with `502 Bad Gateway` after roughly 60 seconds, although smaller uploads worked. Traefik's running `websecure` entrypoint initially had no explicit responding-timeout flags, so its default 60-second request-body `readTimeout` was the leading suspected cause.

In `/opt/stacks/traefik/compose.yaml` on docker-prod-01, the following CLI flags were added under the Traefik service's existing `command:` list:

```
- --entrypoints.websecure.transport.respondingtimeouts.readtimeout=600s
- --entrypoints.websecure.transport.respondingtimeouts.idletimeout=600s
```

- `readTimeout=600s` permits up to 10 minutes to read the complete client request, including large uploads. This is the setting directly relevant to the observed failures.
- `idleTimeout=600s` controls how long an idle keep-alive connection remains open; it does not itself extend an active request-body upload.
- Both are **static entrypoint settings**, not options in `/opt/appdata/traefik/config/immich.yml`.
- Because `websecure` is a shared entrypoint, these values apply to other HTTPS services using it, not only Immich.
- This was a timeout adjustment, **not** a new file-size cap or a change to external exposure. Immich remains LAN-only in the documented standard configuration.

**Validation and deployment performed:**

```
# Before recreation: validated with no error output
docker compose -f /opt/stacks/traefik/compose.yaml config --quiet

# Recreated only the Traefik Compose service
docker compose -f /opt/stacks/traefik/compose.yaml up -d --no-deps traefik

# Verified both flags in the running container
docker inspect "$(docker compose -f /opt/stacks/traefik/compose.yaml ps -q traefik)" \
  --format '{{range .Args}}{{println .}}{{end}}' \
  | grep -i timeout
```

The inspection returned both `readtimeout=600s` and `idletimeout=600s`. Previously failing large videos then uploaded successfully through the Immich iOS app using `https://photos.giohosted.com`.

**Operational note:** Traefik recreation can briefly interrupt routing to internal services. A future change to static entrypoint flags requires Compose validation and container recreation; changes to watched file-provider routes generally do not.

**Version-control status:** The timeout changes are present in the shared `homelab-infra` repository. The subsequent v3.7.19 upgrade preserved both timeout settings.

### HTTP Header Alias Protection — 2026-11-14

Traefik v3.7.19 reported warnings about request headers whose names may be interpreted as aliases of other headers by downstream applications.

Both entrypoints were configured with:

```
- --entrypoints.web.http.aliasheadersstrategy=delete
- --entrypoints.websecure.http.aliasheadersstrategy=delete
```

This removes ambiguous request header names rather than forwarding them or rejecting the entire request.

**Verification:**

- The updated Compose configuration validated successfully.
- Traefik recreated successfully.
- The `aliasHeadersStrategy` startup warnings disappeared.
- Authentik and Immich continued responding normally.
- Both applications were subsequently verified through their browser interfaces.

The setting applies to all requests using these entrypoints.

### Encoded-Character Restrictions — 2026-11-14

Traefik reported a warning about potentially ambiguous URL-encoded characters in request paths.

After reviewing the exposed application inventory, two restrictions were introduced on both entrypoints:

```
- --entrypoints.web.http.encodedcharacters.allowencodedbackslash=false
- --entrypoints.web.http.encodedcharacters.allowencodednullcharacter=false
- --entrypoints.websecure.http.encodedcharacters.allowencodedbackslash=false
- --entrypoints.websecure.http.encodedcharacters.allowencodednullcharacter=false
```

These settings reject:

- Encoded backslash: `%5C`
- Encoded null character: `%00`

Other encoded-character categories were deliberately left at their existing defaults pending a workload-specific compatibility review.

**Validation:**

1. Docker Compose configuration validation passed.
2. An isolated Traefik v3.7.19 container accepted the four options.
3. The production Traefik container was recreated successfully.
4. The encoded-character startup warning disappeared.
5. Requests containing `%00` and `%5C` returned HTTP `400`.
6. Normal Authentik and Immich requests continued to work.

Verified HTTPS results:

| Request | HTTP Status | TLS Verification |
|---|---|---|
| `https://auth.giohosted.com/` | 302 | 0 |
| `https://photos.giohosted.com/` | 200 | 0 |
| `https://photos.giohosted.com/test%00path` | 400 | 0 |
| `https://photos.giohosted.com/test%5Cpath` | 400 | 0 |

The encoded-path tests used `curl --path-as-is` so the encoded paths were transmitted without client-side normalization.

Both Authentik and Immich also passed a subsequent browser-based functional check.

**Important:** These checks verify the specified paths and applications. They do not constitute comprehensive compatibility testing for every Traefik router or application workflow.

---

## Middleware

### local-only@docker

Restricts access to internal VLANs only when explicitly attached to a router.

- **Defined in:** Traefik container labels in `compose.yaml` — NOT in the file provider
- **Allowed ranges:** `192.168.10.0/24`, `192.168.20.0/24`, `192.168.30.0/24`
- **Reference in compose labels:** `local-only@docker`

> **Critical:** Always reference this middleware as `local-only@docker`, never `local-only@file`. An incorrect provider reference can leave the router unable to resolve its middleware. Check Traefik logs and verify routing behavior after any middleware changes.

This middleware is applied to most administrative Docker routers, but it is **not automatically applied to every router**.

During the November 14 inventory, Shelfmark, Seerr, and Audiobookshelf did not have the middleware. Their intended exposure must be reviewed separately.

---

## Dashboard

- **URL:** `https://traefik.giohosted.com`
- **Access:** Internal only — restricted via `local-only@docker` middleware
- **Auth:** None — access controlled by IP allowlist only

---

## Adding a New Service (Docker)

For containers on docker-prod-01, add these labels to the Compose service:

```
networks:
  - proxy

labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<n>.rule=Host(`service.giohosted.com`)"
  - "traefik.http.routers.<n>.entrypoints=websecure"
  - "traefik.http.routers.<n>.tls=true"
  - "traefik.http.routers.<n>.middlewares=local-only@docker"
  - "traefik.http.services.<n>.loadbalancer.server.port=<container_port>"
```

Then add a DNS rewrite in AdGuard: `service.giohosted.com` → `192.168.30.11`

The container must also be on the `proxy` Docker network:

```
networks:
  proxy:
    external: true
```

If a service is intentionally available outside the local-only networks, its exposure must be reviewed and documented rather than simply omitting security middleware without explanation.

---

## Adding a New Service (Cross-Host / File Provider)

For services running on other VMs (auth-prod-01, immich-prod-01, etc.):

1. Create `/opt/appdata/traefik/config/<service>.yml`:

```
http:
  routers:
    <service>:
      rule: "Host(`service.giohosted.com`)"
      entrypoints:
        - websecure
      tls: {}
      service: <service>

  services:
    <service>:
      loadBalancer:
        servers:
          - url: "http://<vm-ip>:<port>"
```

2. Add DNS rewrite in AdGuard: `service.giohosted.com` → `192.168.30.11`

No Traefik restart needed — file provider watches for changes automatically.

> **Important:** Cross-host services must expose their port on the host via a `ports` mapping in their Compose file. Traefik on docker-prod-01 connects to the VM by IP — it cannot reach container ports that aren't exposed to the host network.

Also verify that the destination host's firewall permits connections from docker-prod-01. A configured route alone does not prove backend connectivity.

---

## Exposing a Service Externally via Cloudflare Tunnel

For services that need external access, cloudflared routes traffic through Traefik. The documented pattern is:

1. Deploy the service with Traefik labels (Docker provider) or file provider static route as normal.
2. Add AdGuard DNS rewrite as normal.
3. In Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Published application routes → Add:
    - Hostname: `service.giohosted.com`
    - Service type: `HTTPS`
    - URL: `192.168.30.11`
    - Additional settings → TLS → **No TLS Verify: enabled**

### Why HTTPS and not HTTP?

Using `http://192.168.30.11` can cause an HTTP-to-HTTPS redirect loop with the documented configuration. Use HTTPS for the tunnel origin.

### Why No TLS Verify?

Traefik's certificate is issued for hostnames under `giohosted.com`, not the IP address `192.168.30.11`.

When cloudflared connects by IP, certificate name verification can fail. The documented tunnel setup disables certificate verification on that origin connection.

The Cloudflare Tunnel transport is encrypted; however, disabling origin certificate verification reduces origin-authentication assurance. If this configuration is revisited, using an appropriate origin server name and certificate verification is preferable where supported.

See `services/infra/cloudflared.md` for full cloudflared documentation.

---

## Upgrading Traefik

### Verified Upgrade — 2026-11-14

**Previous version:** v3.6.25  
**New version:** v3.7.19  
**Image:** `traefik:latest`  
**Host:** docker-prod-01

The upgrade followed these steps:

1. Verified the existing running version and Compose configuration.
2. Reviewed the dynamic routes and entrypoint settings.
3. Backed up certificates, dynamic configuration, Compose, and `.env`.
4. Downloaded `traefik:latest` and confirmed it resolved to v3.7.19.
5. Tested the existing Authentik, Beszel, and Immich routing baselines.
6. Changed the Compose image reference from `traefik:v3.6` to `traefik:latest`.
7. Committed and pushed the image configuration change.
8. Recreated only the Traefik service using the downloaded image.
9. Verified the version, startup logs, TLS certificates, and application responses.
10. Applied and verified HTTP header alias and encoded-character hardening in separate changes.
11. Confirmed Authentik and Immich worked through their normal browser interfaces.

### Relevant Git Commits

| Commit | Change |
|---|---|
| `9348c68` | Switch Traefik to latest stable image |
| `8153e1c` | Harden Traefik HTTP header alias handling |
| `d06dac8` | Restrict encoded backslashes and null characters in Traefik |

All three commits were pushed to `homelab-infra`.

**Current synchronization status as of this update:** docker-prod-01 was verified at `d06dac8`. The other three Git checkouts were last verified at `042bbd5` and still require synchronization to the newer Traefik commits. Git synchronization alone does not deploy or restart Traefik on another host.

### Pre-Upgrade Backup

A verified backup was created at:

`/opt/appdata/traefik/backups/pre-upgrade-20261114-220807.tar.gz`

The archive contains:

- `acme.json`
- The dynamic configuration directory as it existed before the upgrade
- `traefik/compose.yaml`
- `traefik/.env`

**Permissions:** `root:root`, mode `600`.

The archive contains certificate private keys and the Cloudflare API token. It must remain protected.

A successful `tar -tzf` listing confirmed that the archive was readable and contained the expected files. A complete restoration exercise was not performed.

### Future Upgrade Procedure

Because `traefik:latest` is a moving tag, do not assume that a future pull will remain within v3.

Before upgrading:

1. Identify the current deployed version and image ID.
2. Review the target release and migration documentation.
3. Verify the current backup and rollback arrangements.
4. Record the currently working routes and TLS behavior.
5. Pull and inspect the intended image separately.
6. Validate Compose and the proposed static flags.
7. Recreate only the Traefik service.
8. Verify startup logs, certificates, application routing, and real application functionality.

Useful commands:

```
cd /opt/stacks/traefik

sudo docker exec traefik traefik version
sudo docker compose config --quiet
sudo docker compose ps traefik

# Pull an image without deploying it
sudo docker pull traefik:latest

# Check the downloaded image version
sudo docker run --rm --network none \
  --entrypoint traefik traefik:latest version
```

Once approved, deploy the already-downloaded image:

```
sudo docker compose up -d \
  --no-deps \
  --pull never \
  traefik
```

Verify the running version and logs:

```
sudo docker exec traefik traefik version
sudo docker compose ps traefik
sudo docker logs --tail 100 traefik
```

**Important:** `--pull never` prevents the deployment command from silently fetching a newer image than the one reviewed.

A future `latest` tag update may involve breaking changes. Do not rely on the current v3.7 compatibility assessment for later releases.

### Rollback Considerations

The previous deployment used `traefik:v3.6`, which was available locally during the upgrade.

If a future Traefik change fails:

1. Stop further deployment attempts.
2. Inspect the container status and logs.
3. Identify whether the failure concerns static configuration, dynamic routing, TLS, or the image itself.
4. Confirm that the intended rollback image remains available locally.
5. Review whether the previous Compose configuration and persistent data are compatible.
6. Restore the previous known-good configuration or image as appropriate.
7. Recreate only Traefik and repeat the routing and TLS checks.

Do not overwrite `acme.json` or restore the entire backup indiscriminately without understanding which component needs recovery.

The backup is a recovery resource, not proof that the complete rollback has been tested.

---

## Directory Structure

```
/opt/stacks/traefik/
  compose.yaml              ← in git
  .env                      ← gitignored (CF_DNS_API_TOKEN)

/opt/appdata/traefik/
  acme.json                 ← TLS cert storage (chmod 600)
  config/                   ← watched file provider config
    authentik.yml           ← static route to auth-prod-01
    immich.yml              ← static route to immich-prod-01
  beszel.yml.disabled       ← retired Beszel route; outside watched directory
  backups/
    pre-upgrade-20261114-220807.tar.gz
```

The `/opt/appdata/traefik/` directory is persistent local application data, not part of the `homelab-infra` Compose repository.

---

## Notes

- Chrome Secure DNS (DoH) can bypass AdGuard and resolve via Cloudflare public DNS — disable or configure it appropriately if internal hostnames aren't resolving correctly.
- `acme.json` must remain restricted to mode `600`.
- The `proxy` Docker network is created by the Traefik Compose stack. Containers requiring local Docker-provider routing must join it.
- Middleware must reference the correct provider, such as `local-only@docker`.
- File-provider routes require reachable remote services and suitable inter-VLAN firewall access.
- Large Immich iOS video uploads were verified working after raising `websecure` responding timeouts to 600 seconds on 2026-10-31.
- Traefik v3.7.19 and its two implemented security controls were verified on 2026-11-14.
- The Beszel hostname route was retired; the hub remains accessible by its direct IP address.
- Additional Docker router exposure review is pending for Shelfmark, Seerr, and Audiobookshelf.
- The current security settings do not guarantee compatibility with all future image versions or every application request path.

---

## Related Documentation

- `runbooks/updating-containers.md` — Docker Compose updates, image management, and rollback.
- `runbooks/git-workflow.md` — Version control and multi-host Git synchronization.
- `services/monitoring/beszel.md` — Beszel hub, agents, and direct-IP management.
- `services/infra/authentik.md` — Authentik deployment and authentication.
- `services/infra/cloudflared.md` — External access through Cloudflare Tunnel.