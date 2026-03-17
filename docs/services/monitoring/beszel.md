# Beszel

**Host (Hub):** pi-prod-01 (192.168.10.20)
**Status:** Running
**Last Updated:** 2026-03-17

---

## Overview

Beszel is a lightweight server monitoring tool. The hub runs as a systemd binary service on pi-prod-01 and collects metrics from agents installed on each host.

---

## Hub

- **Install method:** Binary (systemd service)
- **Service name:** `beszel-hub`
- **Binary location:** `/opt/beszel/beszel`
- **Web UI:** `http://192.168.10.20:8090`
- **Access:** From VLAN 20 (Trusted) — port 8090 is included in `MGMT_ADMIN_PORTS` firewall group

### Useful commands
```bash
sudo systemctl status beszel-hub
sudo systemctl restart beszel-hub
sudo systemctl stop beszel-hub
```

---

## Agents

| Host | IP | Install Method | Agent Port |
|------|----|---------------|------------|
| pi-prod-01 | 192.168.10.20 | Binary (systemd) | 45876 |
| pve-prod-01 | 192.168.10.11 | Binary (systemd) | 45876 |
| pve-prod-02 | 192.168.10.12 | Binary (systemd) | 45876 |
| docker-prod-01 | 192.168.30.11 | Docker Compose | 45876 |
| auth-prod-01 | 192.168.30.13 | Docker Compose | 45876 |
| immich-prod-01 | 192.168.30.14 | Docker Compose | 45876 |
| nas-prod-01 | 192.168.10.10 | Unraid Docker (CA) | 45876 |

Docker Compose agents are at `/opt/stacks/beszel-agent/` on each host. Keys and tokens are in `.env` files (gitignored).

---

## Firewall Rule

A targeted allow rule was added in UniFi to permit Beszel agents on VLAN 30 to reach the hub on VLAN 10:

- **Source:** VLAN 30 (Services)
- **Destination:** 192.168.10.20 (pi-prod-01)
- **Port:** TCP 45876
- **Action:** ALLOW

VLAN 10 hosts (pve-prod-01, pve-prod-02, pi-prod-01 itself) reach the hub without a firewall rule — intra-VLAN traffic is unrestricted.

---

## Notes

- nas-prod-01 temperature sensors `nct6798_auxtin2` and `nct6798_auxtin4` report 127°C — this is a sentinel value for unconnected sensor inputs, not a real reading. Ignore these.
- Hub port 8090 is added to `MGMT_ADMIN_PORTS` UniFi port group for Trusted → Management access.