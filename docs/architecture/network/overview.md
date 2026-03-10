# Network Overview

**Last Updated:** 2026-03-10 **Status:** Phase 1 complete — VLANs, firewall rules, and switch port profiles configured and validated.

---

## Hardware

|Device|Model|IP|Role|
|---|---|---|---|
|Router / Firewall|UniFi UDM-SE|192.168.10.1|Gateway, firewall, DHCP server, WireGuard VPN, UniFi controller|
|Core Switch|UniFi USW-Pro-Max-24|192.168.10.2|Managed 2.5GbE switch, VLAN trunking to all hosts|
|Access Point|UniFi U6-Pro|192.168.10.200|Wireless — wall/ceiling mount, PoE via UDM-SE port 7|

All three devices are on VLAN 10 (Management). The UDM-SE and switch are connected via a 0.5M 10Gtek DAC (SFP+). See `hardware/rack-layout.md` for full cabling map.

---

## VLAN Design

Four VLANs. VLAN 1 (default untagged) is not used as the management VLAN — management is on a dedicated VLAN ID to prevent devices from accidentally landing on it.

|VLAN|Name|Subnet|Purpose|
|---|---|---|---|
|10|Management|192.168.10.0/24|Infrastructure control plane — routers, switches, hypervisors, NAS management UI|
|20|Trusted|192.168.20.0/24|Personal devices, desktop, phones. v2 services live here during migration.|
|30|Services|192.168.30.0/24|All v3 VMs, LXCs, NAS data interface. New services live here.|
|40|IoT|192.168.40.0/24|Smart home devices, printers, untrusted endpoints. Internet only.|

---

## IP Address Summary

### VLAN 10 — Management (192.168.10.0/24)

|Device|IP|Interface|Notes|
|---|---|---|---|
|UDM-SE|192.168.10.1|Built-in|Gateway — do not change|
|USW-Pro-Max-24|192.168.10.2|Management|Switch management interface|
|UPS (reserved)|192.168.10.3|—|Reserved — NUT handles shutdown via USB for now|
|nas-prod-01 mgmt|192.168.10.10|Onboard 2.5GbE|Unraid web UI only — NFS is NOT served here|
|pve-prod-01|192.168.10.11|2.5GbE RJ45|Proxmox management UI|
|pve-prod-02|192.168.10.12|GbE RJ45|Proxmox management UI|
|U6-Pro|192.168.10.200|—|Access point management|
|pi-prod-01|192.168.10.20|GbE RJ45|QDevice + Uptime Kuma + Beszel. Currently on VLAN 20 until Phase 3 cutover.|
|192.168.10.30–.49|—|—|Reserved — future expansion / IPMI / OOB|
|192.168.10.100+|—|—|DHCP pool if needed|

### VLAN 20 — Trusted (192.168.20.0/24)

|Device|IP|Notes|
|---|---|---|
|Gio PC (Desktop)|192.168.20.216|Primary workstation|
|pi-prod-01 (temporary)|DHCP|On VLAN 20 until Phase 3 cutover to VLAN 10|
|Synology NAS|DHCP|Trusted — backup replication target|
|Personal devices|DHCP|Laptops, phones, trusted clients|
|v2 Docker host|192.168.20.10|Decommission after Phase 4 cutover|

### VLAN 30 — Services (192.168.30.0/24)

|Device|IP|Interface|Notes|
|---|---|---|---|
|dns-prod-01|192.168.30.10|Virtual (VLAN 30)|Primary AdGuard Home LXC on pve-prod-01|
|docker-prod-01|192.168.30.11|Virtual (VLAN 30)|All media/app containers behind Traefik|
|pbs-prod-01|192.168.30.12|Virtual (VLAN 30)|Proxmox Backup Server VM on pve-prod-02|
|auth-prod-01|192.168.30.13|Virtual (VLAN 30)|Authentik IdP — dedicated VM on pve-prod-01|
|immich-prod-01|192.168.30.14|Virtual (VLAN 30)|Immich — isolated VM on pve-prod-01|
|dns-prod-02|192.168.30.15|Virtual (VLAN 30)|Secondary AdGuard Home LXC on pve-prod-02|
|nas-prod-01 data|192.168.30.16|X710 Port 2 SFP+|NFS exports and Plex container traffic|
|192.168.30.100+|—|—|DHCP / sandbox / test / future k3s nodes|

