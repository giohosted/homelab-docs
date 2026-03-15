# Traefik

**Role:** Reverse proxy, TLS termination, internal service routing  
**Host:** docker-prod-01 (192.168.30.11)  
**Version:** v3.6.10  
**Compose:** `/opt/stacks/traefik/compose.yaml`  
**Appdata:** `/opt/appdata/traefik/`  
**Last Updated:** 2026-03-15

---

## Overview

Traefik is the v3 reverse proxy, replacing Nginx Proxy Manager from v2. It handles:
- Routing internal requests to the correct container by hostname
- TLS termination with a wildcard cert for `*.giohosted.com`
- Cross-host routing to services on other VMs (auth-prod-01, immich-prod-01) via the file provider
- HTTP → HTTPS redirect on all requests

Traefik is **not** exposed to the internet. All traffic is internal. External access is handled separately by cloudflared (Cloudflare Tunnel).

---

## How It Works

When a request comes in for e.g. `https://sonarr.giohosted.com`:

1. Your device asks AdGuard "what's the IP for sonarr.giohosted.com?"
2. AdGuard returns `192.168.30.11` (DNS rewrite)
3. Your device connects to `192.168.30.11:443`
4. Traefik receives the request, matches the hostname to the sonarr router, and forwards to the sonarr container
5. Response comes back through Traefik to your device

Cloudflare is only involved in **certificate issuance** — not in the traffic path for internal services.

---

## TLS Certificate

- **Type:** Wildcard — covers `giohosted.com` and `*.giohosted.com`
- **Issuer:** Let's Encrypt (via ACME)
- **Challenge:** DNS-01 via Cloudflare API token
- **Storage:** `/opt/appdata/traefik/acme.json` (chmod 600)
- **Auto-renewal:** Yes — Traefik renews automatically 30 days before expiry. No manual intervention needed.
- **Why DNS-01:** Allows wildcard cert issuance without opening any ports to the internet. Traefik adds a temporary TXT record to Cloudflare DNS to prove domain ownership.

---

## Providers

### Docker Provider
Traefik watches the Docker socket on docker-prod-01 for containers. Containers opt in via labels:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<name>.rule=Host(`service.giohosted.com`)"
  - "traefik.http.routers.<name>.entrypoints=websecure"
```
`exposedbydefault=false` — containers without `traefik.enable=true` are ignored entirely.

### File Provider
Used for routing to services on other VMs (cross-host). Traefik cannot discover containers on other hosts via Docker socket, so static routes are defined in config files.

- **Config directory:** `/opt/appdata/traefik/config/`
- **Watch:** enabled — changes to config files are picked up without restart

#### Current static routes

| File | Hostname | Backend |
|------|----------|---------|
| `authentik.yml` | `auth.giohosted.com` | `http://192.168.30.13:9000` |

---

## Entrypoints

| Name | Port | Notes |
|------|------|-------|
| web | 80 | HTTP — immediately redirects to websecure |
| websecure | 443 | HTTPS — all traffic served here |

---

## Dashboard

- **URL:** `https://traefik.giohosted.com`
- **Access:** Internal only — restricted to 192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24 via `local-only` middleware
- **Auth:** None — access controlled by IP allowlist only (internal network assumed trusted)

---

## Adding a New Service (Docker)

For containers on docker-prod-01, add these labels to the compose service:
```yaml
networks:
  - proxy

labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<name>.rule=Host(`service.giohosted.com`)"
  - "traefik.http.routers.<name>.entrypoints=websecure"
  - "traefik.http.services.<name>.loadbalancer.server.port=<container_port>"
```

Then add a DNS rewrite in AdGuard: `service.giohosted.com` → `192.168.30.11`

The container must also be on the `proxy` Docker network:
```yaml
networks:
  proxy:
    external: true
```

---

## Adding a New Service (Cross-Host / File Provider)

For services running on other VMs (auth-prod-01, immich-prod-01, etc.):

1. Create `/opt/appdata/traefik/config/<service>.yml`:
```yaml
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

---

## Exposing a Service Externally via Cloudflare Tunnel

For services that need external access, cloudflared routes traffic through Traefik. The pattern is always:

1. Deploy the service with Traefik labels (Docker provider) or file provider static route as normal
2. Add AdGuard DNS rewrite as normal
3. In Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Published application routes → Add:
   - Hostname: `service.giohosted.com`
   - Service type: `HTTPS`
   - URL: `192.168.30.11`
   - Additional settings → TLS → **No TLS Verify: enabled**

### Why HTTPS and not HTTP?
Using `http://192.168.30.11` causes an infinite redirect loop — Traefik's HTTP→HTTPS redirect fires, cloudflared follows it, repeat forever. Always use `https://`.

### Why No TLS Verify?
Traefik's cert is a hostname wildcard (`*.giohosted.com`) — it doesn't cover bare IPs. When cloudflared connects to `192.168.30.11` by IP, TLS verification fails. No TLS Verify bypasses this check.

This is safe because:
- The cloudflared → Traefik hop is entirely on the private LAN (192.168.30.0/24)
- The Cloudflare Tunnel is already encrypted end-to-end via QUIC
- The user's browser connection to Cloudflare's edge is fully protected by Cloudflare's own valid TLS

See `services/infra/cloudflared.md` for full cloudflared documentation.

---

## Directory Structure
```
/opt/stacks/traefik/
  compose.yaml       ← in git
  .env               ← gitignored (CF_DNS_API_TOKEN)

/opt/appdata/traefik/
  acme.json          ← TLS cert storage (chmod 600, gitignored)
  config/            ← file provider config directory
    authentik.yml    ← static route to auth-prod-01
```

---

## Notes

- Chrome Secure DNS (DoH) bypasses AdGuard and resolves via Cloudflare public DNS — disable it in Chrome settings if internal hostnames aren't resolving correctly
- acme.json must be chmod 600 — Traefik will refuse to start if permissions are wrong
- The `proxy` Docker network is created by the Traefik compose — all containers that need Traefik routing must join this network