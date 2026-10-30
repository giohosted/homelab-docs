# Inter-VLAN Firewall Rules

**Enforced at:** UniFi Dream Machine SE (UDM-SE)  
**Default policy:** DENY ALL inter-VLAN traffic. Explicit ALLOW rules only.  
**Last updated:** 2026-10-29

---

## VLAN Reference

|VLAN|Name|Subnet|Key Members|
|---|---|---|---|
|10|Management|192.168.10.0/24|UDM-SE, switch, pve-prod-01, pve-prod-02, pi-prod-01, nas-prod-01 management interface|
|20|Trusted|192.168.20.0/24|Personal laptops, phones, trusted client devices|
|30|Services|192.168.30.0/24|Production VMs and LXCs, observer-prod-01, nas-prod-01 data/NFS interface|
|40|IoT|192.168.40.0/24|Smart home devices, printers, untrusted endpoints|
|50|Game Servers|Not yet verified|dragonwilds-server (VM 106), configured with VLAN tag 50|

> **NAS dual-interface note:** nas-prod-01 has two network presences. The **onboard 2.5GbE** (192.168.10.10) is on VLAN 10 — Unraid management UI only. The **X710 Port 2 SFP+** (192.168.30.16) is on VLAN 30 — NFS exports, Plex container traffic, and service-facing data. Production services use the VLAN 30 interface for NAS data access.

> **VLAN 50 note:** The VLAN tag is verified through Proxmox, but its subnet, gateway, and UniFi firewall policies have not been verified. Do not assume it inherits VLAN 30 rules. See `ip-addresses.md`.

---

## Firewall Rules Table

The table below represents the documented firewall policy and intended evaluation order. The current UniFi rule list has not been independently re-audited during this documentation update.

|#|Name|Source|Destination|Port / Protocol|Action|Reason|
|---|---|---|---|---|---|---|
|1|Allow Management to All|Management (10)|Trusted + Services + IoT|Any|ALLOW|Management hosts must reach the networks they administer. Auto Allow Return Traffic enabled.|
|2|Allow Trusted to Services|Trusted (20)|Services (30)|Any|ALLOW|Trusted clients access internal services.|
|3|Allow Trusted to Mgmt|Trusted (20)|Management (10)|TCP 443, 8006, 22|ALLOW|Administrative access to Proxmox, Unraid, and SSH. Beszel port 8090 requires separate verification.|
|4|Block Trusted to Mgmt Other|Trusted (20)|Management (10)|Any other|DENY|Prevent unrestricted access to management interfaces.|
|5|Allow Services to Beszel|Services (30)|192.168.10.20|TCP 45876|ALLOW|Documented targeted Beszel exception. Traffic direction and current rule configuration should be verified.|
|6|Allow Observer to Proxmox API|192.168.30.17|192.168.10.11|TCP 8006|ALLOW|Dedicated AI Observer access to the Proxmox API.|
|7|Block Services to Mgmt|Services (30)|Management (10)|Any other|DENY|Prevent general Services-to-Management access, including the Unraid UI.|
|8|Block Services to Trusted|Services (30)|Trusted (20)|Any|DENY|Prevent services from initiating connections to trusted client devices.|
|9|Block Services to IoT|Services (30)|IoT (40)|Any|DENY|Prevent production services from initiating connections to IoT devices.|
|10|Block IoT to All Internal|IoT (40)|Internal networks|Any|DENY|Isolate IoT devices from internal infrastructure.|

**Rule evaluation order:** Allow rules for narrowly authorized traffic must be evaluated before the broader deny rules they override.

In particular:

- The AI Observer exception must take precedence over `Block Services to Mgmt`.
    
- The Beszel exception must take precedence over `Block Services to Mgmt`.
    
- Trusted administrative access must take precedence over `Block Trusted to Mgmt Other`.
    
- The general Services-to-Management block must remain in effect for all other traffic.
    

The numeric identifiers above describe the documented rule order, not immutable UniFi rule IDs.

### Auto Allow Return Traffic

Rule 1 requires **Auto Allow Return Traffic** to be enabled.

Without it, Management VLAN hosts may initiate connections to VLAN 30 but fail to receive replies.

This issue was discovered during Phase 3 when pve-prod-01 could not reach pbs-prod-01 or other Services VLAN guests despite the outbound allow rule being present.

The same principle applies to targeted rules: established return traffic must be permitted without unnecessarily opening the reverse direction for new connections.

---

## AI Observer — Proxmox API Exception

### Purpose

`observer-prod-01` hosts the AI Observer service, which provides read-only Proxmox monitoring and diagnostics through MCP.

