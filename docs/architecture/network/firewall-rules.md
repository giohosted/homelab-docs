# Inter-VLAN Firewall Rules

**Enforced at:** UniFi Dream Machine SE (UDM-SE)  
**Firewall architecture:** UniFi zone-based firewall with user-defined allow/block policies and system-defined fallback policies  
**Last Updated:** 2026-11-14  
**Verification:** Live read-only UniFi Network API inspection, supplemented by Proxmox cluster and existing service documentation

---

## Overview

Homelab v3 uses network segmentation to separate infrastructure management, trusted clients, production services, IoT devices, and game servers.

The intended security model is:

- Management hosts can administer approved internal resources.
- Trusted clients can access internal applications and permitted management services.
- Production services cannot initiate arbitrary connections to infrastructure management interfaces.
- IoT devices cannot initiate connections to the principal Internal networks.
- Game servers reside in a separate DMZ firewall zone.
- Host-specific exceptions are preferred over broad inter-VLAN access.
- Stateful return traffic is permitted for explicitly authorized connections.

**Important distinction:** The previous documentation described the configuration as a global "DENY ALL inter-VLAN traffic" policy.

The November 14 live UniFi inspection confirmed that this was not literally accurate. UniFi has an enabled system-defined `Allow All Traffic` fallback for Internal-to-Internal traffic, while user-defined rules implement the specific restrictions.

The actual effective policy therefore depends on the enabled rules, their ordering, zone membership, and system-defined behavior.

Do not assume that a traffic path is blocked merely because no explicit allow rule is documented.

No firewall changes were performed as part of this documentation review.

---

## VLAN and Firewall Zone Reference

| Network | IPv4 Subnet | UniFi VLAN ID | Firewall Zone | Key Members |
|---|---|---|---|---|
| Management | 192.168.10.0/24 | **1 — default/untagged** | Internal | UDM-SE, switch, pve-prod-01, pve-prod-02, pve-prod-03, pi-prod-01, nas-prod-01 management |
| Trusted | 192.168.20.0/24 | 20 | Internal | Personal laptops, phones, trusted clients |
| Services | 192.168.30.0/24 | 30 | Internal | Production VMs/LXCs, AI Observer, NAS data interface |
| IoT | 192.168.40.0/24 | 40 | Internal | Smart home devices, printers, untrusted endpoints |
| Game Servers | 192.168.50.0/24 | 50 | Dmz | dragonwilds-server (VM 106) |

### Management VLAN Naming Discrepancy

The existing Homelab v3 documentation commonly calls the Management network **"VLAN 10"** because its addressing uses `192.168.10.0/24`.

However, the live UniFi Network API returned:

| Setting | Verified Value |
|---|---|
| Network name | Management |
| VLAN ID | **1** |
| Default network | Yes |
| Gateway | 192.168.10.1 |
| Prefix | /24 |
| Firewall zone | Internal |
| Enabled | Yes |

The current Management network is therefore UniFi's default network using VLAN ID 1, not a separately tagged VLAN ID 10, according to the gateway configuration.

This is a documentation terminology discrepancy, **not an instruction to change VLAN tagging**.

Before standardizing terminology across hardware and networking documents, verify that current switch-port profiles and Proxmox bridge management traffic match the untagged/default-network design.

The network's IP subnet remains `192.168.10.0/24` regardless of this naming discrepancy.

### Current Infrastructure Members

**Management network:**

| Host | IP | Role |
|---|---|---|
| UDM-SE | 192.168.10.1 | Router, firewall, gateway |
| USW-Pro-Max-24 | 192.168.10.2 | Core switch |
| nas-prod-01 | 192.168.10.10 | Unraid management |
| pve-prod-01 | 192.168.10.11 | Primary Proxmox node |
| pve-prod-02 | 192.168.10.12 | Replacement Dell OptiPlex 5000 SFF |
| pve-prod-03 | 192.168.10.13 | Lenovo ThinkCentre M90q Tiny |
| pi-prod-01 | 192.168.10.20 | Monitoring, historical QDevice host |

**Services network:**

