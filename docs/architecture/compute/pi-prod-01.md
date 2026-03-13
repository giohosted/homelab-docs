# pi-prod-01 — Raspberry Pi 4B

**Role:** Proxmox QDevice (cluster tiebreaker) + Uptime Kuma + Beszel server  
**Status:** Active  
**Last Updated:** 2026-03-13

---

## Hardware

| Component | Detail |
|-----------|--------|
| **Model** | Raspberry Pi 4B |
| **RAM** | 4GB |
| **Boot Storage** | SD card |
| **Networking** | 1x GbE RJ45 built-in |
| **OS** | Raspberry Pi OS (Debian GNU/Linux 12 Bookworm) |

---

## Network

| Field | Value |
|-------|-------|
| **IP** | 192.168.10.20 |
| **VLAN** | 10 — Management |
| **Gateway** | 192.168.10.1 |
| **DNS** | 9.9.9.9 (public — VLAN 10 does not use AdGuard) |
| **Switch port profile** | Mgmt-Only (access port, VLAN 10) |
| **Network manager** | NetworkManager (not dhcpcd) |

### Static IP Configuration
Set via nmcli — dhcpcd.conf is present but not active on Raspberry Pi OS Bookworm.

To modify static IP in future, use:
```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.10.20/24 ipv4.gateway 192.168.10.1 ipv4.dns 9.9.9.9 ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

---

## Roles

### 1. Proxmox QDevice (Primary Role)
Acts as a lightweight quorum tiebreaker for the 2-node Proxmox cluster (pve-prod-01 + pve-prod-02). Prevents split-brain scenarios where both nodes lose contact with each other and both attempt to become primary.

- **Package:** corosync-qdevice + corosync-qnetd
- **Initialized via:** `pvecm qdevice setup 192.168.10.20` run from pve-prod-01
- **Votes contributed:** 1 (brings total cluster votes to 3, quorum at 2)
- Must remain on VLAN 10 alongside Proxmox nodes — QDevice communication uses management network

### 2. Uptime Kuma (Future — Phase 5)
Service uptime monitoring. Initiates checks outbound — no inbound connections required from VLAN 30.

### 3. Beszel Server (Future — Phase 5)
Host and VM metrics server. Beszel agents on VLAN 30 hosts connect inbound to pi-prod-01.

> ⚠️ **Pending firewall rule (add during Phase 5 Beszel setup):**  
> VLAN 30 → 192.168.10.20 TCP 45876 ALLOW  
> Required for Beszel agents on VLAN 30 to reach the Beszel server on VLAN 10.  
> See `architecture/network/firewall-rules.md` for full details.

---

## v2 → v3 Role Changes

| Role | v2 | v3 |
|------|----|----|
| AdGuard Home (primary DNS) | ✅ Running on Pi | ❌ Moved to dns-prod-01 LXC (192.168.30.10) |
| AdGuard Home (secondary DNS) | ❌ Not present | ❌ Moved to dns-prod-02 LXC (192.168.30.15) |
| Uptime Kuma | ✅ Running on Pi | ✅ Stays on Pi |
| Beszel server | ✅ Running on Pi | ✅ Stays on Pi |
| QDevice | ❌ Not present | ✅ New in v3 |
| VLAN | 20 — Trusted | 10 — Management |
| IP | 192.168.20.8 | 192.168.10.20 |

---

## SSH Access

Root SSH login is currently enabled (required for QDevice setup). 

> ⚠️ **TODO:** Disable root SSH once QDevice is confirmed stable over several days.  
> Edit `/etc/ssh/sshd_config` — set `PermitRootLogin no` then `sudo systemctl restart ssh`

Normal access:
```bash
ssh gdelgado@192.168.10.20
```

---

## Notes

- SD card boot is acceptable given the lightweight role — monitor for SD card issues, consider USB SSD if problems arise
- Pi must remain powered and reachable for cluster quorum — if Pi goes offline, cluster still operates but loses tiebreaker protection
- Do NOT move Pi off VLAN 10 — QDevice must be on the same management network as Proxmox nodes