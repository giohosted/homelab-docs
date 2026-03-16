# cloudflared

**Role:** Cloudflare Tunnel — external access to internal services  
**Host:** docker-prod-01 (192.168.30.11)  
**Tunnel name:** homelab-v3  
**Compose:** `/opt/stacks/cloudflared/compose.yaml`  
**Appdata:** None — stateless  
**Last Updated:** 2026-03-16

---

## Deployment

- **Image:** cloudflare/cloudflared:latest
- **Compose:** `/opt/stacks/cloudflared/compose.yaml`
- **Network:** proxy
- **Tunnel token:** stored in `/opt/stacks/cloudflared/.env` (gitignored)

## Container Variables

| Variable | Value |
|----------|-------|
| TUNNEL_TOKEN | see `.env` |

---

## Overview

cloudflared creates an outbound tunnel from docker-prod-01 to Cloudflare's edge network. This allows external access to internal services without opening any ports on the UDM-SE firewall. All external traffic enters via Cloudflare and is forwarded through the tunnel to Traefik, which handles internal routing.

cloudflared is not a replacement for Traefik — it works alongside it. The traffic path for external requests is:
```
Internet → Cloudflare edge → cloudflared tunnel → Traefik (192.168.30.11) → service
```

Traefik handles all internal routing regardless of whether the request came from cloudflared or directly from the LAN.

---

## Exposed Services

| Hostname | Cloudflare Route | Notes |
|----------|-----------------|-------|
| `auth.giohosted.com` | `https://192.168.30.11` | Authentik |
| `audiobooks.giohosted.com` | `https://192.168.30.11` | Audiobookshelf — Wave 4 |
| `request.giohosted.com` | `https://192.168.30.11` | Seerr |
| `shelf.giohosted.com` | `https://192.168.30.11` | Shelfmark — Wave 4 |

---

## Cloudflare Tunnel Configuration

All routes are configured in Cloudflare Zero Trust dashboard under:
**Networks → Tunnels → homelab-v3 → Published application routes**

### Route Settings

- **Service URL:** `https://192.168.30.11`
- **No TLS Verify:** enabled on all routes

### Why No TLS Verify?

Traefik's certificate is a hostname-based wildcard cert (`*.giohosted.com`) issued by Let's Encrypt. When cloudflared connects to Traefik by IP (`192.168.30.11`), TLS verification fails because the cert doesn't cover bare IP addresses — only hostnames.

This is safe because:
- The cloudflared → Traefik hop never leaves the private LAN
- The Cloudflare Tunnel itself is encrypted end-to-end via QUIC
- There is no man-in-the-middle risk on a private 192.168.30.0/24 network

The end user's connection (browser → Cloudflare edge) is fully protected by Cloudflare's own valid TLS certificate.

### Why HTTP causes a redirect loop

Using `http://192.168.30.11` instead of `https://` causes an infinite redirect loop:
1. cloudflared sends HTTP request to Traefik on port 80
2. Traefik's HTTP → HTTPS redirect rule fires
3. cloudflared follows the redirect back to Cloudflare
4. Loop repeats indefinitely

Solution: always use `https://192.168.30.11` with No TLS Verify.

---

## Adding a New Externally Exposed Service

1. Deploy the service on docker-prod-01 with Traefik labels (or add a file provider static route for cross-host services)
2. Add AdGuard DNS rewrite: `service.giohosted.com` → `192.168.30.11`
3. In Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Published application routes → Add:
   - Hostname: `service.giohosted.com`
   - Service type: `HTTPS`
   - URL: `192.168.30.11`
   - Additional settings → TLS → No TLS Verify: enabled

---

## Services NOT Exposed via Tunnel

| Service | Reason |
|---------|--------|
| Traefik dashboard | Admin tool — internal only |
| Dockman | Admin tool — internal only |
| Proxmox UI | Admin tool — internal only |
| Unraid UI | Admin tool — internal only |
| ARR stack | LAN only — no external access needed |
| qBittorrent | LAN only — no external access needed |
| qBitrr | LAN only — no external access needed |
| PBS | Admin tool — internal only |
| Plex | Direct port forward on 32400 — Cloudflare ToS prohibits video streaming via tunnel |

---

## Cloudflare DNS Changes Made During v3 Build

| Record | Action | Reason |
|--------|--------|--------|
| `*` A record → 69.216.122.32 | Deleted | v2 NPM wildcard — no longer needed |
| `giohosted.com` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `photos` A record → 69.216.122.32 | Deleted | v2 NPM — no longer needed |
| `auth` Tunnel record → homelab-v2 | Deleted | Dead v2 tunnel — recreated in homelab-v3 tunnel |

---

## Notes

- cloudflared is stateless — no appdata directory, no backup required
- Tunnel token is stored in `/opt/stacks/cloudflared/.env` (gitignored) — if lost, generate a new one in Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Overview → Refresh token
- cloudflared does not need to be on the `proxy` Docker network — it connects to Traefik by IP, not by container name
- Plex is intentionally excluded from the tunnel — direct port forward on 32400 is used instead