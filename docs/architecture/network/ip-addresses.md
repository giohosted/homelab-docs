# IP Address Plan

**Last updated:** 2026-10-29

---

## VLAN 10 — Management (192.168.10.0/24)

Strictly for infrastructure control plane interfaces — devices you administer, not services you consume.

|Device|IP|Interface|Notes|
|---|---|---|---|
|UDM-SE|192.168.10.1|Built-in|Gateway — already configured, do not change|
|Core Switch (UniFi USW-Pro-Max-24)|192.168.10.2|Management|Switch management interface|
|UPS (if networked later)|192.168.10.3|—|Reserved — NUT handles shutdown via USB for now|
|nas-prod-01 — **mgmt only**|192.168.10.10|Onboard 2.5GbE|Unraid web UI access only. NFS is NOT served here.|
|pve-prod-01 (MS-A2)|192.168.10.11|2.5GbE RJ45|Proxmox management UI and API|
|pve-prod-02 (Optiplex)|192.168.10.12|GbE RJ45|Proxmox management UI|
|pi-prod-01 (Raspberry Pi)|192.168.10.20|GbE RJ45|QDevice + Uptime Kuma + Beszel server. Migrated from VLAN 20 (192.168.20.8) in Phase 3. VLAN 10 outbound reach covers all VLANs — Beszel/Kuma probes work without extra rules.|
|192.168.10.30–.49|—|—|Reserved — future expansion / IPMI / OOB|
|192.168.10.100+|—|—|DHCP pool if needed|

> **Note:** VLAN 10 outbound reach to all VLANs requires **Auto Allow Return Traffic** enabled on the Allow Management to All firewall rule — without it, return packets to VLAN 10 hosts are dropped. See `firewall-rules.md`.

---

## VLAN 20 — Trusted (192.168.20.0/24)

Personal laptops, phones, and trusted client devices.

The historical v2 migration entries below are retained from the original IP plan. Their current operating status and any associated decommissioning have not been reverified as part of this update.

|Device|IP|Notes|
|---|---|---|
|v2 Proxmox host|192.168.20.2|Historical v2 allocation — decommission after Phase 4 cutover; current status not verified|
|v2 Docker host|192.168.20.10|Historical v2 allocation — decommission after Phase 4 cutover; current status not verified|
|Personal devices|DHCP|Laptops, phones, trusted clients|
|Synology NAS|192.168.20.9|Historical allocation — planned move to VLAN 30 after Phase 4 cutover; current IP not reverified|

---

## VLAN 30 — Services (192.168.30.0/24)

Production service VMs, LXCs, and the NAS data interface.

Containers inside VMs are generally accessed through their host VM's IP address rather than receiving individual VLAN IPs. Services exposed through the central Traefik instance use docker-prod-01.

Not every v3 VM is on VLAN 30. The Dragonwilds dedicated game server uses VLAN 50, which is documented separately below.

|Device|IP|Interface|Notes|
|---|---|---|---|
|dns-prod-01 (AdGuard LXC, pve-prod-01)|192.168.30.10|Virtual (VLAN tag 30)|Primary DNS — AdGuard Home|
|docker-prod-01 (media/apps VM, pve-prod-01)|192.168.30.11|Virtual (VLAN tag 30)|All media stack containers behind Traefik. Mounts `192.168.30.16:/mnt/user/data` as `/data` via NFSv3 — single mount covering all media and downloads.|
|pbs-prod-01 (PBS VM, pve-prod-02)|192.168.30.12|Virtual (VLAN tag 30)|Proxmox Backup Server|
|auth-prod-01 (Authentik VM, pve-prod-01)|192.168.30.13|Virtual (VLAN tag 30)|Authentik IdP — dedicated VM — deployed Phase 4 Wave 1|
|immich-prod-01 (Immich VM, pve-prod-02)|192.168.30.14|Virtual (VLAN tag 30)|Immich — on pve-prod-02 — deployed Phase 4 Wave 5. Mounts `192.168.30.16:/mnt/user/photos` as `/data/photos` via NFSv3.|
|dns-prod-02 (AdGuard LXC, pve-prod-02)|192.168.30.15|Virtual (VLAN tag 30)|Secondary DNS — synced from dns-prod-01 via adguardhome-sync|
|nas-prod-01 — **data/NFS**|192.168.30.16|X710 Port 2 SFP+|NFS exports, Plex container traffic. Services mount NFS from this IP only.|
|observer-prod-01 (AI Observer VM, pve-prod-01)|192.168.30.17|Virtual (VLAN tag 30, ens18)|AI Observer — Proxmox MCP and OpenAI Secure MCP Tunnel. Dedicated read-only API access to pve-prod-01 (192.168.10.11:8006).|
|plex (Docker container on nas-prod-01)|192.168.30.2|X710 Port 2 SFP+|Plex Media Server — Docker container on nas-prod-01, bound to eth1 (VLAN 30)|
|192.168.30.100+|—|—|DHCP / sandbox / test / future k3s nodes|