| Guest/Service | IP | Current Host |
|---|---|---|
| dns-prod-01 | 192.168.30.10 | pve-prod-01 |
| docker-prod-01 | 192.168.30.11 | pve-prod-01 |
| auth-prod-01 | 192.168.30.13 | pve-prod-03 |
| immich-prod-01 | 192.168.30.14 | pve-prod-02 |
| dns-prod-02 | 192.168.30.15 | pve-prod-03 |
| nas-prod-01 data interface | 192.168.30.16 | Unraid physical host |
| observer-prod-01 | 192.168.30.17 | pve-prod-01 |
| Plex | 192.168.30.2 | Docker on nas-prod-01 |

The guest placement was verified against the three-node Proxmox cluster on November 14, 2026.

### NAS Dual-Interface Design

`nas-prod-01` has two network presences:

- **Onboard 2.5GbE — 192.168.10.10:** Unraid management interface on the Management network.
- **X710 Port 2 SFP+ — 192.168.30.16:** Services network data interface for NFS, Plex-related traffic, and Docker PBS.

Production services use the Services interface for NAS data access.

**Security limitation:** Docker PBS uses Host networking. Its TCP 8007 listener may also be reachable through `192.168.10.10`, depending on process binding and firewall enforcement. Interface-specific exposure has not been independently tested.

The separate X710 Port 1 connection to `pve-prod-01` uses the private `10.0.0.0/30` point-to-point storage network and is not part of the VLAN-based firewall design.

---

## Firewall Rules Table

The following table represents the **enabled user-defined Internal-to-Internal policies returned by the live UniFi Network API on November 14, 2026**.

It does not include the numerous system-defined and derived return-traffic policies.

| Order | Rule Name | Source | Destination | Port / Protocol | Action | Purpose |
|---|---|---|---|---|---|---|
| 1 | Allow Management to All | Management network | Trusted, Services, IoT | Any | ALLOW | Permit management-initiated internal communication |
| 2 | Allow Trusted to Services | Trusted network | Services network | Any | ALLOW | Trusted client access to applications |
| 3 | Allow Trusted to Mgmt - Admin Ports | Trusted network | Management network | `MGMT_ADMIN_PORTS` | ALLOW | Restricted administrative access |
| 4 | Allow Observer to Proxmox API | 192.168.30.17 | 192.168.10.11 | TCP 8006 | ALLOW | AI Observer Proxmox API access |
| 5 | Block Trusted to Mgmt - All Other | Trusted network | Management network | All other traffic | BLOCK | Restrict management access |
| 6 | Allow Services to Beszel | Services network | 192.168.10.20 | Port 45876 | ALLOW | Beszel monitoring connectivity |
| 7 | Block Services to Mgmt | Services network | Management network | Other traffic | BLOCK | Protect infrastructure management |
| 8 | Block Services to Trusted | Services network | Trusted network | Any | BLOCK | Prevent services initiating connections to trusted devices |
| 9 | Block Services to IoT | Services network | IoT network | Any | BLOCK | Prevent services initiating connections to IoT |
| 10 | Block IoT to All Internal | IoT network | `192.168.0.0/16` destination group | Any | BLOCK | Restrict IoT access to internal resources |

**Verification:** All ten user-defined Internal-to-Internal policies above were returned as enabled.

The source and destination network/address filters were also inspected.

The ordering is based on UniFi's returned policy indices. It is not a list of immutable firewall policy IDs.

### Rule Ordering

The applicable user-defined policy ordering is significant:

- `Allow Observer to Proxmox API` appears before `Block Services to Mgmt`.
- `Allow Services to Beszel` appears before `Block Services to Mgmt`.
- `Allow Trusted to Mgmt - Admin Ports` appears before `Block Trusted to Mgmt - All Other`.
- Broad Services-to-Management restrictions follow the targeted allow exceptions.
- Derived return-traffic policies are present for rules with return traffic enabled.

The API returned these policies as enabled.

This confirms their configured ordering but does not replace active packet-level testing of effective enforcement.

### System-Defined Fallback

UniFi also returned an enabled system-defined `Allow All Traffic` policy for Internal-to-Internal traffic.

Therefore:

- The explicit deny rules are essential to the current network segmentation.
- The documentation should not describe all unspecified Internal-to-Internal traffic as automatically denied.
- Missing directional restrictions may leave traffic permitted by the fallback.
- A policy review must account for system-defined rules, not only the ten user-created rules.

**Specific finding:** The reviewed policies did not contain an explicit Trusted-to-IoT block.

The previously documented "Trusted → IoT = Deny" assertion is therefore not established by the current policy list. This path may be allowed by the Internal-to-Internal fallback and requires a safe connectivity test.

