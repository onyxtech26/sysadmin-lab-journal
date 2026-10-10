# Lab Entry 3.3 & 3.4: Package Management & systemd Service Lifecycle

**Phase:** Phase 3: Linux Systems Administration & Hardening  
**Target Node:** `lab-linux-01` (`10.10.10.11`)  
**Operating System:** Ubuntu Server 22.04 LTS (Headless)  
**Date Completed:** 2026-10-10  

---

## 1. Scenario & Objectives

Modern Linux distributions utilize **APT (Advanced Package Tool)** for dependency-resolved software lifecycle management and **`systemd`** as the init system and service orchestrator (PID 1). Logging is unified through **`journald`**.

* **Business Objective:** Deploy enterprise-grade HTTP web services (Nginx) and cross-platform file sharing (Samba), establish service persistence across system reboots, and master operational lifecycle commands.
* **Technical Scope:**
  1. Synchronize APT repository metadata and deploy `nginx` and `samba`.
  2. Transition `nginx` through service lifecycle states (`active` -> `inactive` -> `started` -> `enabled`).
  3. Query structured logging streams via `journalctl` filtered by unit scope.
  4. Perform Layer 7 HTTP health checks via `curl`.

---

## 2. Pre-Configuration State

* **Target Node:** `lab-linux-01` (`10.10.10.11/24`)
* **Administrative Context:** `sysadmin` with active `sudo` privileges over OpenSSH.
* **Network Status:** Connected to `LabNet` with DNS resolution through `DC01` / `8.8.8.8`.

---

## 3. Implementation Steps & Commands

### 3.1 Package Deployment (APT)
```bash
# Update repository package indices
sudo apt update

# Deploy Nginx HTTP server and Samba file sharing suite
sudo apt install nginx samba -y
```

> **Design Note:**  
> On Debian/Ubuntu derivatives, APT automatically enables and launches newly installed daemons by default. On Red Hat/Enterprise Linux derivatives (RHEL, Rocky, AlmaLinux), the equivalent command `sudo dnf install nginx` leaves the daemon in a stopped state until explicitly enabled by the administrator.

### 3.2 Service Lifecycle Orchestration (`systemctl`)
```bash
# 1. Inspect current running state
sudo systemctl status nginx

# 2. Halt the service daemon
sudo systemctl stop nginx

# 3. Re-start the daemon
sudo systemctl start nginx

# 4. Enable automatic start at boot time (creates symlinks in /etc/systemd/system/multi-user.target.wants/)
sudo systemctl enable nginx
```

### 3.3 Centralized Log Inspection (`journalctl`)
```bash
# Query the systemd journal for events generated specifically by nginx.service
sudo journalctl -u nginx -n 20 --no-pager
```

> **Why `journalctl` instead of `/var/log/nginx/`?**  
> While Nginx still maintains its application-level `access.log` and `error.log` in `/var/log/nginx/`, process startup failures, system crashes, and wrapper execution errors originate at the systemd supervisor level. `journalctl -u <unit>` queries the structured binary journal directly from the kernel and init process, providing timestamped visibility into service state transitions.

### 3.4 Layer 7 Health Check
```bash
# Issue an HTTP HEAD request against local loopback socket
curl -I http://localhost
```

---

## 4. Verification & Testing

### 4.1 Service Deactivation Verification
```text
sysadmin@lab-linux-01:~$ sudo systemctl status nginx
○ nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: inactive (dead) since Sat 2026-10-10 12:07:22 UTC; 16s ago
   Duration: 48.856s
   Main PID: 2684 (code=exited, status=0/SUCCESS)
```

### 4.2 Service Re-activation & Boot Persistence
```text
sysadmin@lab-linux-01:~$ sudo systemctl start nginx
sysadmin@lab-linux-01:~$ sudo systemctl enable nginx
Synchronizing state of nginx.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable nginx
```

### 4.3 Structured Journal Stream Audit
```text
sysadmin@lab-linux-01:~$ sudo journalctl -u nginx -n 20 --no-pager
Oct 10 12:06:33 lab-linux-01 systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Oct 10 12:06:33 lab-linux-01 systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
Oct 10 12:07:22 lab-linux-01 systemd[1]: Stopping nginx.service - A high performance web server and a reverse proxy server...
Oct 10 12:07:22 lab-linux-01 systemd[1]: nginx.service: Deactivated successfully.
Oct 10 12:07:22 lab-linux-01 systemd[1]: Stopped nginx.service - A high performance web server and a reverse proxy server.
Oct 10 12:07:58 lab-linux-01 systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Oct 10 12:07:58 lab-linux-01 systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
```

### 4.4 HTTP Endpoint Verification
```text
sysadmin@lab-linux-01:~$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Sat, 10 Oct 2026 12:08:56 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Sat, 10 Oct 2026 12:06:32 GMT
Connection: keep-alive
ETag: "6aca2a48-267"
Accept-Ranges: bytes
```

---

## 5. Checkpoint Summary

- [x] Package repository indices refreshed and `nginx` / `samba` deployed via APT.
- [x] Service lifecycle states (`stop`, `start`, `enable`) tested and verified under systemd.
- [x] Systemd journal parsed cleanly for `nginx.service` events.
- [x] Web server active and responding with `HTTP/1.1 200 OK`.