### AI Observer — Network Configuration

The observer VM is deployed on pve-prod-01 as VM 107.

|Setting|Value|
|---|---|
|Hostname|observer-prod-01|
|IP address|192.168.30.17/24|
|VLAN|30 — Services|
|Network interface|ens18|
|Proxmox bridge|vmbr0|
|Proxmox VLAN tag|30|
|Gateway|192.168.30.1|
|Primary DNS|192.168.30.10|
|Secondary DNS|192.168.30.15|

**Verification:** The IP address, subnet prefix, and interface name were verified through the live Proxmox QEMU guest agent on 2026-10-30 UTC. Gateway and DNS values reflect the documented deployment configuration and were not separately queried during this update.

#### Network Access

The observer requires:

- Outbound HTTPS to the OpenAI Secure MCP Tunnel control plane.
    
- HTTPS access to the Proxmox API at `192.168.10.11:8006`.
    
- Localhost access to the Proxmox MCP server at `127.0.0.1:8080`.
    
- Localhost access to the tunnel health endpoint at `127.0.0.1:8081`.
    

The Proxmox API connection is permitted through a narrowly scoped UniFi firewall exception:

|Source|Destination|Protocol|Port|Action|
|---|---|---|---|---|
|192.168.30.17|192.168.10.11|TCP|8006|Allow|

This rule does not grant general Services-to-Management network access.

No publicly exposed MCP endpoint, Traefik route, or inbound internet port forwarding is configured for AI Observer.

See:

- `services/infra/ai-observer.md`
    
- `architecture/network/firewall-rules.md`
    
- `runbooks/ai-observer-operations.md` 
    

### NAS Dual-Interface Design

nas-prod-01 has two logical network presences:

- **Onboard 2.5GbE at 192.168.10.10:** Management-only (Unraid UI).
    
- **X710 Port 2 at 192.168.30.16:** Data plane (NFS, Plex).
    

Services on VLAN 30 mount NFS from `192.168.30.16` without crossing into VLAN 10.

The firewall denies general VLAN 30 → VLAN 10 access. Narrow, explicitly documented exceptions — such as the AI Observer's access to the Proxmox API — are permitted without opening the entire management network.

The AI Observer exception does not grant access to the Unraid management UI.

> **X710 Port 1** (DAC to MS-A2) is on a private point-to-point storage subnet — not on any VLAN, not reachable from the LAN. See Storage Network section below.

---

## VLAN 40 — IoT (192.168.40.0/24)

Internet access only. No inter-VLAN routing.

|Device|IP|Notes|
|---|---|---|
|Gateway|192.168.40.1|UDM-SE|
|IoT devices|DHCP 192.168.40.100+|Smart home, printers, untrusted endpoints|

---

## VLAN 50 — Game Servers

Dedicated network segment for game-server workloads.

The VLAN tag is verified in the Proxmox configuration for `dragonwilds-server` (VM 106). The network subnet, gateway, DHCP configuration, firewall policies, and the guest's assigned IP address have not been verified during this documentation update.

|Device|VLAN|IP|Notes|
|---|---|---|---|
|dragonwilds-server (VM, pve-prod-01)|50|Not verified|RuneScape: Dragonwilds dedicated server. Proxmox VM 106, attached to vmbr0 with VLAN tag 50. VM was stopped during verification.|

### Pending Network Verification

Before recording the full VLAN 50 IP plan, verify:

- Configured subnet and gateway in UniFi.
    
- Static or DHCP addressing for the game-server VM.
    
- Current guest IP address.
    
- Inter-VLAN firewall policies and isolation.
    
- Intended internet exposure and any port-forwarding rules.
    

Do not infer the subnet or IP address from VLAN ID 50 alone.

---

## Storage Network — Point-to-Point (No VLAN)

|Link|Interface A|Interface B|IP A|IP B|Subnet|Purpose|
|---|---|---|---|---|---|---|
|NAS ↔ MS-A2|nas-prod-01 X710 Port 1 (eth2)|pve-prod-01 SFP+ Port 1 (nic3)|10.0.0.2/30|10.0.0.1/30|10.0.0.0/30|Dedicated 10GbE storage traffic. Not on LAN switch. Not routed.|

> This link is isolated — only these two devices can see it. Proxmox on pve-prod-01 uses this link for NFS storage mounts (nas-isos). VMs running on pve-prod-01 (including docker-prod-01) cannot access this link — they use VLAN 30 (192.168.30.16) for NFS instead. pve-prod-02 cannot access this link at all — it uses `nas-isos-p2` which mounts from 192.168.30.16.