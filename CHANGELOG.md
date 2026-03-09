# Homelab Evolution Log

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