No new deny rule was created during this review.

---

## Auto Allow Return Traffic

The following enabled policies had `allowReturnTraffic=true` in the live API response:

- Allow Management to All.
- Allow Trusted to Services.
- Allow Trusted to Mgmt - Admin Ports.
- Allow Observer to Proxmox API.
- Allow Services to Beszel.

UniFi also returned corresponding derived return-traffic policies.

This supports stateful replies to permitted connections without authorizing arbitrary new connections in the reverse direction.

### Historical Issue

During Phase 3, `pve-prod-01` could not reach PBS and other Services network guests despite an outbound allow rule being present.

Enabling Auto Allow Return Traffic for the management rule corrected the observed behavior.

This setting remains enabled as of November 14.

**Operational principle:** Established or related return traffic should be permitted, while unsolicited new connections should remain subject to the relevant directional restrictions.

---

## AI Observer — Proxmox API Exception

### Purpose

`observer-prod-01` hosts the Homelab v3 AI Observer.

It provides read-only infrastructure monitoring through the connected MCP integrations.

The observer VM resides on the Services network (`192.168.30.17`), while the Proxmox management API is accessed through the Management network.

The general Services-to-Management block would otherwise prevent access.

A narrowly scoped exception was implemented during the original deployment.

### Verified Rule Configuration

| Field | Value |
|---|---|
| Rule name | Allow Observer to Proxmox API |
| Enabled | Yes |
| Source | 192.168.30.17 |
| Source network | Services |
| Destination | 192.168.10.11 |
| Destination network | Management |
| Protocol | TCP |
| Destination port | 8006 |
| Action | ALLOW |
| Auto Allow Return Traffic | Enabled |
| Evaluation | Before general Services-to-Management block |
| Scope | One source IP, one destination IP, one service port |

**November 14 live verification:** The rule exists, is enabled, and references the expected source and destination address groups.

The exception grants the observer access to the API endpoint on `pve-prod-01`.

It does not explicitly authorize access to:

- The Management network as a whole.
- The Unraid administration interface.
- SSH on Proxmox.
- Additional management service ports.
- The direct management APIs of `pve-prod-02` or `pve-prod-03`.

The current Proxmox MCP uses the cluster API through node 01. A successful cluster-wide guest query does not require unrestricted direct firewall access to every Proxmox node.

### Verification History

During AI Observer deployment:

1. The observer initially could not reach the Proxmox API.
2. A targeted UniFi firewall exception was implemented.
3. An unauthenticated HTTPS connection returned HTTP 401, confirming network reachability.
4. An authenticated MCP request successfully queried Proxmox.
5. A later ChatGPT request through the OpenAI Secure MCP Tunnel successfully returned live cluster data.

**November 14:** The rule was independently inspected through the live UniFi integration and confirmed enabled with the expected source, destination, port, and position ahead of the broad deny rule.

The Proxmox integration also successfully returned live information from the three-node cluster.

These observations confirm the deployed application path works.

They do not independently establish that every other possible network path is blocked.

### Additional Security Controls

The firewall exception is one layer of the observer security model.

Additional documented controls include:

- Dedicated Proxmox account: `observer@pve`.
- Dedicated API token: `observer@pve!proxmox-mcp`.
- Proxmox `PVEAuditor` role.
- MCP write, delete, and command-execution capabilities disabled.
- Proxmox MCP listener bound to `127.0.0.1:8080`.
- Outbound OpenAI Secure MCP Tunnel.
- Protected credential files outside Git.
- No public reverse-proxy route for the MCP endpoint.

The observer also hosts separate read-only integrations for Unraid, Komodo, and UniFi.

Each integration has its own service and tunnel configuration. A network path being reachable does not imply the ChatGPT-facing MCP identity has administrative capabilities.

The UniFi upstream API credential's privileges should be distinguished from the narrower read-only operations exposed through the observer.

### Outbound Tunnel Traffic

| Source | Destination | Port | Purpose |
|---|---|---|---|
| observer-prod-01 | OpenAI tunnel control plane | TCP 443 | Outbound secure MCP tunnel |
| Tunnel client | 127.0.0.1:8080 | TCP 8080 | Local Proxmox MCP endpoint |
| Proxmox MCP | 192.168.10.11 | TCP 8006 | Proxmox API |
| Local health checks | 127.0.0.1:8081 | TCP 8081 | Proxmox tunnel readiness |

