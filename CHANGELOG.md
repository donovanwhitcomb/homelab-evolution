# Homelab Evolution Log

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
