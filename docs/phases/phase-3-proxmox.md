# Phase 3 — Proxmox Rebuild & Core VM Infrastructure

**Status:** In Progress  
**Started:** 2026-03-13  
**Last Updated:** 2026-03-13

---

## Objectives

1. Fresh Proxmox install on MS-A2 (pve-prod-01) — ZFS RAID-1 boot mirror on 2x NVMe
2. Fresh Proxmox install on Optiplex (pve-prod-02)
3. Cluster both nodes, add pi-prod-01 as QDevice
4. Create AdGuard LXC (dns-prod-01) on VLAN 30 — 192.168.30.10
5. Create docker-prod-01 VM on VLAN 30 — 192.168.30.11
6. Create pbs-prod-01 VM on pve-prod-02 — 192.168.30.12
7. Install NUT clients on both Proxmox nodes

---

## Entry Criteria

- Phase 2 complete — nas-prod-01 healthy, all pools online, NFS exports verified
- All hardware racked and powered
- Network foundation complete — VLANs 10/20/30/40 configured, firewall rules validated

---

## Build Log

### pve-prod-01 — Minisforum MS-A2

- **Proxmox version:** 9.1.6
- **Kernel:** 6.17.13-1-pve
- **Boot storage:** ZFS RAID-1 mirror — Samsung 980 1TB + Sabrent Rocket 1TB (pool: rpool)
- **Management IP:** 192.168.10.11 (VLAN 10)
- **NIC used for management:** igc (Intel I226-V 2.5GbE) — connected to USW-Pro-Max-24 port 17-24 (2.5GbE ports)
- **vmbr0:** VLAN-aware bridge on igc — carries all VM/LXC traffic tagged per guest + management untagged
- Enterprise repos removed (pve-enterprise.sources, ceph.sources) — replaced with pve-no-subscription repo (trixie)
- System updated via apt dist-upgrade — 79 packages upgraded
- Subscription nag popup removed

#### Issues Encountered
- PVE 9 uses `.sources` format instead of `.list` files for apt repos — rm commands targeting `.list` files failed. Correct files were `pve-enterprise.sources` and `ceph.sources`
- MS-A2 was initially plugged into a 1GbE port on the switch (ports 1-16) instead of a 2.5GbE port (ports 17-24) — moved to correct port, confirmed 2500Mb/s via ethtool

---

### pve-prod-02 — Dell Optiplex 3070 Micro

- **Proxmox version:** 9.1.6
- **Boot storage:** Single NVMe — ext4 (single drive, no redundancy — acceptable for secondary node)
- **Management IP:** 192.168.10.12 (VLAN 10)
- **NIC:** 1GbE RJ45 — connected to USW-Pro-Max-24 (any port, 1GbE max)
- **vmbr0:** VLAN-aware bridge — same config as pve-prod-01
- Enterprise repos removed, pve-no-subscription repo added (trixie)
- System updated, subscription nag removed

#### Issues Encountered
- Installer showed "no supported hard disk found" — resolved by changing SATA controller mode from RAID/RST to AHCI in BIOS (Dell F2 → System Configuration → SATA Operation)

---

### Proxmox Cluster — homelab-v3

- **Cluster name:** homelab-v3
- **Transport:** knet
- **Secure auth:** on
- **Nodes:** pve-prod-01 (primary), pve-prod-02 (secondary)
- **HA:** NOT enabled — clustering for unified management UI only. Optiplex cannot handle MS-A2 workload.
- Cluster created on pve-prod-01 first, pve-prod-02 joined via join information token

---

### pi-prod-01 — QDevice Setup