The localhost listeners are not exposed by the UniFi firewall.

The four MCP integration/tunnel pairs have additional local ports documented in `services/infra/ai-observer.md`.

---

## Beszel — Management Network Exception

The existing Beszel documentation identifies its monitoring hub as running on `pi-prod-01` (`192.168.10.20`) on the Management network.

A targeted firewall exception allows monitoring-related connections from the Services network.

### Verified UniFi Rule

| Field | Value |
|---|---|
| Rule name | Allow Services to Beszel |
| Enabled | Yes |
| Source | Services network |
| Destination | 192.168.10.20 |
| Destination address group | Pi-Prod-01 IP |
| Destination port | 45876 |
| Port group | `Bezsel Ports` (name as configured) |
| Action | ALLOW |
| Auto Allow Return Traffic | Enabled |
| Evaluation | Before Block Services to Mgmt |

The source and destination were confirmed through UniFi's network and traffic-matching-list data.

The port group contains 45876.

**Protocol observation:** The retrieved policy specifies the destination port but does not explicitly narrow the protocol to TCP in its `ipProtocolScope` filter.

The original Beszel documentation describes TCP 45876. The exact effective protocol restriction should be reviewed before stating that the UniFi rule is TCP-only.

No policy change was made.

### Beszel UI Access

The Beszel service documentation states that its UI is reachable on:

`http://192.168.10.20:8090`

The November 14 live UniFi inspection confirmed TCP/UDP port number 8090 is included in the `MGMT_ADMIN_PORTS` traffic-matching list used by the Trusted-to-Management administrative access rule.

This resolves the previous documentation discrepancy about whether port 8090 was present in that port group.

Actual application-level access from a Trusted client was not retested during this documentation review.

See `services/monitoring/beszel.md`.

---

## Management Administrative Port Group

The live `MGMT_ADMIN_PORTS` traffic-matching list contains:

| Port | Typical Purpose | Verification |
|---|---|---|
| 22 | SSH | Included in live group |
| 443 | HTTPS administration | Included in live group |
| 8006 | Proxmox web UI/API | Included in live group |
| 8090 | Beszel hub UI | Included in live group |
| 3001 | Previously added administrative/service port; exact current application use not established | Included in live group |

The older firewall rule description lists only ports 22, 443, and 8006.

**The live port group contains five entries**, so the description is stale.

The actual trusted-access policy references this group.

The API response does not explicitly narrow the rule to TCP. Do not assume port-group entries are TCP-only merely because the historical documentation labels them administrative TCP ports.

### Security Review

Port 3001 deserves a separate review to confirm:

- Which service requires it.
- Whether broad Trusted-to-Management access is necessary.
- Whether a narrower destination-specific rule would be preferable.
- Whether the service is still active.

Do not remove or alter the port without identifying the dependent service.

---

## Proxmox Host Address Group — Inventory Discrepancy

The live UniFi traffic-matching list named `Proxmox host` contains:

- `192.168.10.11` — pve-prod-01.
- `192.168.10.12` — pve-prod-02.

It does **not** contain:

- `192.168.10.13` — pve-prod-03.

The group was created before the three-node architecture was complete.

**November 14 finding:** The group is stale as an inventory of the current Proxmox hosts.

However, no returned firewall policy referenced this particular traffic-matching-list ID during the live policy inspection.

Therefore, the missing third IP is a documentation/configuration housekeeping issue, **not evidence of an active connectivity failure or a missing firewall allow rule**.

Before modifying the group:

1. Confirm whether other UniFi configuration objects reference it.
2. Determine whether its name is intended to mean all Proxmox hosts.
3. Review potential consequences of adding another management destination.
4. Update it only after confirming the intended scope.

No modification was performed.

---

## Per-Network Summary

### Management — 192.168.10.0/24

- **UniFi VLAN ID:** 1, default/untagged.
- **Firewall zone:** Internal.
- **Outbound to Trusted, Services, IoT:** Explicitly allowed.
- **Inbound from Trusted:** Approved management port group.
- **Inbound from Services:** Only specified exceptions within the user-defined policies.
- **Inbound from IoT:** Blocked by the IoT-to-Internal rule.
- **Principal hosts:** UDM-SE, switch, NAS management, all three Proxmox nodes, Raspberry Pi.