The observer resides on VLAN 30, while the Proxmox management API resides on VLAN 10.

The default Services-to-Management policy would otherwise prevent the observer from accessing Proxmox.

A narrowly scoped firewall exception was implemented during deployment.

### Rule Configuration

|Field|Value|
|---|---|
|Documented rule name|Allow Observer to Proxmox API|
|Source|192.168.30.17|
|Source VLAN|30 — Services|
|Destination|192.168.10.11|
|Destination VLAN|10 — Management|
|Protocol|TCP|
|Destination port|8006|
|Action|ALLOW|
|Required priority|Above Block Services to Mgmt|
|Scope|One source IP, one destination IP, one service port|

The rule grants the observer access only to the API endpoint on pve-prod-01.

It does not authorize the observer to access the full Management VLAN, the Unraid administration interface, SSH on Proxmox, or other management ports.

Access to other Proxmox nodes is not implicitly permitted by this exception.

### Verification

During AI Observer deployment:

1. The observer initially could not reach the Proxmox API.
    
2. A targeted UniFi firewall exception was implemented.
    
3. An unauthenticated HTTPS connection to the Proxmox API subsequently returned HTTP 401, confirming network reachability.
    
4. An authenticated MCP request successfully queried the Proxmox cluster.
    
5. A later end-to-end query from ChatGPT through OpenAI Secure MCP Tunnel successfully returned live cluster information.
    

**Result:** The observer-to-Proxmox API connection is operational.

The successful connection verifies functional access but does not independently prove the exact present-day UniFi rule name, priority, or absence of additional permissive rules.

Those details should be confirmed during the next UniFi firewall audit.

### Additional Security Controls

The network exception is only one layer of the AI Observer security model.

Additional controls include:

- Dedicated Proxmox account: `observer@pve`.
    
- Dedicated API token: `observer@pve!proxmox-mcp`.
    
- Proxmox `PVEAuditor` role.
    
- MCP write, delete, and execution capabilities disabled.
    
- MCP server bound to `127.0.0.1:8080` on the observer.
    
- OpenAI Secure MCP Tunnel using outbound HTTPS.
    
- Restricted OpenAI runtime API key.
    
- Protected credential files outside Git.
    

The observer's API credentials are read-only. Network reachability alone does not grant Proxmox administrative permissions.

### Outbound Tunnel Traffic

The tunnel client initiates an outbound connection to OpenAI. No inbound firewall exception or public reverse-proxy route is needed.

|Source|Destination|Port|Purpose|
|---|---|---|---|
|observer-prod-01|OpenAI tunnel control plane|TCP 443|Outbound secure MCP tunnel|
|tunnel-client process|127.0.0.1:8080|TCP 8080|Local Proxmox MCP endpoint|
|Proxmox MCP container|192.168.10.11|TCP 8006|Proxmox API|
|Local health checks|127.0.0.1:8081|TCP 8081|Tunnel readiness and diagnostics|

The localhost listeners are not exposed through the UniFi firewall.

**Related documentation:** `services/infra/ai-observer.md`

---

## Beszel — Management VLAN Exception

Beszel runs its monitoring hub on `pi-prod-01` (192.168.10.20) in VLAN 10.

The existing `services/monitoring/beszel.md` documentation records a targeted rule permitting traffic involving Services VLAN agents and the hub.

|Field|Documented Value|
|---|---|
|Source|Services VLAN (30)|
|Destination|192.168.10.20|
|Port|TCP 45876|
|Action|ALLOW|
|Purpose|Beszel monitoring connectivity|

This exception is also present in the historical firewall rules table.

### Documentation Reconciliation

An earlier version of this firewall document contained a contradictory section describing this same rule as pending Phase 5 implementation.

That pending entry is removed from this version because the Beszel service document records the rule as implemented.

However, the exact current UniFi configuration and traffic initiation direction have not been independently verified during this update.

The Beszel service documentation additionally states that TCP 8090 was added to `MGMT_ADMIN_PORTS` for access to the hub UI, whereas the previous firewall rules table listed only TCP 443, 8006, and 22.

**Pending verification:**

- Confirm that the targeted Beszel rule exists in UniFi.
    
- Confirm the source, destination, direction, and port match the actual Beszel agent connection model.
    
- Confirm whether TCP 8090 is permitted from Trusted to Management.
    
- Reconcile the `MGMT_ADMIN_PORTS` group with the firewall table.
    

Until verified, do not broaden the firewall rule to all management hosts or ports.

See `services/monitoring/beszel.md`.

---

## Per-VLAN Summary

### VLAN 10 — Management

