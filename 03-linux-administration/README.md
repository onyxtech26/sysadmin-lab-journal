# Phase 3: Linux Systems Administration & Hardening

**Node:** `lab-linux-01`  
**Operating System:** Ubuntu Server 22.04 LTS (Headless)  
**Assigned Static IP:** `10.10.10.11/24`  
**Default Gateway:** `10.10.10.1`  
**Upstream DNS:** `10.10.10.10` (`DC01`) / `8.8.8.8`  

---

## 1. Phase Overview

Phase 3 focuses on enterprise Linux administration, role-based access control (RBAC), service orchestration via `systemd`, task automation with `cron`, cryptographic OpenSSH hardening, and cross-platform file sharing using Samba.

In corporate environments, Linux servers underpin critical back-end services, web servers, databases, and containers. Managing these nodes entirely via remote headless sessions requires strict operational discipline, security hygiene, and diagnostic proficiency across system logs and process trees.

---

## 2. Phase 3 Roadmap & Documentation Index

| Step | Document | Focus & Key Technologies | Status |
| :--- | :--- | :--- | :--- |
| **3.1 – 3.2** | [01-rbac-users-groups.md](./01-rbac-users-groups.md) | User & group lifecycle, octal permission delegation (`chmod 750`), safe `sudoers` drop-in rules (`/etc/sudoers.d/`) | **Complete** |
| **3.3 – 3.4** | [02-package-management-systemd.md](./02-package-management-systemd.md) | APT repositories, Nginx web server lifecycle, systemd unit control & journald log inspection | **Complete** |
| **3.5** | [03-cron-automation.md](./03-cron-automation.md) | Crontab automation, scheduled storage telemetry, log appending | **Complete** |
| **3.6** | [04-ssh-hardening.md](./04-ssh-hardening.md) | Ed25519 public key authentication, SSH daemon hardening (`sshd_config`), disabling password auth | **Complete** |
| **3.7** | [05-services-nginx-samba.md](./05-services-nginx-samba.md) | Nginx virtual host serving, Samba CIFS network share (`/srv/labshare`), cross-platform integration | **Complete** |
| **3.8** | [06-troubleshooting-incidents.md](./06-troubleshooting-incidents.md) | Self-inflicted incident diagnosis: stopped daemons, simulated disk exhaustion (`dd`), and file permission locks | **Complete** |

---

## 3. Key Architectural & Security Decisions

1. **Modular Sudoers Configuration (`/etc/sudoers.d/`)**:
   * Directly editing `/etc/sudoers` with a generic text editor introduces high risk of syntax corruption that can permanently lock administrators out of root escalation.
   * Creating isolated drop-in configuration files under `/etc/sudoers.d/` with strict `0440` permissions and validating them via `visudo -c` enforces change control and modular access delegation.

2. **Octal Permission Notation (`750` / `rwxr-x---`)**:
   * Root project directories must adhere to the principle of least privilege.
   * `750` ensures that the project owner retains write capabilities, project group members retain read and traverse access for collaboration, and unprivileged system users (`others`) have zero visibility into corporate workloads.

3. **Remote Headless Operations**:
   * All administrative operations are conducted over SSH through the VirtualBox NAT port forwarding rule (`127.0.0.1:2222 -> 10.10.10.11:22`), maintaining parity with remote datacenter bastion jump-box administration.

---

## 4. Executive Narrative: What We Actually Built & Why

This phase simulates transforming a raw, headless Ubuntu node (`lab-linux-01`) into a production-ready enterprise server under strict operational hygiene.

### 🏢 Act 1: Safe Employee Onboarding (`bob` & `devteam`)
* **Business Challenge:** Engineers need administrative powers to deploy workloads, but granting unrestricted `root` logins invites catastrophic typos or unmonitored changes.
* **Architecture Solution:** Provisioned an isolated user account (`bob`), associated him with an engineering security group (`devteam`), created a restricted team working root (`/srv/projects`, mode `750`), and delegated elevated rights safely using an isolated drop-in (`/etc/sudoers.d/devteam`) validated by `visudo -c`.

### 🌐 Act 2: Daemon Services & Network Visibility (Nginx & Samba)
* **Business Challenge:** Modern servers must provide both web services and cross-platform network shares with persistent startup behavior.
* **Architecture Solution:** Deployed Nginx HTTP server and Samba CIFS file sharing. Verified unit state persistence via `systemd` (`active` and `enabled`) and validated Layer 4 listening ports via `ss -tulpn` on ports `80` (HTTP), `139`, and `445` (SMB).

### ⏰ Act 3: Automated Nightly Maintenance (`cron`)
* **Business Challenge:** Disk volumes steadily fill over time; relying on manual sysadmin log reviews guarantees outages.
* **Architecture Solution:** Installed a headless cron schedule (`0 1 * * *`) into user `bob`'s spool file. Nightly storage telemetry (`df -h`) appends automatically to `disk-usage.log` without requiring interactive sessions.

### 🔑 Act 4: Cryptographic Perimeter Defense (SSH Hardening)
* **Business Challenge:** Exposed SSH ports with password authentication invite continuous brute-force and password-spraying attacks.
* **Architecture Solution:** Replaced password authentication with asymmetric **Ed25519** elliptic-curve keypairs (`id_ed25519_labadmin`). Deployed `/etc/ssh/sshd_config.d/01-hardening.conf` disabling `PasswordAuthentication` and direct root login.
* **Crucial Operational Reflex:** Maintained an active SSH session while verifying key authentication in a secondary window. Attempting password login returns an immediate `Permission denied (publickey)`.

### 📂 Act 5: Collaborative Multi-User Storage & SGID (`/srv/labshare`)
* **Business Challenge:** When multiple users share a network folder, files created by one user normally inherit only their primary group, blocking collaborators from editing.
* **Architecture Solution:** Configured `/srv/labshare` with the **Set Group ID (SGID)** bit (`chmod 2775`). Any file created in the directory automatically inherits the `devteam` group. Configured the Samba CIFS share with `create mask = 0664` and verified write capability with `smbclient`.

### 🚨 Act 6: Incident Triage & Root Cause Recovery
* **Business Challenge:** Production outages occur from bad syntax, storage saturation, and permission drift. Panicking or blindly rebooting compounds the downtime.
* **Architecture Solution:** Simulated three controlled failures:
  1. *Config Syntax Typo:* Diagnosed via `nginx -t` down to the exact line number, remediated without server downtime.
  2. *Filesystem Exhaustion (2 GB runaway log):* Isolated via single-filesystem boundary scan (`find / -xdev -size +500M`) and reclaimed storage instantly.
  3. *Permission Drift (HTTP 403 Forbidden):* Correlated Layer 7 HTTP 403 responses with `/var/log/nginx/lab-linux-01.error.log` and restored standard POSIX web permissions (`755` directories / `644` files).

---

## 5. Core Senior Sysadmin Takeaways

1. **Linters Over Guesswork:** Never blindly reload a service after editing its config. Always run native syntax checkers (`nginx -t`, `visudo -c`, `sshd -t`, `testparm`) first.
2. **Never Cut Your Own Tether:** When modifying remote access or firewall rules, keep your existing active management session alive until a second session connects successfully with the new credentials.
3. **Bounded Filesystem Scans:** When hunting large files on a full disk, always pass `-xdev` to `find` so the process does not walk into virtual kernel filesystems like `/proc` or `/sys`.
4. **Permissions Tell Stories:** HTTP 403 Forbidden is almost always a filesystem read/traverse issue on the web document root for the daemon's runtime user (`www-data`).

