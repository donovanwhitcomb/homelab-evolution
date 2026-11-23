# Homelab Evolution Log

## 2026-11-22 — media-01 Rebuild, Storage Passthrough & First Media Apps

### Summary

Destroyed and rebuilt **media-01 (VM 102)** from scratch on Ubuntu Server 24.04 LTS, replaced the old NFS-based media mount with a **virtiofs** share from the `tank` ZFS pool, and deployed the first apps: **Audiobookshelf**, **Cockpit**, and **Portainer**. Also confirmed the host is ready for Intel iGPU passthrough (Plex is the next phase).

Management approach for this VM: headless Ubuntu Server, managed through web GUIs (Cockpit for the OS, Portainer for Docker). No desktop environment, to keep overhead low.

---

### Incident — Lost Password on the Old media-01

* The login password for the old media-01 had been lost, so the VM couldn't be reached over SSH.
* Tried resetting it through Ubuntu recovery mode from the Proxmox console:
  * Ubuntu hides the GRUB menu by default. Pressing `Esc` too many times dropped to the `grub>` command line, and `normal` just carried on booting.
  * While the guest was sitting at GRUB, Proxmox **Reboot/Shutdown** tasks failed (`can't lock file`, `VM quit/powerdown failed`) because nothing in the guest was answering ACPI requests. A hard **Stop** was needed.
  * Got to the firmware boot menu (OVMF) and then the GRUB menu, but the console display corrupted when booting the recovery kernel.
* No media stack had been deployed on the old VM yet, so a clean rebuild was faster and lower risk than continuing the recovery.

**Resolution**

* Removed VM 102 in Proxmox with **Purge from job configurations** and **Destroy unreferenced disks owned by guest** both checked (task log: `VM 102 - Destroy: OK`).
* Rebuilt it as a new VM 102, `media-01`.

**Lesson:** store every credential in a password manager when it's created.

---

### Added — New media-01 VM (VM 102)

**Proxmox VM configuration** (from the Create VM confirmation screen):