- **Previous state:** VLAN 20, IP 192.168.20.8, running AdGuard Home (v2 role)
- **New state:** VLAN 10, IP 192.168.10.20
- **Network manager:** NetworkManager (not dhcpcd — Raspberry Pi OS Bookworm default)
- Static IP set via nmcli — dhcpcd.conf edits do nothing on this OS version
- Switch port changed to Mgmt-Only profile (access port, VLAN 10) on USW-Pro-Max-24
- `corosync-qdevice` installed on Pi, `corosync-qnetd` installed on Pi
- `corosync-qdevice` installed on both Proxmox nodes
- QDevice initialized via `pvecm qdevice setup 192.168.10.20` on pve-prod-01
- Root SSH enabled on Pi to allow pvecm to authenticate during setup

#### QDevice Status (confirmed working)
```
Expected votes: 3
Total votes:    3
Quorate:        Yes
Flags:          Quorate Qdevice
```

#### Issues Encountered
- dhcpcd.conf edits had no effect — Pi OS Bookworm uses NetworkManager, not dhcpcd
- Root SSH login disabled by default on Pi — had to enable PermitRootLogin in sshd_config
- corosync-qnetd package missing from Pi (only qdevice installed) — caused certutil errors
- corosync-qdevice package missing from Proxmox nodes — caused certutil errors on node side

---

### dns-prod-01 — AdGuard LXC

- **CT ID:** 100
- **Host:** pve-prod-01
- **Template:** debian-12-standard
- **IP:** 192.168.30.10/24 (VLAN 30)
- **Resources:** 1 vCPU, 512MB RAM, 4GB disk (local-zfs)
- **AdGuard Home:** installed via official install script
- **Admin UI:** http://192.168.30.10:3000
- **DNS port:** 53
- SSH enabled (openssh-server installed manually)

#### AdGuard Configuration
- **Upstream DNS:** `https://dns.quad9.net/dns-query`, `https://doh.opendns.com/dns-query`
- **Fallback DNS:** `9.9.9.10`, `1.1.1.1`
- **Bootstrap DNS:** `9.9.9.10`, `1.1.1.1`
- **Upstream mode:** Parallel requests
- **DNSSEC:** Enabled
- **Blocklist:** AdGuard DNS filter (default)
- **Custom filtering rules:**
```
@@||api.pubnative.net^
@@||api.sprig.com^
@@||api2.branch.io^$important
```
- **DNS rewrites:** None configured — will be added in Phase 4 pointing at Traefik (192.168.30.11)

#### UDM-SE DNS Configuration (updated)
| VLAN | DNS Server | Notes |
|------|------------|-------|
| 10 — Management | 9.9.9.9 | Public DNS — management must not depend on Proxmox |
| 20 — Trusted | 192.168.30.10 | AdGuard on dns-prod-01 |
| 30 — Services | 192.168.30.10 | AdGuard on dns-prod-01 |
| 40 — IoT | 9.9.9.9 | Public DNS — IoT fully isolated, no need for AdGuard |

#### Issues Encountered
- curl not installed by default on debian-12 LXC — installed manually before running AdGuard install script
- Pi's AdGuard DNS rewrites not carried over — all pointed at 192.168.20.10 (v2 NPM). Will be re-added in Phase 4 pointing at 192.168.30.11 (Traefik)

---

## Pending Tasks

- [ ] Create docker-prod-01 VM (192.168.30.11) on pve-prod-01
- [ ] Create pbs-prod-01 VM (192.168.30.12) on pve-prod-02
- [ ] Create dns-prod-02 LXC (192.168.30.15) on pve-prod-02
- [ ] Install NUT clients on pve-prod-01 and pve-prod-02
- [ ] Disable root SSH on pi-prod-01 after QDevice is confirmed stable
- [ ] Remove old v2 firewall rules from UDM-SE (Nginx Proxy Manager port forward)

---

## Exit Criteria

- [ ] Proxmox healthy on both nodes
- [ ] Cluster quorate with QDevice
- [ ] dns-prod-01 serving DNS for VLAN 20 and VLAN 30
- [ ] docker-prod-01 VM running with NFS mounts verified
- [ ] pbs-prod-01 running and taking test backups
- [ ] NUT clients configured on both Proxmox nodes