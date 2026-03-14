# Inter-VLAN Firewall Rules

**Enforced at:** UniFi Dream Machine SE (UDM-SE)  
**Default policy:** DENY ALL inter-VLAN traffic. Explicit ALLOW rules only.  
**Last updated:** 2026-03-13

---

## VLAN Reference

| VLAN | Name       | Subnet          | Key Members                                                                                        |
| ---- | ---------- | --------------- | -------------------------------------------------------------------------------------------------- |
| 10   | Management | 192.168.10.0/24 | UDM-SE, Switch, pve-prod-01, pve-prod-02, pi-prod-01, nas-prod-01 mgmt interface (onboard 2.5GbE) |
| 20   | Trusted    | 192.168.20.0/24 | Personal laptops, phones, trusted client devices                                                   |
| 30   | Services   | 192.168.30.0/24 | All VMs, LXCs, nas-prod-01 data/NFS interface (X710 Port 2)                                        |
| 40   | IoT        | 192.168.40.0/24 | Smart home devices, printers, untrusted endpoints                                                  |

> **NAS dual-interface note:** nas-prod-01 has two network presences. The **onboard 2.5GbE** (192.168.10.10) is on VLAN 10 — Unraid management UI only. The **X710 Port 2 SFP+** (192.168.30.16) is on VLAN 30 — NFS exports, Plex container traffic, all service-facing data. Services never cross into VLAN 10 to reach the NAS.

---

## Firewall Rules Table

| #  | Name                        | Source           | Destination              | Port / Protocol   | Action | Reason                                                                           |
| -- | --------------------------- | ---------------- | ------------------------ | ----------------- | ------ | -------------------------------------------------------------------------------- |
| 1  | Allow Management to All     | Management (10)  | Trusted + Services + IoT | Any               | ALLOW  | Management must reach everything it manages. Auto Allow Return Traffic enabled.  |
| 2  | Allow Trusted to Services   | Trusted (20)     | Services (30)            | Any               | ALLOW  | Users reach all internal services                                                |
| 3  | Allow Trusted to Mgmt       | Trusted (20)     | Management (10)          | TCP 443, 8006, 22 | ALLOW  | Admin access to Proxmox UI, Unraid UI, SSH                                       |
| 4  | Block Trusted to Mgmt Other | Trusted (20)     | Management (10)          | Any other         | DENY   | Block everything else into management from trusted                               |
| 5  | Block Services to Mgmt      | Services (30)    | Management (10)          | Any               | DENY   | Services cannot touch infrastructure management interfaces — including Unraid UI |
| 6  | Block Services to Trusted   | Services (30)    | Trusted (20)             | Any               | DENY   | Services never initiate connections to user devices                              |
| 7  | Block Services to IoT       | Services (30)    | IoT (40)                 | Any               | DENY   | Services have no business reaching IoT devices                                   |
| 8  | Block IoT to All Internal   | IoT (40)         | Any internal             | Any               | DENY   | IoT fully isolated from all internal VLANs                                       |

> **Rule evaluation order:** Rules are evaluated top to bottom. The first match wins. Rule 1 (Allow Management to All) must be at the top — it explicitly permits outbound management traffic before any block rules can match. Rules 3 and 4 together implement limited access from Trusted to Management. Auto Allow Return Traffic must be enabled on Rule 1 or return packets to VLAN 10 hosts will be dropped.

> **Important:** Without Auto Allow Return Traffic enabled on Rule 1, pve-prod-01 and pve-prod-02 can send packets to VLAN 30 but receive no responses. This was discovered during Phase 3 when pve-prod-01 could not reach pbs-prod-01 or other VLAN 30 VMs despite the allow rule being present.

---

## Per-VLAN Summary

### VLAN 10 — Management

- **Outbound:** Can reach all VLANs and internet — management needs to reach everything it manages
- **Inbound allowed from:** Trusted (20) on TCP 443, 8006, 22 only
- **Inbound denied from:** Services (30), IoT (40)
- **Hosts:** UDM-SE (10.1), Switch (10.2), pve-prod-01 (10.11), pve-prod-02 (10.12), nas-prod-01 mgmt (10.10), pi-prod-01 (10.20)

### VLAN 20 — Trusted

- **Outbound allowed to:** Services (30) — any port; Management (10) — TCP 443, 8006, 22 only; Internet — any
- **Outbound denied to:** Any other Management port
- **Hosts:** Personal laptops, phones, trusted client devices

