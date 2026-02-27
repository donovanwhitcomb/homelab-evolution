# Homelab Evolution Log

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
