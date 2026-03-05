#### VLAN 10 — Management (192.168.10.0/24)
#### VLAN 20 — Trusted (192.168.20.0/24)

Unchanged from v2. Personal laptops, phones, trusted devices. v2 services (Proxmox at .2, docker host at .10) remain here during the v3 build and migration. Decommissioned v2 IPs freed up once Phase 4 cutover is complete.
#### VLAN 30 — Services (192.168.30.0/24)

All v3 VMs and LXCs. Containers inside VMs are accessed via Traefik on the VM's IP — they do not get individual VLAN IPs.
#### VLAN 40 — IoT (192.168.40.0/24)

7 existing IoT devices migrate from 192.168.30.0 during Phase 1. Internet access only. No inter-VLAN routing. Gateway at 192.168.40.1. DHCP pool 192.168.40.100+.