- **Outbound:** Documented access to Trusted, Services, IoT, and internet.
    
- **Inbound allowed from Trusted:** Administrative ports as defined by UniFi policy.
    
- **Inbound allowed from Services:** Only specifically authorized exceptions, including AI Observer and the documented Beszel exception.
    
- **Inbound denied from Services otherwise:** General access to management interfaces.
    
- **Inbound denied from IoT:** General internal access.
    
- **Hosts:** UDM-SE (10.1), switch (10.2), pve-prod-01 (10.11), pve-prod-02 (10.12), nas-prod-01 management (10.10), pi-prod-01 (10.20).
    

Access involving VLAN 50 remains unverified.

### VLAN 20 — Trusted

- **Outbound allowed to:** Services (30) on documented general access policy; Management (10) on approved administrative ports; internet.
    
- **Outbound denied to:** Other Management ports not explicitly authorized.
    
- **Hosts:** Personal laptops, phones, and trusted client devices.
    

### VLAN 30 — Services

- **Outbound allowed to:** Services (30) on the local network; internet; specifically authorized Management VLAN exceptions.
    
- **Outbound denied to:** Other Management destinations and ports, Trusted VLAN, and IoT VLAN.
    
- **Hosts:** Production VMs, LXCs, observer-prod-01 (192.168.30.17), nas-prod-01 data interface (192.168.30.16).
    

**Exception:** The AI Observer may access `192.168.10.11:8006`.

This does not change the general Services-to-Management deny policy.

### VLAN 40 — IoT

- **Intended outbound access:** Internet only.
    
- **Intended outbound restrictions:** No general access to Management, Trusted, or Services.
    
- **Hosts:** Smart home devices, printers, untrusted endpoints.
    

The current DNS interception behavior requires verification; see DNS Handling below.

### VLAN 50 — Game Servers

- **Purpose:** Dedicated game-server network segment.
    
- **Verified guest:** `dragonwilds-server`, Proxmox VM 106, configured with VLAN tag 50.
    
- **Subnet:** Not verified.
    
- **Gateway:** Not verified.
    
- **Firewall policy:** Not verified.
    
- **Port forwards:** Not verified as part of this documentation review.
    

No VLAN 50 allow or deny rules have been added to the main table because the current UniFi configuration has not been inspected.

Do not assume VLAN 50 is isolated from or permitted to reach the other VLANs based solely on its role or tag.

See `architecture/network/ip-addresses.md`.

---

## NAS Security Model

The NAS holds backups, photos, and other irreplaceable data.

Its dual-interface design intentionally separates management access from service-facing data traffic.

```
docker-prod-01 (VLAN 30)
  → NFS mount
  → nas-prod-01 X710 Port 2 (192.168.30.16)
  → Allowed within VLAN 30

docker-prod-01 (VLAN 30)
  → Unraid management UI
  → nas-prod-01 onboard 2.5GbE (192.168.10.10)
  → Denied by general Services-to-Management policy

observer-prod-01 (VLAN 30)
  → Proxmox API (192.168.10.11:8006)
  → Allowed by targeted AI Observer exception

observer-prod-01 (VLAN 30)
  → Unraid management UI (192.168.10.10)
  → Not authorized by the AI Observer exception

Trusted laptop (VLAN 20)
  → Unraid management UI (192.168.10.10:443)
  → Allowed by documented administrative access policy
```

A compromised container on docker-prod-01 may reach NFS resources already accessible to that host, but the documented firewall policy prevents it from directly reaching the Unraid management interface.

The AI Observer exception does not authorize access to the NAS management interface.

Actual enforcement should be revalidated through periodic firewall testing.

---

## DNS Handling

- **Primary DNS:** dns-prod-01 at 192.168.30.10 (AdGuard LXC on pve-prod-01).
    
- **Secondary DNS:** dns-prod-02 at 192.168.30.15 (AdGuard LXC on pve-prod-02, synced via adguardhome-sync).
    
- **DHCP server:** UDM-SE provides DNS settings to VLAN 20 and VLAN 30.
    
- **Management DNS:** 9.9.9.9 (public) — VLAN 10 should not depend on Proxmox-hosted services for DNS.
    
- **IoT DNS:** Requires verification.
    

### IoT DNS Documentation Discrepancy

The previous firewall documentation contained two conflicting descriptions:

1. IoT DNS traffic is intercepted by the UDM-SE and forwarded to AdGuard.
    
2. IoT uses public DNS at 9.9.9.9 without AdGuard.
    

Both cannot be treated as the definitive current configuration without checking the actual UniFi DNS and firewall settings.