> Containers inside VMs are accessed via Traefik on the VM's IP — they do not get individual VLAN IPs.

### VLAN 40 — IoT (192.168.40.0/24)

|Device|IP|Notes|
|---|---|---|
|Gateway|192.168.40.1|UDM-SE|
|IoT devices|DHCP 192.168.40.100+|Smart home, printers, untrusted endpoints|

Current IoT devices: Eufy Homebase, Eufy doorbell camera, Eufy indoor camera, Cove alarm panel, Govee outdoor lights, TP-Link Tapo power strip, LG TV, Nest thermostat.

### Storage Network — Point-to-Point (No VLAN)

|Link|Interface A|Interface B|Subnet|Purpose|
|---|---|---|---|---|
|NAS ↔ MS-A2|nas-prod-01 X710 Port 1|pve-prod-01 SFP+ Port 1|10.0.0.0/30|Dedicated 10GbE storage — not on LAN switch, not routed|

---

## Firewall Rules

Default policy is DENY ALL inter-VLAN. Explicit ALLOW rules only. Enforced at UDM-SE under Settings → Routing → Firewall → LAN In.

|#|Source|Destination|Port / Protocol|Action|
|---|---|---|---|---|
|1|Trusted (20)|Services (30)|Any|ALLOW|
|2|Trusted (20)|Management (10)|TCP 443, 8006, 22|ALLOW|
|3|Trusted (20)|Management (10)|Any other|DENY|
|4|Services (30)|Management (10)|Any|DENY|
|5|Services (30)|Trusted (20)|Any|DENY|
|6|Services (30)|IoT (40)|Any|DENY|
|7|IoT (40)|Any internal|Any|DENY|

All VLANs can reach WAN by default — no explicit rule needed. See `networking/firewall-rules.md` for full rule rationale, per-VLAN summaries, and the NAS security model.

---

## DNS

- **Primary:** dns-prod-01 at 192.168.30.10 (AdGuard Home LXC on pve-prod-01)
- **Secondary:** dns-prod-02 at 192.168.30.15 (AdGuard Home LXC on pve-prod-02, synced via adguardhome-sync)
- **DHCP:** UDM-SE pushes 192.168.30.10 as DNS to all VLANs
- **IoT DNS:** UDM-SE intercepts port 53 from VLAN 40 and forwards to AdGuard — IoT devices get ad-blocking without a direct path to VLAN 30
- **Split-horizon:** `*.giohosted.com` resolves internally to Traefik IP via AdGuard DNS rewrites

> **Current state:** During Phase 1/2, DNS is still served by pi-prod-01 (AdGuard on VLAN 20). dns-prod-01 and dns-prod-02 will be stood up in Phase 3 and DHCP updated at that time.

---

## Switch Port Profiles

|Profile|Native VLAN|Tagged VLAN Mgmt|Used For|
|---|---|---|---|
|Trunk - All VLANs|Management (10)|Allow All|Proxmox hosts, Pi (post-cutover)|
|Mgmt - Only|Management (10)|Allow All|nas-prod-01 onboard 2.5GbE|
|Services Access|Services (30)|Block All|nas-prod-01 X710 Port 2|
|Trusted Access|Trusted (20)|Block All|Desktop, personal devices, Synology|
|IoT Access|IoT (40)|Block All|Smart home devices, printers|

See `hardware/rack-layout.md` for the full patch panel → switch port map with profile assignments per port.

---

## Remote Access

|Method|Used For|Notes|
|---|---|---|
|Port forward 32400|Plex|Direct — Plex handles its own TLS|
|Cloudflare Tunnel|Audiobookshelf, Shelfmark, Seerr, Authentik|No port forwarding required|
|WireGuard VPN|Secure remote LAN access|Hosted on UDM-SE — carry forward from v2|

---

## Related Files

- `networking/firewall-rules.md` — full firewall rule set with rationale
- `networking/ip-addresses.md` — complete IP address plan
- `networking/dns-rewrites.md` — AdGuard DNS rewrite entries
- `networking/wireguard-vpn.md` — WireGuard config
- `hardware/rack-layout.md` — physical cabling and switch port map