# Homelab Evolution

A personally built, Proxmox-based homelab designed and operated as production-style infrastructure. This is not a tutorial follow-along — it’s a working environment that has gone through real failures, real debugging, and real operational decisions.

All architecture choices, configuration changes, troubleshooting incidents, and lessons learned are tracked in the [CHANGELOG](./CHANGELOG.md).

-----

## Hardware

|Component     |Spec                                                                    |
|--------------|------------------------------------------------------------------------|
|CPU           |Intel Core i5-13500                                                     |
|RAM           |64GB DDR5 (running at 4200MHz JEDEC; XMP disabled — see stability notes)|
|Boot Drive    |Crucial T500 1TB NVMe                                                   |
|Workload Drive|WD 2TB NVMe                                                             |
|NAS Drives    |2× WD Red Plus 12TB (ZFS mirror)                                        |
|PSU           |Corsair RM850x 850W                                                     |
|UPS           |APC BR1500MS2 (1500VA / 900W, pure sine wave)                           |
|Motherboard   |ASUS B760M-A AX                                                         |

-----

## Core Stack

|Layer           |Technology                               |
|----------------|-----------------------------------------|
|Hypervisor      |Proxmox VE 9.1.1                         |
|Storage         |ZFS mirror (`tank`), local-lvm, NVMe pool|
|Containerization|Docker (on dedicated VM)                 |
|Remote Access   |Tailscale mesh VPN + MagicDNS            |
|Monitoring      |Prometheus + Grafana + Node Exporter     |
|Shell / Admin   |Linux CLI, SSH, RCON                     |
|Version Control |Git + GitHub                             |

-----

## Infrastructure Layout

### Proxmox Host

- Hypervisor managing all VMs and previously LXC containers
- ZFS mirror pool (`tank`) on 12TB drives with ARC capped at 8GB to prevent RAM contention
- Storage pools: `local`, `local-lvm`, `nvme2tb`, `tank`
- GRUB tuned for stability: `nosoftlockup split_lock_detect=off nohz_full=off`
- Tailscale installed for secure out-of-band access without port forwarding

### docker-01 (VM 101)

Primary workload VM. Runs all Docker services.

**Active services:**

- **block2a** — Minecraft SMP server (NeoForge 1.21.1, 178-mod modpack, 12GB heap, Aikar JVM flags, 20.00 TPS sustained)
- **Gone Fishing** — Vintage Story 1.22 SMP server
- **Valheim** — Dedicated game server
- **Media stack** — Plex, Radarr, Sonarr, Lidarr, Prowlarr, qBittorrent, Bazarr, Overseerr, Tautulli (in progress)

All services defined as Docker Compose stacks. Server files and configs version-controlled where possible.

### media-01 (VM 102)

- Ubuntu 22.04.5 LTS
- Static LAN IP: 10.0.0.211
- NFS mount from Proxmox `tank` pool (`/mnt/media` → movies, tv, music)
- Intel iGPU passthrough planned for Plex hardware transcoding
- Docker installed, media stack being brought online

### Monitoring Stack

- **Prometheus** scraping three targets: itself, Node Exporter (host metrics), Proxmox
- **Grafana** with Node Exporter Full dashboard (ID 1860), job-based host switching
- Proxmox scrape target uses Tailscale IP to bypass vmbr0 bridge routing issue
- UFW rules scoped to `tailscale0` interface — monitoring ports not exposed publicly

### Network & Security

- Tailscale mesh VPN across all nodes (Proxmox host, docker-01, media-01, personal devices)
- MagicDNS for hostname-based internal routing
- UFW configured on all VMs; default deny inbound, tailscale0 interface rules for internal services
- No raw port forwarding for administrative services — everything behind VPN

-----

## Notable Incidents & Resolutions

These are not just “things I learned” — these are real failures that took real debugging to resolve.

**March 2026 — Repeated kernel lockups (root cause: XMP instability)**

- Symptoms: Proxmox soft lockups and then hard lockups under load
- Initial mitigations: added `nosoftlockup split_lock_detect=off` to GRUB (helped, didn’t fix)
- Root cause identified: DDR5 XMP at 5200MHz was unstable under sustained workloads
- Fix: Disabled XMP → dropped to 4200MHz JEDEC spec; added `nohz_full=off`, removed `mitigations=off`
- System has been stable since

**ZFS ARC memory contention**

- ARC was consuming excess RAM under VM workloads
- Capped ARC at 8GB via `/etc/modprobe.d/zfs.conf`

**Prometheus scrape failure (Proxmox target)**

- Proxmox node_exporter unreachable via LAN IP due to `vmbr0` bridge not self-routing
- Fix: changed scrape target from LAN IP to Tailscale IP — all three targets now UP

**Minecraft server performance tuning**

- Profiled with Spark; identified and removed mods causing DH LOD issues and background crashes (ServerCore) and mixin conflicts with Lithium (Better Chunk Loading)
- Final result: 20.00 TPS, 1.87ms median MSPT at launch

-----

## Skills Demonstrated

- Hypervisor administration (Proxmox VE)
- ZFS storage configuration, pool management, ARC tuning
- Docker and Docker Compose service deployment and management
- Linux system administration (Ubuntu, Debian-based)
- Network design: VLANs, VPN mesh, firewall rules (UFW)
- Monitoring stack deployment and configuration (Prometheus, Grafana)
- Kernel-level stability debugging (GRUB flags, CPU frequency scaling)
- NFS configuration and persistent mounts
- JVM performance tuning (heap sizing, GC flags)
- Infrastructure documentation and change management

-----

## Roadmap

- [ ] iGPU passthrough to media-01 for Plex hardware transcoding
- [ ] Bring full media stack online (Plex + arr suite)
- [ ] VLAN segmentation between lab, game servers, and media
- [ ] Automated off-site backup strategy (tank pool)
- [ ] Self-hosted password manager (Vaultwarden)
- [ ] Network monitoring expansion (interface stats, per-VM dashboards)