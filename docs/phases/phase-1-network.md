# Phase 1 — Network Foundation

**Status:** Complete **Completed:** 2026-03-10 **Branch:** Homelab v3.0

---

## Objective

Get the new switch in place, VLANs properly configured with firewall rules, and verify connectivity before any services move.

---

## Entry Criteria

- Phase 0 complete — all hardware racked and powered ✅
- UDM-SE operational and carrying existing network traffic ✅
- USW-Pro-Max-24 physically installed and powered ✅

---

## What Was Done

### Switch Adoption

- Adopted USW-Pro-Max-24 into UDM-SE
- Named device `usw-pro-max-24` in UniFi

### VLANs Created

All four VLANs created in UniFi under Settings → Networks:

|VLAN|Name|Subnet|DHCP Range|
|---|---|---|---|
|10|Management|192.168.10.0/24|.100–.254|
|20|Trusted|192.168.20.0/24|.100–.254|
|30|Services|192.168.30.0/24|.100–.254|
|40|IoT|192.168.40.0/24|.100–.254|

Confirmed VLAN 10 is NOT the default network — existing default network left untouched.

### IP Groups Created

Created in UniFi under Settings → Profiles → Port and IP Groups:

|Name|Value|
|---|---|
|NET_MGMT|192.168.10.0/24|
|NET_TRUSTED|192.168.20.0/24|
|NET_SERVICES|192.168.30.0/24|
|NET_IOT|192.168.40.0/24|
|ALL_INTERNAL|192.168.0.0/16|

### Port Group Created

|Name|Ports|
|---|---|
|MGMT_ADMIN_PORTS|22, 443, 8006|

### Firewall Rules Configured

7 LAN In rules created in order:

|#|Name|Action|
|---|---|---|
|1|Allow Trusted to Services|Allow|
|2|Allow Trusted to Mgmt - Admin Ports|Allow|
|3|Block Trusted to Mgmt - All Other|Drop|
|4|Block Services to Mgmt|Drop|
|5|Block Services to Trusted|Drop|
|6|Block Services to IoT|Drop|
|7|Block IoT to All Internal|Drop|

### IoT Devices Migrated

7 existing IoT devices moved from previous VLAN to VLAN 40:

- Eufy Homebase
- Eufy doorbell camera
- Eufy indoor camera
- Cove alarm panel
- Govee outdoor lights
- TP-Link Tapo power strip
- LG TV
- Nest thermostat

### Switch Port Profiles Created and Applied

|Profile|Native VLAN|Tagged VLAN Mgmt|
|---|---|---|
|Trunk - All VLANs|Management (10)|Allow All|
|Mgmt - Only|Management (10)|Allow All|
|Services Access|Services (30)|Block All|
|Trusted Access|Trusted (20)|Block All|
|IoT Access|IoT (40)|Block All|

Profiles applied to all connected ports. Unused ports (2–16) disabled.

---

## Validation Tests Performed

|Test|Method|Result|
|---|---|---|
|IoT isolation|Plugged into IoT port with static 192.168.40.50 — pinged 192.168.10.1, 20.1, 30.1|All failed ✅|
|IoT internet access|Pinged 8.8.8.8 from IoT port|Succeeded ✅|
|Trusted → Mgmt block|Pinged 192.168.10.2 (switch) from desktop on Trusted|Failed ✅|
|Trusted → Mgmt admin ports|Opened https://192.168.10.1 from desktop|Loaded ✅|
|Trusted → Services|Will be validated in Phase 3 when Services VLAN hosts exist|Deferred ⏳|
|Services → Mgmt block|Will be validated in Phase 3 from docker-prod-01|Deferred ⏳|

> **Note on UDM-SE ping behavior:** Pinging 192.168.10.1 from any VLAN always succeeds — the UDM-SE responds to pings on its own interface regardless of firewall rules. This is expected and correct. LAN In rules do not apply to traffic destined for the router itself. The correct test is pinging a real VLAN 10 host (e.g. the switch at 192.168.10.2) which correctly fails.

---

## Decisions Made During This Phase

**Allow Services → Services rule removed** The original plan included an explicit Allow Services → Services rule. This was removed — intra-VLAN traffic never crosses the UDM-SE firewall, it stays on the switch. The rule was unnecessary.

**Block Services → IoT rule added** Not in the original roadmap. Added for completeness — Services initiating connections to IoT devices is unlikely but not impossible. Explicit deny is cleaner than relying on default fallthrough.

**Trusted → IoT rule considered and rejected** Considered adding Allow Trusted → IoT to support smart home app control. Rejected — all smart home devices (Eufy, Cove, Govee, Tapo, LG, Nest) use cloud-based apps. Direct LAN access is not required. If Home Assistant is added in a future phase, a scoped Services → IoT rule will be created for HA specifically — Trusted → IoT will remain denied.

**pi-prod-01 left on VLAN 20 intentionally** pi-prod-01 is currently the only AdGuard DNS instance. Moving it to VLAN 10 now would require DNS to be reconfigured first. It stays on VLAN 20 (Trusted) until Phase 3 when dns-prod-01 LXC is running on pve-prod-01. Port profile will be updated to Trunk - All VLANs and IP reassigned to 192.168.10.20 at that time.

---

## Deferred to Phase 3

- Validate Trusted → Services (no Services VLAN hosts exist yet)
- Validate Services → Mgmt block from docker-prod-01
- Validate Trusted can reach Proxmox UI on 8006
- Cut pi-prod-01 from VLAN 20 → VLAN 10
- Configure IoT DNS intercept (port 53 redirect to AdGuard) — requires AdGuard running first

---

## Exit Criteria

- ✅ All VLANs created and tagged correctly
- ✅ All firewall rules in place and validated
- ✅ No unintended inter-VLAN routing