Management-to-DMZ traffic is governed by the separate Internal-to-Dmz zone policy.

The November 14 UniFi configuration contains an enabled system-defined Internal-to-Dmz `Allow All Traffic` policy.

### Trusted — VLAN 20

- **Firewall zone:** Internal.
- **Access to Services:** Explicitly allowed.
- **Access to Management:** Restricted to configured administrative ports.
- **Other Management ports:** Explicitly blocked.
- **Internet access:** Enabled.
- **Access to IoT:** No explicit Trusted-to-IoT deny was found; effective restriction requires verification.
- **Access to Game Servers:** Subject to the Internal-to-Dmz zone policy.

### Services — VLAN 30

- **Firewall zone:** Internal.
- **Internet access:** Enabled.
- **Access to Management:** General block with specific AI Observer and Beszel exceptions.
- **Access to Trusted:** Explicitly blocked.
- **Access to IoT:** Explicitly blocked.
- **Same-network communication:** Not generally blocked by the inter-VLAN policy.

The existing Services network includes the Docker host, identity provider, AI Observer, primary and secondary DNS services, Immich, and the NAS data interface.

**Specific exception:** The AI Observer can access `192.168.10.11:8006` through its dedicated rule.

That does not authorize general Services-to-Management communication.

### IoT — VLAN 40

- **Firewall zone:** Internal.
- **Internet access:** Enabled.
- **Access to Internal networks:** Explicit blocking rule targeting the configured `NET_INTERNAL` address list.
- **DHCP-provided DNS:** 9.9.9.9, confirmed through live network configuration.
- **Access to Game Servers DMZ:** Not separately validated through active traffic testing.

The IoT-to-Internal rule uses a destination group containing `192.168.0.0/16`, but its firewall zone scope is Internal-to-Internal.

That does not automatically establish a matching restriction from Internal-to-Dmz.

The current system-defined Internal-to-Dmz policy must be considered during future IoT isolation testing.

### Game Servers — VLAN 50

**Network:** Game Servers  
**Firewall zone:** Dmz  
**Subnet:** 192.168.50.0/24  
**Gateway:** 192.168.50.1  
**DHCP range:** 192.168.50.6–192.168.50.254  
**Internet access:** Enabled  
**Configured guest:** dragonwilds-server (VM 106)

These network details were verified through the live UniFi integration on November 14.

The Game Servers network is separate from the Internal firewall zone.

### Verified DMZ Zone Policy

The live firewall policy list contains system-defined rules including:

| Direction | System-Defined Policy | Observed Action |
|---|---|---|
| Internal → Dmz | Allow All Traffic | ALLOW |
| Dmz → Internal | Block All Traffic (with established/related return traffic exception) | BLOCK new traffic by default |
| Dmz → External | Allow All Traffic | ALLOW |
| External → Dmz | Block All Traffic (with return traffic exception) | BLOCK new traffic by default |

These are zone-level policies, not individual game-server port forwards.

**Important:** The Internal-to-Dmz default allowance is broader than a blanket assumption that game servers are unreachable from internal clients.

The guest's exact IP address, any specific port forwards, and real-world packet-level enforcement have not been independently verified.

Do not infer the guest IP from the DHCP range.

See `architecture/network/ip-addresses.md` and `architecture/network/vlan-design.md`.

---

## NAS Security Model

The NAS holds backups, photos, and other irreplaceable data.

Its dual-interface design intentionally separates management access from service-facing data traffic.

```text
docker-prod-01 (Services)
  → NFS mount
  → nas-prod-01 X710 Port 2 (192.168.30.16)
  → Same Services network

docker-prod-01 (Services)
  → Unraid management UI
  → nas-prod-01 onboard 2.5GbE (192.168.10.10)
  → Blocked by configured Services-to-Management rule

observer-prod-01 (Services)
  → Proxmox API (192.168.10.11:8006)
  → Allowed by targeted AI Observer exception

observer-prod-01 (Services)
  → Unraid management UI (192.168.10.10)
  → Not authorized by the AI Observer exception

Trusted laptop (Trusted)
  → Unraid management UI (192.168.10.10:443)
  → Allowed by documented administrative access policy
```

A compromised container on docker-prod-01 may reach NFS resources already accessible to that host, but the configured Services-to-Management rule restricts direct access to the Unraid management interface.

