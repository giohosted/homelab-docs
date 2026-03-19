# cloudflared

**Role:** Cloudflare Tunnel — external access to internal services  
**Host:** docker-prod-01 (192.168.30.11)  
**Tunnel name:** homelab-v3  
**Compose:** `/opt/stacks/cloudflared/compose.yaml`  
**Appdata:** None — stateless  
**Last Updated:** 2026-03-18

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

| Hostname | Cloudflare Route | CF Access | Notes |
|----------|-----------------|-----------|-------|
| `auth.giohosted.com` | `https://192.168.30.11` | No | Authentik — has its own auth |
| `audiobooks.giohosted.com` | `https://192.168.30.11` | Yes | Audiobookshelf — service token bypass for mobile app |
| `request.giohosted.com` | `https://192.168.30.11` | No | Seerr — has its own auth |
| `shelf.giohosted.com` | `https://192.168.30.11` | Yes | Shelfmark |
| `photos.giohosted.com` | Not active | N/A | Immich — tunnel route added day before wedding (October), removed after |

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

## Cloudflare Access

Cloudflare Access adds an authentication layer in front of externally exposed services. Users must authenticate before reaching the service. Currently enabled for:

- **Audiobookshelf** (`audiobooks.giohosted.com`) — browser users authenticate via Authentik OIDC; mobile app bypasses via service token (see below)
- **Shelfmark** (`shelf.giohosted.com`) — login via Authentik OIDC through CF Access

### ABS Mobile App — Service Token Bypass

The ABS mobile app cannot complete CF Access's browser-based auth challenge. A CF Access service token (`abs-mobile`) is configured as a Service Auth policy on the Audiobookshelf CF Access application. The mobile app sends the token headers with every request, bypassing the browser challenge.

- **Service token name:** `abs-mobile`
- **CF Access application:** `Audiobookshelf`
- **Policy name:** `abs-mobile-token` — action: SERVICE AUTH, order: 1 (evaluated before all other policies)
- **Token storage:** Password manager (Client ID + Client Secret)
- **Mobile app config:** Custom Headers → `CF-Access-Client-Id` and `CF-Access-Client-Secret`

See `audiobookshelf.md` for full policy order and mobile app setup details.

---

## Immich — Temporary Tunnel for Wedding (October)

Immich is LAN-only by default. The tunnel route for `photos.giohosted.com` is added temporarily for the wedding event and removed afterwards.

**Traefik file provider route:** `/opt/appdata/traefik/config/immich.yml` — already configured, points to `http://192.168.30.14:2283`

**Checklist (day before wedding):**
1. Cloudflare Zero Trust → Networks → Tunnels → homelab-v3 → Add public hostname:
   - Subdomain: `photos` / Domain: `giohosted.com`
   - Type: `HTTPS` / URL: `192.168.30.11`
   - Additional settings → TLS → No TLS Verify: enabled
2. Test QR code link from phone on LTE
3. No CF Access policy — guests access the shared album directly, protected by Immich's built-in album password

**After the wedding (a few days later):**
1. Remove the `photos.giohosted.com` route from the tunnel
2. Immich returns to LAN-only

> **Immich server external domain** is set to `https://photos.giohosted.com` in Administration → Server Settings — shared album QR codes will use this as the base URL.

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
| CWA | LAN only — no external access needed |
| PBS | Admin tool — internal only |
| Immich | LAN-only by default — temporary tunnel for wedding event only |
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