# AdGuard Home

**Host:** dns-prod-01 (192.168.30.10) — primary  
**Host:** dns-prod-02 (192.168.30.15) — secondary  
**Container type:** LXC (Debian 12)  
**Status:** Running  
**Last Updated:** 2026-03-16

---

## Deployment

- **Primary:** dns-prod-01 — LXC on pve-prod-01, CT ID 100
- **Secondary:** dns-prod-02 — LXC on pve-prod-02, synced from primary via adguardhome-sync
- **Version:** v0.107.73
- **Web UI (primary):** `http://192.168.30.10:3000`
- **Web UI (secondary):** `http://192.168.30.15:3000`
- **DNS port:** 53 (UDP/TCP)

> AdGuard Home runs as an LXC, not a Docker container. It is managed via the Proxmox UI, not Dockman.

---

## Overview

AdGuard Home is the DNS server for all VLANs. It provides:
- Ad and tracker blocking across all network clients
- Split-horizon DNS — `*.giohosted.com` resolves internally to Traefik (192.168.30.11)
- DNS rewrites for all internal services
- Upstream DNS resolution via encrypted DoT/DoH resolvers

The secondary instance (dns-prod-02) is a read-only replica synced from the primary via adguardhome-sync. It provides DNS redundancy — if pve-prod-01 goes down, dns-prod-02 continues serving DNS from pve-prod-02.

---

## Network

| Instance | IP | VLAN | Host |
|----------|----|------|------|
| dns-prod-01 (primary) | 192.168.30.10 | 30 — Services | pve-prod-01 |
| dns-prod-02 (secondary) | 192.168.30.15 | 30 — Services | pve-prod-02 |

DHCP on the UDM-SE pushes `192.168.30.10` as the DNS server to all VLANs. dns-prod-02 is a failover — it is not pushed via DHCP by default.

---

## Upstream DNS

Upstream resolvers use DNS-over-TLS (DoT) for encrypted resolution:

| Resolver | Address |
|----------|---------|
| Cloudflare | `tls://1.1.1.1` |
| Quad9 | `tls://9.9.9.9` |

> VLAN 10 (Management) does not use AdGuard — Proxmox nodes and pi-prod-01 use `9.9.9.9` directly as their DNS server. AdGuard runs on VLAN 30 and management hosts are configured before AdGuard exists.

---

## DNS Rewrites

All `*.giohosted.com` services point to Traefik at `192.168.30.11`. Rewrites are managed on the primary (dns-prod-01) and synced to the secondary automatically.

| Domain | Answer |
|--------|--------|
| `traefik.giohosted.com` | 192.168.30.11 |
| `auth.giohosted.com` | 192.168.30.11 |
| `dockman.giohosted.com` | 192.168.30.11 |
| `sonarr-tv.giohosted.com` | 192.168.30.11 |
| `sonarr-anime.giohosted.com` | 192.168.30.11 |
| `radarr-1080p.giohosted.com` | 192.168.30.11 |
| `radarr-4k.giohosted.com` | 192.168.30.11 |
| `prowlarr.giohosted.com` | 192.168.30.11 |
| `bazarr.giohosted.com` | 192.168.30.11 |
| `profilarr.giohosted.com` | 192.168.30.11 |
| `maintainerr.giohosted.com` | 192.168.30.11 |
| `request.giohosted.com` | 192.168.30.11 |
| `tautulli.giohosted.com` | 192.168.30.11 |
| `qbittorrent.giohosted.com` | 192.168.30.11 |
| `qbitrr.giohosted.com` | 192.168.30.11 |

> Always add new DNS rewrites to dns-prod-01 (primary) only — adguardhome-sync propagates them to dns-prod-02 automatically within 30 minutes.

---

## IoT DNS Interception

IoT devices on VLAN 40 are not permitted to reach VLAN 30 directly. The UDM-SE intercepts DNS queries from VLAN 40 on port 53 and forwards them to AdGuard at 192.168.30.10. IoT devices get ad-blocking without needing a direct path to the Services VLAN.

---

## Adding a New DNS Rewrite

1. Go to AdGuard Web UI at `http://192.168.30.10:3000`
2. Filters → DNS Rewrites → Add DNS Rewrite
3. Domain: `service.giohosted.com`
4. Answer: `192.168.30.11`
5. Save — adguardhome-sync will propagate to dns-prod-02 within 30 minutes

---

## Troubleshooting

### Internal hostnames not resolving
- Verify the DNS rewrite exists in AdGuard primary (dns-prod-01)
- Check your client is using `192.168.30.10` as its DNS server
- If on Windows, flush DNS cache: `ipconfig /flushdns`
- If using Chrome, disable Secure DNS (DoH) — it bypasses AdGuard entirely: Chrome Settings → Privacy and Security → Security → Use secure DNS → Off

### AdGuard Web UI not reachable
- Verify dns-prod-01 LXC is running in Proxmox UI
- Try the secondary at `http://192.168.30.15:3000`

---

## Notes

- Rewrites added directly to dns-prod-02 will be overwritten by the next adguardhome-sync run — always add to primary only
- adguardhome-sync runs every 30 minutes — see `services/infra/adguardhome-sync.md`
- Chrome Secure DNS (DoH) bypasses AdGuard entirely — disable it on any client where internal hostnames need to resolve correctly