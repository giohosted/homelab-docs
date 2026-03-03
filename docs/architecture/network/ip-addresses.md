#### VLAN 10 — Management (192.168.10.0/24)

|Device|IP|Notes|
|---|---|---|
|UDM-SE|192.168.10.1|Gateway — already configured, do not change|
|Core Switch (UniFi)|192.168.10.2|Management interface|
|UPS (if networked later)|192.168.10.3|Reserved — NUT handles shutdown via USB for now|
|nas-prod-01 (Unraid)|192.168.10.10|Management/NFS interface|
|pve-prod-01 (MS-A2)|192.168.10.11|Proxmox management UI|
|pve-prod-02 (Optiplex)|192.168.10.12|Proxmox management UI|
|pi-prod-01 (Raspberry Pi)|192.168.10.20|QDevice + monitoring|
|192.168.10.30–.49|—|Reserved future expansion / IPMI / OOB|
|192.168.10.100+|—|DHCP pool if needed|

#### VLAN 20 — Trusted (192.168.20.0/24)

Unchanged from v2. Personal laptops, phones, trusted devices. v2 services (Proxmox at .2, docker host at .10) remain here during the v3 build and migration. Decommissioned v2 IPs freed up once Phase 4 cutover is complete.

#### VLAN 30 — Services (192.168.30.0/24)

All v3 VMs and LXCs. Containers inside VMs are accessed via Traefik on the VM's IP — they do not get individual VLAN IPs.

|Device|IP|Notes|
|---|---|---|
|dns-prod-01 (AdGuard LXC, pve-prod-01)|192.168.30.10|Primary DNS — AdGuard Home|
|docker-prod-01 (media/apps VM, pve-prod-01)|192.168.30.11|All media stack containers behind Traefik|
|pbs-prod-01 (PBS VM, pve-prod-02)|192.168.30.12|Proxmox Backup Server|
|auth-prod-01 (Authentik VM, pve-prod-01)|192.168.30.13|Authentik IdP — dedicated VM|
|immich-prod-01 (Immich VM, pve-prod-01)|192.168.30.14|Immich — isolated for resource tuning|
|dns-prod-02 (AdGuard LXC, pve-prod-02)|192.168.30.15|Secondary DNS — synced from dns-prod-01|
|192.168.30.100+|—|DHCP / sandbox / test / future k3s nodes|

#### VLAN 40 — IoT (192.168.40.0/24)

7 existing IoT devices migrate from 192.168.30.0 during Phase 1. Internet access only. No inter-VLAN routing. Gateway at 192.168.40.1. DHCP pool 192.168.40.100+.