### VLAN 30 — Services

- **Outbound allowed to:** Services (30) — any (inter-service); Internet — any (container pulls, CF tunnel, updates)
- **Outbound denied to:** Management (10) — hard block; Trusted (20) — services never initiate to clients; IoT (40)
- **Hosts:** All VMs and LXCs (192.168.30.10–.15+), nas-prod-01 NFS interface (30.16)

### VLAN 40 — IoT

- **Outbound allowed to:** Internet only
- **Outbound denied to:** Management (10), Services (30), Trusted (20) — fully isolated
- **DNS:** UDM-SE intercepts port 53 from VLAN 40 and forwards to AdGuard — IoT devices get ad-blocking without a direct path to VLAN 30
- **Hosts:** Smart home devices, printers, untrusted endpoints (DHCP 192.168.40.100+)

---

## NAS Security Model

The NAS holds backups, photos, and all irreplaceable data. Two-interface design intentionally prevents service-layer compromise from reaching the management UI:
```
docker-prod-01 (VLAN 30)
  → NFS mount → nas-prod-01 X710 Port 2 (192.168.30.16) ✅ ALLOWED (same VLAN)
  → Unraid UI → nas-prod-01 onboard 2.5GbE (192.168.10.10) ❌ BLOCKED (Rule 5: Services → Mgmt = DENY)

Laptop (VLAN 20)
  → Unraid UI → nas-prod-01 onboard 2.5GbE (192.168.10.10) ✅ ALLOWED (Rule 3: Trusted → Mgmt on 443)
```

A compromised container on docker-prod-01 can reach the NFS share it already has mounted — it cannot reach the Unraid web UI to delete arrays, modify shares, or cause further destruction.

---

## DNS Handling

- **Primary DNS:** dns-prod-01 at 192.168.30.10 (AdGuard LXC on pve-prod-01)
- **Secondary DNS:** dns-prod-02 at 192.168.30.15 (AdGuard LXC on pve-prod-02, synced via adguardhome-sync)
- **DHCP server:** UDM-SE pushes 192.168.30.10 as DNS to VLAN 20 and VLAN 30
- **Management DNS:** 9.9.9.9 (public) — VLAN 10 must not depend on Proxmox VMs for DNS
- **IoT DNS:** 9.9.9.9 (public) — IoT fully isolated, no AdGuard needed

---

## What Is Intentionally NOT Restricted

- Services (30) → Internet: Containers need to pull images, reach Cloudflare Tunnel, check updates
- Management (10) → all VLANs: Proxmox and Unraid need to reach everything they manage
- Trusted (20) → Services (30): Unrestricted — users should be able to reach all their services freely

---

## Implementation Notes

- All rules configured in UDM-SE under **Settings → Routing → Firewall**
- Network groups used: Management, Trusted, Services, IoT
- Port group: `MGMT_ADMIN_PORTS` = TCP 443, 8006, 22
- Rule 1 (Allow Management to All) must have **Auto Allow Return Traffic** enabled
- Verify IoT isolation with a test device: confirm it cannot ping 192.168.10.1, 192.168.20.x, or 192.168.30.x
- Verify Rule 5 by attempting to curl the Unraid UI from docker-prod-01 — connection should time out

---

## Traffic Policy Summary

| Source → Destination | Management       | Trusted | Services | IoT     | Internet |
| -------------------- | ---------------- | ------- | -------- | ------- | -------- |
| **Management (10)**  | —                | ✅ Allow | ✅ Allow  | ✅ Allow | ✅ Allow  |
| **Trusted (20)**     | ⚠️ Admin Ports Only | —    | ✅ Allow  | ❌ Deny  | ✅ Allow  |
| **Services (30)**    | ❌ Deny          | ❌ Deny  | —        | ❌ Deny  | ✅ Allow  |
| **IoT (40)**         | ❌ Deny          | ❌ Deny  | ❌ Deny   | —       | ✅ Allow  |

---

## Pending Rule (add during Beszel setup in Phase 5)

- Source: VLAN 30 (Services)
- Destination: 192.168.10.20 (pi-prod-01) specifically — not all of VLAN 10
- Port: TCP 45876 (Beszel agent default)
- Action: ALLOW
- Reason: Beszel agents on VLAN 30 hosts initiate connections to Beszel server on pi-prod-01 (VLAN 10). Rule 5 blocks this by default. Targeted rule keeps blast radius small — does not open broad VLAN 30 → VLAN 10 access.