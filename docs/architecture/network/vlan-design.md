# VLAN Design — Homelab v3

**Last Updated:** 2026-10-29

---

## Overview

Homelab v3 uses VLAN segmentation to separate infrastructure management, trusted clients, production services, IoT devices, and game-server workloads.

Inter-VLAN traffic is controlled by the UniFi Dream Machine SE (UDM-SE), following a default-deny design with explicit exceptions for required communication.

Detailed IP allocations and firewall rules are documented separately:

- `architecture/network/ip-addresses.md`
    
- `architecture/network/firewall-rules.md`
    

The VLANs below describe the established architecture. Where a configuration has not been verified against the current UniFi environment, that limitation is noted.

---

## VLAN 10 — Management (192.168.10.0/24)

Dedicated to infrastructure management interfaces and control-plane services.

**Key devices:**

- UDM-SE — 192.168.10.1
    
- UniFi USW-Pro-Max-24 — 192.168.10.2
    
- nas-prod-01 management interface — 192.168.10.10
    
- pve-prod-01 — 192.168.10.11
    
- pve-prod-02 — 192.168.10.12
    
- pi-prod-01 — 192.168.10.20
    

The NAS management interface is isolated from its service-facing NFS interface on VLAN 30.

Management hosts have broad outbound access to the established internal networks for administrative operations.

Inbound access from other VLANs is restricted to approved administrative ports and explicitly authorized exceptions.

### AI Observer Exception

`observer-prod-01` resides on VLAN 30 and requires access to the Proxmox API on VLAN 10.

A targeted firewall exception allows:

- Source: `192.168.30.17`
    
- Destination: `192.168.10.11`
    
- Protocol: TCP
    
- Port: `8006`
    

This connection supports the read-only Proxmox MCP integration.

The exception does not grant unrestricted access from VLAN 30 to VLAN 10.

See `services/infra/ai-observer.md`.

---

## VLAN 20 — Trusted (192.168.20.0/24)

Dedicated to trusted client devices, including personal laptops, desktops, and phones.

Trusted devices may access internal services on VLAN 30 and approved management interfaces on VLAN 10.

Access to management interfaces is limited to designated administrative ports.

### Historical v2 Configuration

VLAN 20 was retained from Homelab v2.

During the v3 migration, the previous Proxmox and Docker hosts used this VLAN:

- v2 Proxmox — 192.168.20.2
    
- v2 Docker host — 192.168.20.10
    

These addresses are historical references. Their current operational status and decommissioning state have not been reverified during this update.

---

## VLAN 30 — Services (192.168.30.0/24)

Dedicated to production service VMs, LXCs, and service-facing storage interfaces.

**Key services include:**

- dns-prod-01 — Primary AdGuard Home
    
- docker-prod-01 — Application and media Docker host
    
- pbs-prod-01 — Proxmox Backup Server
    
- auth-prod-01 — Authentik identity provider
    
- immich-prod-01 — Immich photo service
    
- dns-prod-02 — Secondary AdGuard Home
    
- nas-prod-01 — NFS and data interface
    
- observer-prod-01 — AI Observer
    

Production Docker containers generally use their host's networking and Docker networks rather than individual VLAN IP addresses.

Services routed through the central Traefik instance use docker-prod-01. Other services may be accessed directly through their respective hosts.

### NAS Data Interface

The NAS uses `192.168.30.16` for NFS and other service-facing traffic.

Its management interface remains on VLAN 10 at `192.168.10.10`.

This separation allows production services to access required storage without granting general access to the NAS management interface.

### Network Restrictions

The established firewall design denies general outbound access from VLAN 30 to:

- VLAN 10 — Management
    
- VLAN 20 — Trusted
    
- VLAN 40 — IoT
    

Explicit exceptions may be configured for narrowly scoped infrastructure dependencies.

Examples include the AI Observer's access to the Proxmox API and the documented Beszel monitoring exception.

VLAN 30 retains outbound internet access for container updates, image pulls, external APIs, and tunnel connectivity.

---

## VLAN 40 — IoT (192.168.40.0/24)

Dedicated to smart home devices, printers, and other untrusted endpoints.

**Gateway:** `192.168.40.1`  
**Documented DHCP pool:** `192.168.40.100+`

The intended security policy permits internet access while preventing general access to other internal VLANs.

The original migration plan identified seven existing IoT devices to move from VLAN 30 to VLAN 40 during Phase 1.

The completion status of those individual device migrations has not been reverified as part of this update.

IoT DNS handling also requires verification because older documentation contains conflicting references to public DNS and UDM-SE interception to AdGuard.

See `architecture/network/firewall-rules.md`.

---

## VLAN 50 — Game Servers

Dedicated network segment for game-server workloads, separating them from general production services.

### Verified Deployment

|Setting|Value|
|---|---|
|VLAN ID|50|
|Name|Game Servers|
|Proxmox VM|dragonwilds-server|
|VMID|106|
|Proxmox host|pve-prod-01|
|Bridge|vmbr0|
|Configured VLAN tag|50|

The VLAN tag was verified through the live Proxmox integration on October 30, 2026 UTC.

### Pending Verification

The following configuration details have not yet been verified:

- Network subnet and gateway
    
- DHCP configuration
    
- Game-server guest IP address
    
- Inter-VLAN firewall policies
    
- Internet access restrictions
    
- Port forwarding and inbound access policies
    

The existence of VLAN 50 does not establish its isolation or permitted traffic flows.

These details should be recorded only after reviewing the current UniFi configuration.

See `architecture/network/ip-addresses.md` for the current guest inventory and verification status.

---

## Design Principles

Homelab v3's VLAN architecture follows these principles:

1. **Separate management from services.** Production workloads should not have unrestricted access to infrastructure control interfaces.
    
2. **Keep trusted clients distinct.** Personal devices can access required services without sharing the same broadcast domain.
    
3. **Isolate untrusted devices.** IoT devices should not have general access to trusted clients or infrastructure management.
    
4. **Use targeted exceptions.** Required cross-VLAN dependencies should use the narrowest practical source, destination, and port restrictions.
    
5. **Maintain dedicated workload segments where useful.** Game servers use VLAN 50 rather than sharing the general Services VLAN.
    
6. **Verify actual enforcement.** A VLAN assignment alone does not guarantee isolation; UniFi firewall policy and connectivity must also be checked.
    

---

## Related Documentation

- `architecture/network/ip-addresses.md` — IP allocations, interfaces, and subnets.
    
- `architecture/network/firewall-rules.md` — Inter-VLAN access policies and exceptions.
    
- `architecture/compute/pve-prod-01.md` — Proxmox node and guest inventory.
    
- `services/infra/ai-observer.md` — AI Observer network access and security architecture.
    
- `services/monitoring/beszel.md` — Beszel monitoring connectivity.
    

---

## Verification Status

**Verified:**

- AI Observer uses VLAN 30 with IP `192.168.30.17`.
    
- AI Observer can reach the Proxmox API through its targeted network exception.
    
- Dragonwilds VM 106 has Proxmox VLAN tag 50.
    

**Documented design, not independently re-audited:**

- General UniFi inter-VLAN firewall rules.
    
- VLAN 10–40 segmentation policies.
    

**Unresolved:**

- VLAN 50 addressing and firewall policy.
    
- IoT DNS behavior.
    
- Historical v2 device decommissioning status.
    

Update the corresponding architecture and firewall documentation after these details have been verified.