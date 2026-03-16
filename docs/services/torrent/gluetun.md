# Gluetun

**Role:** VPN killswitch container for qBittorrent  
**Host:** docker-prod-01 (192.168.30.11)  
**Image:** qmcgaw/gluetun:latest  
**Compose:** `/opt/stacks/torrent/compose.yaml`  
**Appdata:** `/opt/appdata/gluetun/`  
**Last Updated:** 2026-03-16

---

## Overview

Gluetun is a VPN client container that creates a WireGuard tunnel to ProtonVPN. qBittorrent runs inside Gluetun's network namespace — meaning qBittorrent has no independent network identity and all its traffic is forced through the VPN tunnel. If the VPN drops, qBittorrent loses all connectivity entirely (killswitch by design).

Gluetun also handles ProtonVPN port forwarding — it obtains a forwarded port from ProtonVPN and runs a script to update qBittorrent's listen port automatically on every connection.

---

## Network Architecture
```
Internet
  ↑↓ (encrypted WireGuard tunnel)
ProtonVPN Chicago P2P server
  ↑↓
Gluetun container (tun0 interface)
  ↑↓ (shared network namespace)
qBittorrent (no independent network — uses Gluetun's interfaces)
```

Because qBittorrent shares Gluetun's network namespace via `network_mode: service:gluetun`:
- qBittorrent has no entry on the `proxy` Docker network
- Traefik labels for qBittorrent's WebUI live on the **Gluetun container**, not qBittorrent
- ARR instances connect to qBittorrent using hostname `gluetun` (not `qbittorrent`)
- Port 8080 (qBittorrent WebUI) is accessible through Gluetun's container IP

---

## VPN Configuration

| Setting | Value |
|---------|-------|
| Provider | ProtonVPN |
| Type | WireGuard |
| Server cities | Chicago |
| Port forward only | Enabled — only connects to P2P servers that support port forwarding |
| Port forwarding | Enabled |
| Firewall interface | eth0 |
| Timezone | America/Chicago |

Private key stored in `/opt/stacks/torrent/.env` as `WIREGUARD_PRIVATE_KEY` — never committed to git.

---

## Port Forwarding

ProtonVPN assigns a forwarded port on every VPN connection. Gluetun handles this automatically:

1. VPN tunnel establishes
2. Gluetun requests a forwarded port from ProtonVPN
3. Port is written to `/tmp/gluetun/forwarded_port`
4. `VPN_PORT_FORWARDING_UP_COMMAND` runs `/gluetun/scripts/qbit-set-port.sh {{PORT}}`
5. Script logs into qBittorrent WebUI via API and sets the listen port to match

The forwarded port changes on every VPN reconnection. The script handles this automatically — no manual intervention needed.

### Port Forward Script

Located at `/opt/appdata/gluetun/scripts/qbit-set-port.sh` — mounted read-only into the container at `/gluetun/scripts/`.

The script:
- Reads `QBIT_USER` and `QBIT_PASS` from Gluetun's environment (`.env` file)
- Logs into qBittorrent WebUI via cookie-based auth
- Sets `listen_port`, `random_port=false`, `upnp=false` via the qBittorrent API
- Retries up to 30 times with 2 second delays (handles slow qBittorrent startup)

### Verifying Port Forwarding
```bash
# Check what port Gluetun forwarded
docker logs gluetun 2>&1 | grep -A6 "port forwarded"

# Verify qBittorrent is actually listening on that port
docker exec gluetun netstat -ulnp
```

The VPN interface (`10.2.0.x`) should show the forwarded port. The `0.0.0.0` entries are local peer discovery — ignore those.

---

## Traefik Routing

Traefik labels for qBittorrent's WebUI live on Gluetun since qBittorrent has no independent network presence:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.qbittorrent.rule=Host(`qbittorrent.giohosted.com`)"
  - "traefik.http.routers.qbittorrent.entrypoints=websecure"
  - "traefik.http.routers.qbittorrent.tls=true"
  - "traefik.http.routers.qbittorrent.middlewares=local-only@docker"
  - "traefik.http.services.qbittorrent.loadbalancer.server.port=8080"
```

---

## Directory Structure
```
/opt/stacks/torrent/
  compose.yaml                        ← in git (shared with qBittorrent and qBitrr)
  .env                                ← gitignored (WIREGUARD_PRIVATE_KEY, QBIT_USER, QBIT_PASS)

/opt/appdata/gluetun/
  servers.json                        ← auto-generated server list, do not edit
  scripts/
    qbit-set-port.sh                  ← port forward script, mounted read-only
```

---

## Environment Variables

| Variable | Source | Notes |
|----------|--------|-------|
| `WIREGUARD_PRIVATE_KEY` | `.env` | ProtonVPN WireGuard private key — never commit |
| `QBIT_USER` | `.env` | qBittorrent WebUI username — used by port script |
| `QBIT_PASS` | `.env` | qBittorrent WebUI password — used by port script |

---

## Troubleshooting

### VPN not connecting
Check Gluetun logs: `docker logs gluetun`. Look for WireGuard connection errors or DNS failures. ProtonVPN Chicago servers are P2P-capable — if no servers are available Gluetun will retry automatically.

### Port not updating in qBittorrent
The port script runs once when Gluetun first obtains the forwarded port. If qBittorrent restarts after the script ran, the listen port reverts to the value in its config file. Fix: restart the full stack so Gluetun re-runs the script after qBittorrent is ready.
```bash
cd /opt/stacks/torrent
docker compose down
docker compose up -d
```

### Verifying VPN is active
```bash
# Should show ProtonVPN Chicago IP, not your home IP
docker logs gluetun 2>&1 | grep "Public IP"
```

### qBittorrent WebUI not reachable
Since the WebUI port is exposed through Gluetun, if Gluetun is unhealthy the WebUI goes down too. Check Gluetun health first:
```bash
docker inspect gluetun | grep -i "Status"
```

---

## Notes

- Gluetun is marked `healthy` in Docker when the VPN tunnel is up and health targets (cloudflare.com, github.com) are reachable
- `HEALTH_RESTART_VPN=on` — Gluetun automatically restarts the VPN connection if health checks fail
- Do not expose port 8080 directly on the host — all access goes through Traefik via the proxy network
- The `servers.json` file is auto-generated by Gluetun on first run — do not manually edit or delete it