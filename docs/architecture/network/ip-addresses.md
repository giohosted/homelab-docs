# IP Address Plan

**Last updated:** 2026-03-13

---

## VLAN 10 — Management (192.168.10.0/24)

Strictly for infrastructure control plane interfaces — devices you administer, not services you consume.

| Device                             | IP            | Interface      | Notes                                                                                                                                                                       |
| ---------------------------------- | ------------- | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UDM-SE                             | 192.168.10.1  | Built-in       | Gateway — already configured, do not change                                                                                                                                 |
| Core Switch (UniFi USW-Pro-Max-24) | 192.168.10.2  | Management     | Switch management interface                                                                                                                                                 |
| UPS (if networked later)           | 192.168.10.3  | —              | Reserved — NUT handles shutdown via USB for now                                                                                                                             |
| nas-prod-01 — **mgmt only**        | 192.168.10.10 | Onboard 2.5GbE | Unraid web UI access only. NFS is NOT served here.                                                                                                                          |
| pve-prod-01 (MS-A2)                | 192.168.10.11 | 2.5GbE RJ45    | Proxmox management UI                                                                                                                                                       |
| pve-prod-02 (Optiplex)             | 192.168.10.12 | GbE RJ45       | Proxmox management UI                                                                                                                                                       |
| pi-prod-01 (Raspberry Pi)          | 192.168.10.20 | GbE RJ45       | QDevice + Uptime Kuma + Beszel server. Migrated from VLAN 20 (192.168.20.8) in Phase 3. VLAN 10 outbound reach covers all VLANs — Beszel/Kuma probes work without extra rules. |
| 192.168.10.30–.49                  | —             | —              | Reserved — future expansion / IPMI / OOB                                                                                                                                    |
| 192.168.10.100+                    | —             | —              | DHCP pool if needed                                                                                                                                                         |

> **Note:** VLAN 10 outbound reach to all VLANs requires **Auto Allow Return Traffic** enabled on the Allow Management to All firewall rule — without it, return packets to VLAN 10 hosts are dropped. See `firewall-rules.md`.

---

## VLAN 20 — Trusted (192.168.20.0/24)

Personal laptops, phones, trusted client devices. v2 services remain here during migration — left untouched until Phase 4 cutover.

| Device | IP | Notes |
|--------|----|-------|
| v2 Proxmox host | 192.168.20.2 | Decommission after Phase 4 cutover |
| v2 Docker host | 192.168.20.10 | Decommission after Phase 4 cutover |
| Personal devices | DHCP | Laptops, phones, trusted clients |

---

## VLAN 30 — Services (192.168.30.0/24)

All v3 VMs, LXCs, and the NAS data interface. Containers inside VMs are accessed via Traefik on the VM's IP — they do not get individual VLAN IPs.

| Device                                      | IP            | Interface             | Notes                                                                         |
| ------------------------------------------- | ------------- | --------------------- | ----------------------------------------------------------------------------- |
| dns-prod-01 (AdGuard LXC, pve-prod-01)      | 192.168.30.10 | Virtual (VLAN tag 30) | Primary DNS — AdGuard Home                                                    |
| docker-prod-01 (media/apps VM, pve-prod-01) | 192.168.30.11 | Virtual (VLAN tag 30) | All media stack containers behind Traefik                                     |
| pbs-prod-01 (PBS VM, pve-prod-02)           | 192.168.30.12 | Virtual (VLAN tag 30) | Proxmox Backup Server                                                         |
| auth-prod-01 (Authentik VM, pve-prod-01)    | 192.168.30.13 | Virtual (VLAN tag 30) | Authentik IdP — dedicated VM — Pending Phase 4                                |
| immich-prod-01 (Immich VM, pve-prod-01)     | 192.168.30.14 | Virtual (VLAN tag 30) | Immich — isolated for resource tuning — Pending Phase 4                       |
| dns-prod-02 (AdGuard LXC, pve-prod-02)      | 192.168.30.15 | Virtual (VLAN tag 30) | Secondary DNS — synced from dns-prod-01 via adguardhome-sync                  |
| nas-prod-01 — **data/NFS**                  | 192.168.30.16 | X710 Port 2 SFP+      | NFS exports, Plex container traffic. Services mount NFS from this IP only.    |
| plex (Docker container, nas-prod-01)        | 192.168.30.2  | X710 Port 2 SFP+      | Plex Media Server — Docker container on nas-prod-01, bound to eth1 (VLAN 30)  |
| 192.168.30.100+                             | —             | —                     | DHCP / sandbox / test / future k3s nodes                                      |

> **NAS dual-interface design:** nas-prod-01 has two logical network presences. The **onboard 2.5GbE at 192.168.10.10** is management-only (Unraid UI). The **X710 Port 2 at 192.168.30.16** is the data plane (NFS, Plex). Services on VLAN 30 mount NFS from 192.168.30.16 without ever crossing into VLAN 10. The firewall hard-blocks VLAN 30 → VLAN 10, so a compromised service container cannot reach the Unraid management UI.

> **X710 Port 1** (DAC to MS-A2) is on a private point-to-point storage subnet — not on any VLAN, not reachable from the LAN. Used exclusively for high-speed Proxmox storage traffic.

---

## VLAN 40 — IoT (192.168.40.0/24)

Internet access only. No inter-VLAN routing.

| Device | IP | Notes |
|--------|----|-------|
| Gateway | 192.168.40.1 | UDM-SE |
| IoT devices | DHCP 192.168.40.100+ | Smart home, printers, untrusted endpoints |

---

## Storage Network — Point-to-Point (No VLAN)

| Link | Interface A | Interface B | Subnet | Purpose |
|------|-------------|-------------|--------|---------|
| NAS ↔ MS-A2 | nas-prod-01 X710 Port 1 | pve-prod-01 SFP+ Port 1 | 10.0.0.0/30 | Dedicated 10GbE storage traffic. Not on LAN switch. Not routed. |

> This link is isolated — only these two devices can see it. Used for Proxmox storage traffic between pve-prod-01 and nas-prod-01. Keeps storage bandwidth off the main LAN switch entirely.