Verify the intended IoT DNS configuration and update this section with the observed behavior.

---

## What Is Intentionally NOT Restricted

According to the established network design:

- **Services (30) → Internet:** Required for container image pulls, external API access, updates, Cloudflare Tunnel, and OpenAI Secure MCP Tunnel.
    
- **Management (10) → Documented internal VLANs:** Required for infrastructure administration and service communication.
    
- **Trusted (20) → Services (30):** General access is permitted for trusted users to reach internal services.
    

These allowances do not imply unrestricted access from VLAN 50 or override specific security controls applied to individual services.

The AI Observer remains limited by its dedicated Proxmox API permissions even though it has network connectivity to TCP 8006.

---

## Implementation Notes

- Firewall management is performed in the UniFi Dream Machine SE.
    
- Existing documentation places rules under **Settings → Routing → Firewall**. UI navigation may change between UniFi Network versions.
    
- Network groups documented historically: Management, Trusted, Services, IoT.
    
- Port group `MGMT_ADMIN_PORTS` historically lists TCP 443, 8006, and 22; Beszel documentation additionally mentions 8090.
    
- **Auto Allow Return Traffic** must be enabled where required for stateful return traffic.
    
- The AI Observer exception must be evaluated before the Services-to-Management deny policy.
    
- General Services-to-Management access must remain denied.
    
- Do not substitute an entire VLAN or subnet as the source when a targeted host-specific rule is sufficient.
    
- Changes to management access should be validated from an approved administrative device to avoid accidentally losing infrastructure access.
    

### AI Observer Verification

The following were successfully verified during the AI Observer deployment:

- TCP connectivity from observer-prod-01 to the Proxmox API.
    
- Authenticated Proxmox API access.
    
- Read-only MCP tool invocation.
    
- Outbound OpenAI Secure MCP Tunnel readiness.
    
- Successful Proxmox cluster query through the ChatGPT plugin.
    

These checks confirm the application path works. They are not a complete UniFi firewall policy audit.

### Pending Firewall Verification

- Confirm the current AI Observer rule name, priority, source, destination, and port in UniFi.
    
- Confirm the Beszel exception's direction and permitted ports.
    
- Reconcile the Trusted-to-Management port group with Beszel's documented TCP 8090 access.
    
- Verify VLAN 50 subnet, routing, isolation, and any game-server port forwards.
    
- Resolve the contradictory IoT DNS documentation.
    
- Recheck general Services-to-Management restrictions using safe connectivity tests.
    
- Review any stale rules left over from the v2 migration.
    

---

## Traffic Policy Summary

This table summarizes the established VLAN 10–40 design. It does not represent verified VLAN 50 policy.

|Source → Destination|Management|Trusted|Services|IoT|Internet|
|---|---|---|---|---|---|
|**Management (10)**|—|Allow|Allow|Allow|Allow|
|**Trusted (20)**|Admin ports only|—|Allow|Deny|Allow|
|**Services (30)**|Deny except targeted exceptions|Deny|—|Deny|Allow|
|**IoT (40)**|Deny|Deny|Deny|—|Allow|

### Targeted Services-to-Management Exceptions

|Source|Destination|Port|Purpose|Evidence|
|---|---|---|---|---|
|192.168.30.17|192.168.10.11|TCP 8006|AI Observer Proxmox MCP|Successful authenticated end-to-end connection|
|Services (30)|192.168.10.20|TCP 45876|Beszel monitoring|Recorded in Beszel service documentation; current UniFi rule requires verification|

These exceptions do not override the general deny policy for other Services-to-Management traffic.

---

## Related Documentation

- `architecture/network/ip-addresses.md` — VLAN subnets and assigned infrastructure IP addresses.
    
- `architecture/network/vlan-design.md` — Network segmentation design; requires update for VLAN 50.
    
- `architecture/compute/pve-prod-01.md` — Proxmox host and guest inventory.
    
- `services/infra/ai-observer.md` — AI Observer architecture, authentication, and security boundaries.
    
- `services/monitoring/beszel.md` — Beszel monitoring and documented firewall requirements.
    
- `runbooks/ai-observer-operations.md` — AI Observer operations and troubleshooting
    

---

## Documentation Status

**Verified:** AI Observer can reach the Proxmox API and retrieve live cluster data through ChatGPT.

**Documented but not independently reverified:** Existing general inter-VLAN policy and Beszel firewall rules.

**Unresolved:** VLAN 50 firewall policy, Beszel port-group discrepancy, and conflicting IoT DNS descriptions.

Do not mark these outstanding items as verified until the corresponding UniFi configuration or connectivity evidence has been inspected.