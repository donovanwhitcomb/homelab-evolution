# Proxmox-Based Homelab Infrastructure

This repository documents my personally built Proxmox-based homelab, designed to simulate a small production environment while developing hands-on experience in virtualization, storage, networking, Linux administration, and secure remote access.

This lab is treated as a real infrastructure project — not just experimentation. All major decisions, configurations, and lessons learned are documented here as part of an ongoing evolution log.

---

## Hardware Platform

**Compute**
- Intel Core i5-13500
- 64GB DDR5 RAM (2 × 32GB)

**Storage**
- 1TB NVMe (Primary Proxmox OS drive)
- 2TB NVMe (VM / container storage and lab workloads)
- 2 × 12TB WD Red Plus HDDs configured as a ZFS mirror

**Power Protection**
- APC BR1500MS2 UPS (1500VA / 900W, pure sine wave)

This hardware provides significant headroom for virtualization, storage experimentation, and multi-service deployment.

---

## Virtualization & Core Stack

- **Hypervisor:** Proxmox VE
- **Storage:** ZFS mirror on 2 × 12TB drives 
  - End-to-end checksumming
  - Snapshot capability 
  - Data integrity protection 
- **Remote Access:** Tailscale mesh VPN + MagicDNS 
  - Secure administrative access without exposing public ports 
- **Administration:** Linux CLI + SSH workflow 
- **Version Control:** Git + GitHub documentation

---

## Implemented Architecture

### ZFS Storage
- Configured a ZFS mirror on the 12TB drives.
- Structured datasets for organized storage management.
- Leveraging snapshot capabilities for rollback and data protection.

### Secure Remote Access
- Installed and configured Tailscale on:
  - Proxmox host
  - Laptop
  - Mobile device
- Verified secure remote access to the Proxmox web UI and shell.
- Avoided raw port-forwarding in favor of encrypted mesh VPN access.

### Virtualization & Containers
- Deploying services using Proxmox VMs and LXC containers.
- Experimented with containerized game server hosting (Minecraft).
- Performed troubleshooting on service behavior and resource handling.

---

## Troubleshooting & Operational Experience

- Diagnosed and resolved a Proxmox web UI 401 authentication issue by:
  - SSH access to host
  - Inspecting service status
  - Restarting `pveproxy` and `pvedaemon`
  - Verifying PAM authentication
- Practiced safe shutdown procedures:
  - Graceful VM/container shutdown
  - Controlled host shutdown
  - UPS management considerations
- Worked through filesystem navigation, Git workflow, and service configuration using Linux CLI.

---

## Design Principles

- Prefer secure VPN-based access over exposed services.
- Build with hardware headroom to avoid early bottlenecks.
- Treat homelab as production-style infrastructure.
- Document changes and architectural decisions for clarity and auditability.
- Continuously refine based on troubleshooting and lessons learned.

---

## Skills Demonstrated

- Linux filesystem navigation and CLI proficiency
- Virtualization management (Proxmox)
- ZFS storage configuration and planning
- Secure remote access implementation (Tailscale)
- Git version control and structured documentation
- Infrastructure troubleshooting and service debugging
- Resource planning and system design thinking

---

## Ongoing Roadmap

- Self-hosted password manager (Vaultwarden)
- Monitoring stack (Prometheus + Grafana)
- VLAN segmentation and network isolation
- Automated and off-site backup strategy
- Continued service deployment and performance tuning