The AI Observer exception does not authorize access to the NAS management interface.

Actual enforcement should be periodically validated with safe connectivity tests.

### PBS Host-Network Exposure

PBS runs on Unraid using Docker Host networking.

Although its intended production endpoint is:

`192.168.30.16:8007`

the listener may also be reachable through:

`192.168.10.10:8007`

This question cannot be resolved solely by reviewing the inter-VLAN firewall rules.

Verify the actual service bind address and effective network reachability before recording the PBS management-plane exposure as restricted.

---

## DNS Handling

### Current DHCP DNS Configuration

The live UniFi network details returned the following DNS server overrides:

| Network | DNS Servers Supplied Through DHCP |
|---|---|
| Management | 9.9.9.9, 1.1.1.1 |
| Trusted | 192.168.30.10, 192.168.30.15 |
| Services | 192.168.30.10, 192.168.30.15 |
| IoT | 9.9.9.9 |
| Game Servers | No custom DNS override returned |

**Primary AdGuard:** `dns-prod-01`, 192.168.30.10, CT 100 on pve-prod-01.

**Secondary AdGuard:** `dns-prod-02`, 192.168.30.15, CT 102 on pve-prod-03.

Both DNS instances are on separate Proxmox hosts.

### IoT DNS Documentation Reconciliation

The previous firewall documentation contained conflicting descriptions:

1. IoT DNS traffic was intercepted and forwarded to AdGuard.
2. IoT used public DNS at 9.9.9.9 without AdGuard.

**November 14 finding:**

The live IoT DHCP configuration specifies `9.9.9.9`.

No custom DNS policy objects were returned by the available UniFi DNS policy listing.

This confirms the DHCP-advertised DNS server, but does **not** establish whether separate NAT, DNS interception, or traffic-routing behavior changes the actual destination used by clients.

Therefore:

- **Verified:** IoT DHCP supplies `9.9.9.9`.
- **Unverified:** Whether DNS interception redirects IoT traffic.
- **Unverified:** Whether individual IoT devices use manually configured DNS, encrypted DNS, or hardcoded resolvers.

Do not state that IoT traffic is forcibly intercepted by AdGuard without inspecting the relevant routing/NAT behavior or performing a controlled query test.

---

## Other Verified Firewall Policies

The primary table covers user-defined Internal-to-Internal policies.

The November 14 UniFi inspection also returned other enabled policies, including:

| Rule | Direction | Purpose |
|---|---|---|
| Allow External to Plex Server | External → Internal | TCP 32400 access to the Plex server address group |
| VPN TO INT ALL | VPN → Internal | Broad documented VPN access to internal addresses |

The Plex address group identifies:

`192.168.30.2`

The VPN policy references the `NET_INTERNAL` address group containing:

`192.168.0.0/16`

These rules are not part of the primary inter-VLAN table but remain relevant to the overall gateway security model.

The review also returned system-defined policies for established traffic, invalid traffic, gateway services, mDNS, DHCP, and inter-zone defaults.

This document does not attempt to reproduce all system-defined UniFi firewall objects or all port-forwarding configuration.

---

## What Is Intentionally NOT Restricted

According to the established network design and currently observed configuration:

- **Services → Internet:** Required for container image pulls, external API calls, software updates, Cloudflare Tunnel, and OpenAI Secure MCP Tunnels.
- **Management → Trusted, Services, IoT:** Allowed for infrastructure administration and service communication.
- **Trusted → Services:** Allowed for trusted users to access internal services.
- **Internal → Game Servers DMZ:** Allowed by the current system-defined zone policy.
- **Configured VPN → Internal:** Allowed by an enabled VPN firewall policy.

These allowances do not authorize new Services-to-Management connections outside the targeted exceptions.

They also do not imply that all cross-network behavior has been safely validated through packet-level tests.

The AI Observer remains limited by its dedicated Proxmox API permissions even though it has network access to TCP 8006.

---

## Implementation Notes

