# Lab Entry 3.7: Services Configuration — Nginx Virtual Host & Samba CIFS Network Share

**Phase:** Phase 3: Linux Systems Administration & Hardening  
**Target Node:** `lab-linux-01` (`10.10.10.11`)  
**Operating System:** Ubuntu Server 22.04 / 26.04 LTS (Headless)  
**Date Completed:** 2026-10-12  

---

## 1. Scenario & Objectives

Enterprise infrastructure requires secure, multi-tenant application hosting and cross-platform network storage. Linux nodes must provide isolated HTTP virtual hosting for internal services and CIFS/SMB file sharing for heterogeneous operating systems (Windows and Linux) while maintaining strict POSIX permission boundaries.

* **Business Objective:** Deploy a dedicated virtual host on Nginx with an enterprise dashboard landing page and configure a collaborative multi-user Samba network share (`/srv/labshare`) for the engineering team (`devteam`).
* **Technical Scope:**
  1. Provision dedicated web root `/var/www/lab-linux-01/html` and construct an enterprise HTML status portal.
  2. Author modular Nginx server block in `/etc/nginx/sites-available/lab-linux-01` with logging channels, disable stock default vhost, and activate configuration.
  3. Deploy filesystem storage root `/srv/labshare` with SGID permissions (`2775`) owned by `sysadmin:devteam`.
  4. Configure Samba CIFS share `[labshare]` in `/etc/samba/smb.conf` restricting access to `@devteam` and `sysadmin`.
  5. Initialize Samba cryptographic database (`smbpasswd` / `pdbedit`) for `sysadmin` and `bob`.
  6. Execute SMB Layer 7 verification with `smbclient` confirming authenticated write access, file masking (`0664`), and SGID group inheritance.

---

## 2. Pre-Configuration State

* **Target IP Address:** `10.10.10.11/24` (Interface `enp0s3`)
* **Installed Packages:** `nginx` (`v1.28+`), `samba` (`v4.23+`)
* **Existing Groups & Users:** User `bob` (`uid=1001`), User `sysadmin` (`uid=1000`), Group `devteam` (`gid=1002`)

---

## 3. Implementation Steps & Commands

### 3.1 Nginx Dedicated Virtual Host & Landing Page
```bash
# 1. Create document root
sudo mkdir -p /var/www/lab-linux-01/html

# 2. Deploy enterprise HTML status dashboard
# (File created at /var/www/lab-linux-01/html/index.html)

# 3. Configure virtual host server block
sudo bash -c 'printf "server {\n    listen 80;\n    listen [::]:80;\n\n    server_name lab-linux-01 lab-linux-01.lab.local localhost 10.10.10.11;\n\n    root /var/www/lab-linux-01/html;\n    index index.html;\n\n    location / {\n        try_files \$uri \$uri/ =404;\n    }\n\n    access_log /var/log/nginx/lab-linux-01.access.log;\n    error_log /var/log/nginx/lab-linux-01.error.log;\n}\n" > /etc/nginx/sites-available/lab-linux-01'

# 4. Deactivate default site, activate custom site, and test syntax
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sf /etc/nginx/sites-available/lab-linux-01 /etc/nginx/sites-enabled/
sudo nginx -t

# 5. Reload Nginx
sudo systemctl reload nginx
```

### 3.2 Samba Directory Provisioning & SGID Configuration
```bash
# Create shared directory
sudo mkdir -p /srv/labshare

# Assign group ownership to devteam
sudo chown -R sysadmin:devteam /srv/labshare

# Apply octal mode 2775 (rwxrwsr-x)
# The leading '2' activates SGID, ensuring newly created files inherit group 'devteam'
sudo chmod -R 2775 /srv/labshare
```

### 3.3 Samba Share Configuration (`/etc/samba/smb.conf`)
```bash
# Append [labshare] configuration stanza
sudo bash -c 'printf "\n[labshare]\n   comment = Enterprise Engineering Share\n   path = /srv/labshare\n   browseable = yes\n   read only = no\n   guest ok = no\n   valid users = @devteam, sysadmin\n   create mask = 0664\n   directory mask = 0775\n   force group = devteam\n" >> /etc/samba/smb.conf'

# Validate syntax
sudo testparm -s --section-name=labshare
```

### 3.4 Samba User Provisioning & Service Restart
```bash
# Provision accounts in Samba passdb.tdb
sudo smbpasswd -a sysadmin
sudo smbpasswd -a bob

# Restart SMB/NMB daemons
sudo systemctl restart smbd nmbd
```

---

## 4. Verification & Testing

### 4.1 Nginx HTTP Endpoint Verification
```bash
sysadmin@lab-linux-01:~$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Content-Type: text/html
Content-Length: 2125
```

### 4.2 Samba Configuration Check (`testparm`)
```text
[labshare]
        comment = Enterprise Engineering Share
        create mask = 0664
        directory mask = 0775
        force group = devteam
        path = /srv/labshare
        read only = No
        valid users = @devteam sysadmin
```

### 4.3 Authenticated SMB File Creation Test (`smbclient`)
```bash
sysadmin@lab-linux-01:~$ smbclient //localhost/labshare -U bob -c 'put /etc/issue testfile.txt; ls'
putting file /etc/issue as \testfile.txt (3.9 kB/s)
  .                                   D        0  Mon Oct 12 12:32:30 2026
  ..                                  D        0  Mon Oct 12 12:32:30 2026
  testfile.txt                        A       24  Mon Oct 12 12:32:30 2026
```

### 4.4 POSIX SGID Inheritance Verification
```bash
sysadmin@lab-linux-01:~$ ls -la /srv/labshare
total 12
drwxrwsr-x 2 sysadmin devteam 4096 Oct 12 12:32 .
drwxr-xr-x 4 root     root    4096 Oct 12 12:18 ..
-rw-rw-r--+1 bob      devteam   24 Oct 12 12:32 testfile.txt
```

- [x] Nginx custom vhost active and serving HTTP 200 OK
- [x] Samba `[labshare]` configured with `@devteam` access restriction
- [x] Samba accounts provisioned (`bob`, `sysadmin`)
- [x] Authenticated CIFS write verified via `smbclient`
- [x] Filesystem SGID inheritance (`2775`) verified (`bob:devteam`, `0664`)

---

## 5. Troubleshooting & Incident Log (STAR Format)

* **Situation:** During initial Nginx configuration generation, pasting interactive heredocs triggered line indentation whitespace errors, causing bash prompts to hang waiting for EOF delimiters.
* **Task:** Safely write configuration files non-interactively without risk of whitespace or escaping truncation.
* **Action:** Swapped interactive heredocs for explicit `printf` one-liners redirected through `sudo bash -c`. Verified syntax immediately using `sudo nginx -t`.
* **Result:** Configuration written cleanly, syntax check passed with code 0, and Nginx reloaded seamlessly.
