# Lab Entry 3.8: Linux Systems Troubleshooting & Operational Incident Simulations

**Phase:** Phase 3: Linux Systems Administration & Hardening  
**Target Node:** `lab-linux-01` (`10.10.10.11`)  
**Operating System:** Ubuntu Server 22.04 / 26.04 LTS (Headless)  
**Date Completed:** 2026-10-13  

---

## 1. Scenario & Objectives

Enterprise systems operations require administrators to rapidly detect, isolate, and remediate service degradation, storage capacity alerts, and permission lockouts without introducing collateral system damage or unnecessary downtime.

* **Business Objective:** Validate incident triage procedures against three real-world production failure scenarios: daemon configuration syntax errors, sudden filesystem saturation, and POSIX permission boundary denials.
* **Technical Scope:**
  1. **Incident 1 (Daemon Failure):** Corrupt Nginx virtual host configuration, capture systemd failure messages, isolate syntax breakages via `nginx -t`, and restore clean operation.
  2. **Incident 2 (Storage Exhaustion):** Simulate a 2.0 GB unmonitored runaway log via `dd`, detect root partition pressure with `df -h`, locate top storage consumers with `find -xdev`, and safely reclaim disk capacity.
  3. **Incident 3 (Permission Denial):** Simulate restrictive permission drift (`chmod 700`) triggering HTTP 403 Forbidden errors, query structured access error logs in `/var/log/nginx/`, and enforce standard least-privilege web permissions (`755` / `644`).

---

## 2. Pre-Configuration State

* **Target IP Address:** `10.10.10.11/24` (Interface `enp0s3`)
* **Baseline Storage Utilization:** `12G` volume, `5.1G` used, `5.7G` available (`48%` capacity)
* **Active Web Services:** Nginx (`v1.28+`) serving `/var/www/lab-linux-01/html`

---

## 3. Implementation Steps & Commands

### 3.1 Incident 1: Nginx Configuration Syntax Corruption & Recovery
```bash
# 1. Induce simulated directive corruption
sudo sed -i 's/listen 80;/listen 80;\n    broken_directive_typo on;/' /etc/nginx/sites-available/lab-linux-01

# 2. Attempt service reload (Fails with systemd error)
sudo systemctl reload nginx

# 3. Diagnostic triage using native linter
sudo nginx -t

# 4. Remediation: Purge faulty directive and verify syntax
sudo sed -i '/broken_directive_typo/d' /etc/nginx/sites-available/lab-linux-01
sudo nginx -t

# 5. Reload daemon and verify state
sudo systemctl reload nginx
systemctl is-active nginx
```

### 3.2 Incident 2: Disk Exhaustion Simulation & Storage Triage
```bash
# 1. Establish baseline capacity
df -h /

# 2. Simulate rapid runaway disk consumption (2 GB dump file)
sudo dd if=/dev/zero of=/var/log/runaway_system_dump.log bs=1M count=2048 status=progress

# 3. Confirm capacity alert (Jumped to 66% utilization)
df -h /

# 4. Discover rogue large files (>500M) across filesystem boundary (-xdev)
sudo find / -xdev -type f -size +500M -exec ls -lh {} + 2>/dev/null | awk '{print $5, $9}'

# 5. Safely reclaim storage and re-verify baseline
sudo rm -f /var/log/runaway_system_dump.log
df -h /
```

### 3.3 Incident 3: Web Root Permission Drift (HTTP 403 Forbidden) & POSIX Normalization
```bash
# 1. Induce permission lockout on web root
sudo chmod 700 /var/www/lab-linux-01/html

# 2. Health check returns Layer 7 error
curl -I http://localhost # Yields HTTP/1.1 403 Forbidden

# 3. Inspect application error logs
sudo tail -n 2 /var/log/nginx/lab-linux-01.error.log
# Captures: "[error] ... is forbidden (13: Permission denied)"

# 4. Restore proper directory (755) and file (644) modes
sudo chmod 755 /var/www/lab-linux-01/html
sudo chmod 644 /var/www/lab-linux-01/html/index.html

# 5. Verify service restoration
curl -I http://localhost # Yields HTTP/1.1 200 OK
```

---

## 4. Verification & Testing

### 4.1 Linter Diagnostics Output (`nginx -t`)
```text
2026/10/12 12:38:43 [emerg] 3435#3435: unknown directive "broken_directive_typo" in /etc/nginx/sites-enabled/lab-linux-01:3
nginx: configuration file /etc/nginx/nginx.conf test failed
```

### 4.2 Storage Consumption Discovery Output
```text
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  7.1G  3.7G  66% /

Top Consumers Identified (>500M):
2.0G /swap.img
2.0G /var/log/runaway_system_dump.log
```

### 4.3 Permission Remediation Verification
```text
sysadmin@lab-linux-01:~$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Content-Type: text/html
Content-Length: 2126
```

- [x] Corrupted configuration identified and pinpointed to exact line number
- [x] Nginx service successfully recovered and verified active
- [x] Sudden disk consumption isolated via `find -xdev` and reclaimed to baseline (48%)
- [x] HTTP 403 Forbidden error diagnosed via error log inspection
- [x] POSIX permissions normalized (`755` directory / `644` files) and HTTP 200 OK verified

---

## 5. Troubleshooting & Incident Log (STAR Format)

* **Situation:** In modern system operations, unannounced service interruptions frequently stem from typos in configuration files, unmonitored disk filling, and faulty directory permissions.
* **Task:** Systematically reproduce and resolve each failure mode without rebooting the server or causing data loss.
* **Action:**
  1. Utilized pre-flight configuration tests (`nginx -t`) prior to restarting services to prevent downtime.
  2. Executed bounded single-filesystem scans (`find -xdev`) to isolate space consumption while avoiding virtual system paths (`/proc`, `/sys`).
  3. Correlated Layer 7 HTTP status codes (403 Forbidden) with filesystem permissions and application error logs.
* **Result:** All three incidents resolved smoothly, validating root-cause troubleshooting workflows.