- Firewall management is performed in the UniFi Dream Machine SE.
- Existing documentation places rules under **Settings → Routing → Firewall**. Exact UI navigation depends on the installed UniFi Network version.
- The live configuration uses named firewall zones: Internal, Dmz, Gateway, External, Vpn, and Hotspot.
- Management, Trusted, Services, and IoT are assigned to Internal.
- Game Servers VLAN 50 is assigned to Dmz.
- Existing user-defined policies are ordered ahead of relevant broader deny rules.
- Return traffic is explicitly enabled for important allow policies.
- The Trusted administrative port group contains 22, 443, 8006, 8090, and 3001.
- General Services-to-Management access is blocked by an enabled user-defined policy.
- AI Observer access uses a dedicated host-specific source, destination, and TCP port.
- Beszel access uses the Services network, a Raspberry Pi address group, and port 45876.
- The `Proxmox host` address group still has only nodes 01 and 02.
- The Management network's actual UniFi VLAN ID is 1, despite historical references to "VLAN 10" in documentation.
- The default Internal-to-Internal system policy is allow, so explicitly defined deny policies are important.

### Change Safety

When changing firewall rules:

1. Record the existing configuration.
2. Identify affected clients, hosts, and services.
3. Review source/destination zones and network groups.
4. Confirm the correct rule evaluation order.
5. Consider whether return traffic is required.
6. Prefer narrow host and port exceptions.
7. Validate management access from an approved administrative device.
8. Apply only the approved change.
9. Test actual traffic paths.
10. Document the verified result.

Do not broaden Services-to-Management permissions solely because new Proxmox nodes have joined the cluster.

### AI Observer Verification

The following have been successfully verified:

- Connectivity from observer-prod-01 to the Proxmox API.
- Authenticated Proxmox API access.
- Read-only MCP tool invocation.
- Outbound OpenAI Secure MCP Tunnel operation.
- Successful Proxmox cluster queries through ChatGPT.
- Live UniFi inspection of the relevant observer firewall policy.

These checks confirm the application path works.

They do not amount to comprehensive firewall penetration testing or proof that all unintended paths are blocked.

---

## Traffic Policy Summary

The following table summarizes the observed inter-network policy model.

| Source → Destination | Management | Trusted | Services | IoT | Internet |
|---|---|---|---|---|---|
| **Management** | — | Allow | Allow | Allow | Allow |
| **Trusted** | Admin ports only | — | Allow | **No explicit block verified** | Allow |
| **Services** | Block except targeted exceptions | Block | — | Block | Allow |
| **IoT** | Block | Block | Block | — | Allow |
| **Game Servers (DMZ)** | Block new DMZ-initiated connections by default | Block new DMZ-initiated connections by default | Block new DMZ-initiated connections by default | Block new DMZ-initiated connections by default | Allow |

**Interpretation:**

- The table describes observed policies, not an exhaustive network reachability test.
- Game Servers uses a different firewall zone from the other networks.
- Internal-to-DMZ traffic is permitted by an enabled system-defined policy.
- Trusted-to-IoT lacks an explicit matching deny in the retrieved rules and should not be represented as confirmed blocked.
- Established/related return-traffic rules may permit responses in otherwise blocked directions.
- Other network, host, VPN, NAT, or application controls can further affect effective reachability.

### Targeted Services-to-Management Exceptions

| Source | Destination | Port | Purpose | Evidence |
|---|---|---|---|---|
| 192.168.30.17 | 192.168.10.11 | TCP 8006 | AI Observer Proxmox MCP | Enabled, live-verified UniFi rule; successful authenticated end-to-end connection |
| Services network | 192.168.10.20 | Port 45876 | Beszel monitoring | Enabled, live-verified UniFi rule; protocol-specific narrowing not confirmed |

These exceptions do not override the general deny policy for other Services-to-Management traffic.

---

## Verification Results — November 14, 2026

| Check | Result |
|---|---|
| UniFi firewall policy list | Retrieved; 92 total policies, including system-defined and derived policies |
| User-defined Internal-to-Internal policies | Ten enabled policies identified |
| Management, Trusted, Services, IoT zone membership | Verified as Internal |
| Game Servers zone | Verified as Dmz |
| Game Servers subnet and gateway | 192.168.50.0/24, 192.168.50.1 |
| Management network VLAN ID | 1 (default) |
| AI Observer rule | Enabled; correct source, destination, TCP 8006 |
| Beszel rule | Enabled; Services → 192.168.10.20, port 45876 |
| `MGMT_ADMIN_PORTS` group | Contains 22, 443, 8006, 8090, 3001 |
| `Proxmox host` address group | Contains node 01 and node 02; node 03 missing |
| Management return traffic | Enabled |
| Trusted-to-IoT explicit deny | Not found in reviewed policies |
| Internal-to-Internal fallback | System-defined Allow All Traffic enabled |
| DMZ-to-Internal default | Block new traffic, with return-traffic exception |
| Internal-to-DMZ default | Allow All Traffic enabled |
| IoT DHCP DNS | 9.9.9.9 |
| Trusted/Services DHCP DNS | 192.168.30.10, 192.168.30.15 |
| DNS interception behavior | Not established |
| Changes applied during review | None |