| Setting | Value |
|---|---|
| Machine | q35 |
| BIOS | OVMF (UEFI), EFI disk on `local-lvm`, `pre-enrolled-keys=1` (Secure Boot keys) |
| CPU | 1 socket × 4 cores, type `x86-64-v3` (hides the i5-13500's hybrid P/E-core layout from the guest) |
| Memory | 8192 MB |
| Disk | `scsi0` — 64 GB on `local-lvm`, `discard=on`, `ssd=on`, `iothread=on` |
| SCSI controller | VirtIO SCSI single |
| Network | VirtIO on `vmbr0`, Proxmox firewall on |
| QEMU guest agent | Enabled |
| HA / TPM | Not used (single node; not needed for Linux) |

**OS install**

* Ubuntu Server **24.04.4** ISO. The installer self-updated to the 24.04.5 installer (subiquity) at the start.
* Caught at the confirmation screen: the ISO was first set to **22.04.5** by mistake and was switched to 24.04.4 before the VM was created.
* Network: DHCP on the VirtIO NIC.
* Storage: whole disk, **no LVM, no disk encryption**
  * `/` — 62.948 GB ext4
  * `/boot/efi` — 1.049 GB fat32
  * Encryption was skipped on purpose. A passphrase at every boot would leave an unattended server stuck at a prompt after reboots or power outages. Sensitive data will go in an encrypted ZFS dataset on `tank` instead.
* OpenSSH server installed during setup.

**Post-install**

* `sudo apt update && sudo apt full-upgrade -y`
* Applied the same hybrid-CPU stability parameters used on the Proxmox host and docker-01, in `/etc/default/grub`:

  ```
  GRUB_CMDLINE_LINUX_DEFAULT="nosoftlockup split_lock_detect=off nohz_full=off quiet"
  ```

  Then `sudo update-grub && sudo reboot`. Checked `cat /proc/cmdline` to confirm all three parameters are active on the GA 6.8 kernel.

* **Tailscale** installed with the official script (`curl -fsSL https://tailscale.com/install.sh | sh`). Node `media-01` is connected, and **key expiry is disabled** in the admin console. The old media-01 node had been removed beforehand so the new one could keep the `media-01` name.
* **DHCP reservation** for media-01 set on the router.
* **Claude Code** installed (`curl -fsSL https://claude.ai/install.sh | bash`) to `~/.local/bin`. `~/.local/bin` added to `PATH` in `~/.bashrc`.
* **Docker** installed with the official script (`curl -fsSL https://get.docker.com | sudo sh`).
  * Rootless mode was not used, because it complicates GPU device access and storage mount permissions.
  * Admin user added to the `docker` group (`sudo usermod -aG docker $USER`). Tested with `docker run hello-world`.

---

### Changed — Media Storage: NFS → virtiofs

The old media-01 used an NFS mount from the Proxmox host. The new VM uses **virtiofs**, Proxmox's built-in way to pass a host directory straight into a VM.

**ZFS (on the Proxmox host)**

```
zfs set recordsize=1M tank/media          # tuned for large media files; children inherit it
zfs create tank/media/audiobooks
chown -R <uid>:<gid> /tank/media          # UID/GID of the admin user inside media-01
```

Resulting datasets:

```
NAME                    USED  AVAIL  REFER  MOUNTPOINT
tank                   10.4G  10.8T    96K  /tank
tank/backups           10.4G  10.8T  10.4G  /tank/backups
tank/media              192K  10.8T    96K  /tank/media
tank/media/audiobooks    96K  10.8T    96K  /tank/media/audiobooks
```

**Proxmox**

* Datacenter → **Directory Mappings** → `media` → `/tank/media` on node `homelab`.
  * The parent `/tank/media` is mapped (not just `audiobooks`), so future child datasets are shared without new mappings.
* VM 102 → Hardware → Add → **Virtiofs** → directory ID `media`, default options (no xattr/ACL/direct IO).
* Full VM stop/start so the new device would attach.

**Inside media-01**

```
sudo mkdir -p /mnt/media
sudo mount -t virtiofs media /mnt/media
echo 'media /mnt/media virtiofs defaults,nofail 0 0' | sudo tee -a /etc/fstab
```

**Verification**

* `ls -la /mnt/media` shows `audiobooks`, owned by the VM's admin user.
* Wrote `/mnt/media/audiobooks/test-from-vm` inside the VM and confirmed it appeared at `/tank/media/audiobooks/` on the host. So writes cross the ZFS child-dataset boundary correctly.
* Unmounted and ran `sudo mount -a`, which remounted cleanly from fstab.

---

### Added — Audiobookshelf

* Compose stack at `~/docker/audiobookshelf/` (copy in [`docker/audiobookshelf/`](./docker/audiobookshelf/)).
* Image `ghcr.io/advplyr/audiobookshelf:latest`.
* Host port set with `ABS_PORT` in `.env` (see `.env.example`), mapped to container port 80.
* Library: `/mnt/media/audiobooks` (on `tank`) → `/audiobooks`.
* `config/` (SQLite database) and `metadata/` are kept on the VM's local disk, not on the virtiofs share.
* Root user created through the initial setup screen.
* Accessed over Tailscale. The iOS client **SoundLeaf** connects to the server.

---

### Added — Cockpit (web admin for the VM)

```
sudo apt install -y cockpit && sudo systemctl enable --now cockpit.socket
```

* Web UI over HTTPS (self-signed certificate). Logs in with the VM's Linux account.

---

### Added — Portainer CE (web GUI for Docker)

* Compose stack at `~/docker/portainer/` (copy in [`docker/portainer/`](./docker/portainer/)).
* Image `portainer/portainer-ce:lts`.
* Host port set with `PORTAINER_PORT` in `.env` (see `.env.example`).
* First-run setup required a **setup token** from the container logs (`docker logs portainer`). Edge compute features left disabled.
* Dashboard shows the `audiobookshelf` and `portainer` stacks.

---

### Verification — Services Listening on media-01

Checked with `sudo ss -tlnp`. SSH, Cockpit, Portainer and Audiobookshelf were all listening, and DNS (systemd-resolved) was bound to localhost only.

---

### Investigation — iGPU Passthrough Readiness (for Plex)

On the Proxmox host:

```
dmesg | grep -i -e DMAR -e IOMMU
lspci -nn | grep -i -e vga -e display
```

* IOMMU (VT-d) is active: DMAR units initialized, IRQ remapping enabled in x2apic mode, default domain type `Translated`.
* iGPU: `00:02.0 VGA compatible controller [0300]: Intel Corporation AlderLake-S GT1 [8086:4680]` (UHD 770 on the i5-13500), added to **IOMMU group 0**.
* Passthrough **not done yet.** Deciding between:
  * full iGPU passthrough to media-01 (the host loses its own video output), or
  * buying an Intel Arc card (A380 is the one being considered) for Plex and leaving the iGPU on the host.

---

### Known Issues / Open Items

* **Not verified yet:**
  * that VM 102's minimum memory equals its maximum (needed for PCI passthrough)
  * that the iGPU is the only device in its IOMMU group
* A security hardening pass on media-01 and the Proxmox host is pending. Details are tracked privately.
* Leftover empty plain directories in `/tank/media` from the March setup need cleaning up.
* Deliberately staying on Ubuntu 24.04 LTS (supported to 2029) rather than upgrading to 26.04 while the media stack is built out.

---

### Outcome

* media-01 rebuilt on Ubuntu Server 24.04 with the GA 6.8 kernel, hybrid-CPU GRUB fix, Tailscale (no key expiry), reserved LAN IP, Docker, and Claude Code.
* Media storage moved from NFS to **virtiofs**, backed by a dedicated ZFS dataset (`tank/media/audiobooks`, `recordsize=1M`), with an auto-mount at boot.
* **Audiobookshelf**, **Cockpit**, and **Portainer** running and reachable over Tailscale.
* Host confirmed ready for GPU passthrough. Plex is the next phase.

---

## 2026-03-09 — Monitoring Stack Deployment & ATM10 Crash

### Incident — ATM10 Crash (Reliquary Mod SIGSEGV)

* ATM10 server crashed mid-session with JVM `SIGSEGV` originating from the **Reliquary** mod (`MobCharmItem$$Lambda`).
* `docker kill` and `docker compose restart` both hung due to a zombie container process.
* Forced container removal with:

  ```
  docker rm -f mc-atm10
  ```
* Docker daemon became unresponsive; required hard reboot of the **docker-01 VM** via Proxmox.

**Resolution**

* VM reboot cleared zombie Docker processes.
* Server successfully restarted on **ATM10 version 6.0.1**.

**Action Items**

* Investigate **Reliquary mod update or removal** to prevent recurrence.

---

### Added — Grafana + Prometheus Monitoring Stack

* Created monitoring stack directory:

  ```
  ~/docker/monitoring/
  ```
* Deployed via `docker-compose.yml` with three containers:

  * `grafana`
  * `prometheus`
  * `node_exporter`

**Firewall Configuration**

* Added UFW rules on `tailscale0` interface:

  * `3000` — Grafana
  * `9090` — Prometheus
  * `9100` — Node Exporter

**Verification**

* Prometheus target health page confirmed all exporters **UP**.
* Fixed Grafana datasource URL to:

  ```
  http://prometheus:9090
  ```
* Imported **Node Exporter Full** dashboard (Grafana ID `1860`).

**Access**

* Grafana available via **Tailscale** on port `3000`
* Prometheus available via **Tailscale** on port `9090`

**Note**

* Grafana **Nodename dropdown** does not populate Proxmox host.
* Switch host view using **Job selector** instead.

---

### Added — Proxmox Host Monitoring

* Installed **node_exporter v1.8.2** directly on the Proxmox host.

**Binary**

```
/usr/local/bin/node_exporter
```

**Systemd Service**

```
/etc/systemd/system/node_exporter.service
```

**Firewall Rule**

```
iptables -A INPUT -p tcp --dport 9100 -j ACCEPT
```

**Persistence**

```
/etc/iptables/rules.v4
```

**Prometheus Integration**

* Added Proxmox scrape target in `prometheus.yml`.
* Used **Tailscale IP** instead of LAN IP due to connectivity issues caused by Proxmox `vmbr0` bridge routing.

---

### Infrastructure Changes

* Increased **docker-01 VM memory allocation**

```
27 GB → 32 GB (32768 MB)
```

---

### Known Issues

* Monitoring stack does **not auto-start after VM reboot** — requires manual:

  ```
  docker compose up -d
  ```
* Grafana Nodename dropdown missing Proxmox host.
* Proxmox LAN IP unreachable from docker-01 due to `vmbr0` bridge routing behavior.

---

### Outcome

* Full **Grafana + Prometheus monitoring stack operational**.
* Monitoring coverage includes:

  * `docker-01` VM
  * Proxmox host
* All Prometheus targets reporting **UP**.
* ATM10 server stabilized on **version 6.0.1**.

---

## 2026-03-08 — Homelab Recovery & ATM10 Rollback

### Incident — ISP Outage

* Area ISP outage caused temporary loss of connectivity.
* Both **Proxmox host** and **docker-01 VM** went offline during outage.
* Systems restored automatically once power and internet returned.

**Verification**

* docker-01 confirmed reachable at reserved LAN IP.

---

### Investigation — ATM10 TPS Spikes

Observed:

* Severe TPS drops every ~10–14 minutes.
* Random player disconnects:

  ```
  java.net.SocketException: Connection reset
  ```

Root cause identified:

* ATM10 **auto-updated from 6.0 → 6.1** during restart.
* ATM10 `6.1` contains a broken tag from the `pamhc2trees` mod:

```
minecraft:pig_food
```

* This caused an **LMFT tag reload loop**, repeatedly stalling the main server thread and collapsing TPS.

Issue reported upstream with full logs and mod list.

---

### Fix — ATM10 Rollback to 6.0.1

**Performance Adjustment**

```
simulation-distance=6
```

**Backup**

```
cp -r data/mods data/mods_6.1_backup
```

**Installed unzip**

```
sudo apt install unzip -y
```

**Transferred ServerFiles**

* SCP from Windows workstation → docker-01 VM.

**Replaced mods directory**

* Extracted **ServerFiles-6.0.1.zip**
* Restored correct mod set.

---

### CurseForge Deployment Lessons

Correct configuration required:

```
CF_FILE_ID=<CLIENT file ID>
```

Important findings:

* `CF_PAGE_ID` → **not a valid variable**
* `CF_FORCE_SYNCHRONIZE` → does **not prevent mod re-download**
* Using **server pack file ID** causes:

  ```
  missing manifest
  ```
* Must use the **CLIENT file ID** instead.

---

### Outcome

* ATM10 server stable on **version 6.0.1** with version pinned.
* TPS stabilized.


## 2026-03-02 — ATM10 Minecraft Server Deployment

### Objective
Deploy a modded Minecraft server (All The Mods 10, NeoForge) using Docker on docker-01, accessible publicly via DuckDNS and secured with UFW firewall rules.

### Actions Taken
- Deployed `mc-atm10` container using `itzg/minecraft-server` image with CurseForge modpack support
- Configured modpack type, slug, and memory allocation via `.env`
- Tuned JVM flags for performance (G1GC garbage collector):
- Set swap to 0 to prevent memory thrashing
- Deployed DuckDNS container on docker-01 for dynamic DNS (subdomain and token in `.env`)
- Enabled UFW firewall on docker-01 with deny-all inbound, SSH and Minecraft port allowlisted
- Verified UFW rules applied for both IPv4 and IPv6

### Issues Encountered
- Modpack download time was significant on first container start
- Required careful JVM memory allocation to stay within docker-01's available RAM

### Resolution
- Allocated approximately 52% of total RAM to JVM heap — balanced for stable performance with small private group
- Confirmed container starts cleanly and server is reachable via DuckDNS address

### Outcome
- Running ATM10 NeoForge modpack in Docker
- Publicly accessible via DuckDNS dynamic DNS
- Protected by UFW with deny-all inbound default, allowlist only
- JVM tuned for low-latency GC with G1GC

---

## 2026-03-02 — Valheim Dedicated Server Deployment

### Objective
Deploy a Valheim dedicated server using Docker on docker-01 for private group play, accessible via LAN and externally via port forward.

### Actions Taken
- Deployed `valheim` container using `lloesche/valheim-server` image
- Configured server name, world name, and password via `.env`
- Forwarded required UDP ports on router to docker-01 (LAN IP statically reserved via DHCP)
- Confirmed docker-01 has a static LAN IP reservation in router

### Issues Encountered
- Valheim requires UDP — confirmed both UFW and router port forward rules specify UDP protocol

### Resolution
- Verified UDP protocol on all relevant rules
- Confirmed server visible in Valheim server browser

### Outcome
- Running on docker-01 with static LAN IP
- Accessible via LAN and external port forward
- All sensitive values stored in `.env`
- Stable alongside Minecraft server with sufficient RAM headroom
---

## 2026-02-26 — SSH-Based Git Authentication & Repo Hardening

### Objective
Transition repository access from HTTPS (password-based) to SSH key authentication for secure, passwordless Git operations.

### Actions Taken
- Generated ED25519 SSH key pair on docker-01:
  - `ssh-keygen -t ed25519`
- Verified key fingerprint using:
  - `ssh-keygen -lf ~/.ssh/id_ed25519.pub`
- Added public key to GitHub under SSH keys.
- Switched Git remote from HTTPS to SSH:
  - `git remote set-url origin git@github.com:donovanwhitcomb/homelab-evolution.git`
- Validated authentication with:
  - `ssh -T git@github.com`
- Successfully pushed repository using SSH without password prompts.

### Issues Encountered
- Initial authentication failure (`Permission denied (publickey)`).
- Remote misconfiguration due to hostname typo.
- Incorrect file path spacing when verifying SSH key.
- Confusion between HTTPS and SSH remotes.

### Resolution
- Corrected remote URL syntax.
- Confirmed matching SHA256 fingerprint between local key and GitHub.
- Verified SSH authentication success.
- Confirmed successful `git push` via SSH.

### Outcome
Repository is now:
- Using SSH-based authentication
- Fully synchronized between local lab and GitHub
- Configured for secure, passwordless Git operations
