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
| **3.6** | [04-ssh-hardening.md](./04-ssh-hardening.md) | Ed25519 public key authentication, SSH daemon hardening (`sshd_config`), disabling password auth | In Progress |
| **3.7** | [05-services-nginx-samba.md](./05-services-nginx-samba.md) | Nginx virtual host serving, Samba CIFS network share (`/srv/labshare`), cross-platform integration | Upcoming |
| **3.8** | [06-troubleshooting-incidents.md](./06-troubleshooting-incidents.md) | Self-inflicted incident diagnosis: stopped daemons, simulated disk exhaustion (`dd`), and file permission locks | Upcoming |

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