---

## Pending Firewall and Documentation Verification

### Configuration and Security

- Confirm actual Trusted-to-IoT connectivity and determine whether a new deny rule is needed to meet the intended isolation policy.
- Confirm whether the broad Internal-to-DMZ allowance is appropriate for the Game Servers network.
- Review TCP/UDP protocol scoping on the Trusted administrative and Beszel port-group rules.
- Identify the application requiring management port 3001 and determine whether it still needs broad access.
- Check whether the `Proxmox host` address group should include `192.168.10.13`; confirm references before changing it.
- Verify the actual IoT DNS interception behavior.
- Check the active host and application IP addressing of `dragonwilds-server`.
- Review current game-server port forwards separately from the zone firewall list.
- Validate Services-to-Management blocking with safe connectivity tests.
- Verify intended interface-level exposure of Docker PBS TCP 8007.
- Review stale policies and groups inherited from earlier Homelab versions.

### Documentation Consistency

The following related documents should reflect the live findings where appropriate:

- `architecture/network/ip-addresses.md` — Management VLAN ID discrepancy; verified Game Servers subnet and gateway.
- `architecture/network/vlan-design.md` — Management default-network VLAN ID, DMZ zone assignment, firewall fallback behavior, and verified DHCP/DNS design.
- `architecture/hardware/inventory.md` — Clarify Management VLAN terminology if the physical network review confirms the default/untagged configuration.
- `architecture/compute/pve-prod-01.md` — Clarify Management network VLAN terminology if needed.
- `architecture/compute/pve-prod-02.md` — Clarify Management network VLAN terminology if needed.
- `architecture/compute/pve-prod-03.md` — Clarify Management network VLAN terminology if needed.
- `services/monitoring/beszel.md` — Only update the firewall description if a protocol or scope change is subsequently confirmed.

These are documentation reconciliation items, not authorization to reconfigure the network.

---

## Related Documentation

- `architecture/network/ip-addresses.md` — Infrastructure IP addresses, guest placement, and network subnets.
- `architecture/network/vlan-design.md` — Network segmentation and VLAN/zone design.
- `architecture/compute/pve-prod-01.md` — Primary Proxmox host and AI Observer.
- `architecture/compute/pve-prod-02.md` — Replacement Dell Proxmox node and Immich.
- `architecture/compute/pve-prod-03.md` — Lenovo Proxmox node, secondary DNS, and Authentik.
- `architecture/hardware/inventory.md` — Current hardware and host inventory.
- `architecture/storage/nas-prod-01.md` — NAS interfaces and management/data separation.
- `services/infra/ai-observer.md` — AI Observer architecture, authentication, and security boundaries.
- `services/infra/authentik.md` — Identity provider and SSO.
- `services/infra/adguard.md` — Primary and secondary DNS.
- `services/monitoring/beszel.md` — Beszel monitoring and firewall requirements.
- `runbooks/ai-observer-operations.md` — AI Observer operations and troubleshooting.

---

## Documentation Status

**Live-verified November 14:**

- Current UniFi network and zone assignments.
- All ten user-defined Internal-to-Internal firewall policies.
- Rule order and enabled state.
- AI Observer and Beszel policy configuration.
- Administrative port-group membership.
- Proxmox host address-group membership.
- Game Servers DMZ subnet and gateway.
- DHCP DNS server overrides.
- Three-node Proxmox cluster architecture.

**Documented but requiring further verification:**

- Effective packet-level isolation for every network pair.
- Trusted-to-IoT access behavior.
- Game Servers application IP and port-forward configuration.
- IoT DNS interception and NAT behavior.
- Actual PBS host-interface exposure.
- Port 3001's service dependency.
- Physical switch-port tagging and Management VLAN terminology.

**Changes performed:** Documentation review only. No UniFi, Proxmox, DNS, firewall, or network configuration was modified.

The network should not be considered fully security-audited solely because its firewall policies were retrieved successfully.
