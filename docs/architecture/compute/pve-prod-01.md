**Role:** Primary Proxmox node. Runs all primary VMs and LXCs (docker-prod-01, auth-prod-01, immich-prod-01, dns-prod-01).

| Component                | Detail                                                                     |
| ------------------------ | -------------------------------------------------------------------------- |
| **Model**                | Minisforum MS-A2                                                           |
| **CPU**                  | AMD Ryzen 9 7945HX (16C/32T, 5.4GHz boost)                                 |
| **iGPU**                 | AMD Radeon 680M (available for future ML/transcoding — not used at launch) |
| **RAM**                  | 32GB DDR5 SO-DIMM (1x 32GB. Will add 2nd stick in the future)              |
| **RAM Max**              | 96GB                                                                       |
| **Boot Drive 1**         | Samsung 980 NVMe 1TB S/N: S64ANS0W120169T                                  |
| **Boot Drive 2**         | Sabrent Rocket NVMe 1TB                                                    |
| **Boot Config**          | ZFS RAID-1 mirror — configured in Proxmox installer                        |
| **Networking (LAN)**     | 2x 2.5GbE RJ45 built-in — uplink to USW-Pro-Max-24 for VMs                 |
| **Networking (Storage)** | 2x 10GbE SFP+ built-in — Port 1: DAC to NAS on own storage subnet          |
| **OS**                   | Proxmox VE 9.1.5                                                           |
| **IP**                   | 192.168.10.11 (VLAN 10 — Management)                                       |
| **Purchased**            | 2026-02-21                                                                 |
| **Source**               | Amazon                                                                     |
| **Price Paid**           | $575                                                                       |
