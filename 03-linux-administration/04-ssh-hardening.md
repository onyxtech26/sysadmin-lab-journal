# Lab Entry 3.6: OpenSSH Cryptographic Hardening & Key Authentication

**Phase:** Phase 3: Linux Systems Administration & Hardening  
**Target Node:** `lab-linux-01` (`10.10.10.11`)  
**Operating System:** Ubuntu Server 22.04 / 26.04 LTS (Headless)  
**Date Completed:** 2026-10-12  

---

## 1. Scenario & Objectives

Remote server access must be secured against credential brute-forcing, password spraying, and unauthorized interactive logins. Standard enterprise baseline hardening mandates the deprecation of interactive password authentication in favor of asymmetric cryptographic keys (Ed25519) and blocking direct root SSH sessions.

* **Business Objective:** Eliminate password-based vector attacks against remote administration interfaces while preserving seamless, secure jump-box access for verified systems engineers.
* **Technical Scope:**
  1. Generate dedicated asymmetric Ed25519 keypair on the host management workstation.
  2. Install public key into target user (`sysadmin`) `~/.ssh/authorized_keys` with strict POSIX permissions (`0600` / `0700`).
  3. Deploy modular SSH hardening policy in `/etc/ssh/sshd_config.d/01-hardening.conf`.
  4. Disable `PasswordAuthentication` and set `PermitRootLogin no`.
  5. Configure client-side SSH host aliases (`~/.ssh/config`) for automated routing.
  6. Perform dual-vector validation: confirm keypair acceptance and enforce password denial.

---

## 2. Pre-Configuration State

* **Target IP Address:** `10.10.10.11/24` (NAT Port Forward: `127.0.0.1:2222 -> 10.10.10.11:22`)
* **Initial SSH Settings:** `PasswordAuthentication yes`, `PermitRootLogin prohibit-password`
* **Administrative User:** `sysadmin` (`uid=1000`)

---

## 3. Implementation Steps & Commands

### 3.1 Key Generation (Host Workstation - PowerShell)
```powershell
# Generate dedicated Ed25519 keypair for lab administration without passphrase for automated tasks
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_labadmin" -C "sysadmin@lab-linux-01" -N '""'
```

### 3.2 Key Installation & Permission Hardening (Target Node)
```bash
# Append public key to authorized keys spool
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHk8dm5mGbAHJK0zCwTQaH/uBiGMX06yZc5NVLxuAPO2 sysadmin@lab-linux-01' >> ~/.ssh/authorized_keys

# Enforce strict owner-only permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 3.3 Modular Daemon Hardening (`sshd_config.d`)
```bash
# Place hardening directives in drop-in configuration
sudo bash -c 'cat <<EOF > /etc/ssh/sshd_config.d/01-hardening.conf
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
EOF'

# Validate syntax
sudo sshd -t

# Reload SSH service
sudo systemctl reload ssh || sudo systemctl restart ssh
```

### 3.4 Client Host Alias Configuration (`~/.ssh/config`)
```text
Host lab-linux-01
    HostName 127.0.0.1
    Port 2222
    User sysadmin
    IdentityFile ~/.ssh/id_ed25519_labadmin
    IdentitiesOnly yes
```

---

## 4. Verification & Testing

### 4.1 Effective Daemon Configuration Evaluation
```bash
sysadmin@lab-linux-01:~$ sudo sshd -T | grep -iE 'passwordauthentication|pubkeyauthentication|permitrootlogin'
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

### 4.2 Key Authentication Verification (Host)
```powershell
PS C:\Users\ShadowBlade> ssh lab-linux-01 "hostname && whoami"
lab-linux-01
sysadmin
```

### 4.3 Password Authentication Enforcement Test (Host)
Attempting to connect with `PubkeyAuthentication=no` proves passwords cannot be used as fallback:
```powershell
PS C:\Users\ShadowBlade> ssh -o PubkeyAuthentication=no -p 2222 sysadmin@127.0.0.1
sysadmin@127.0.0.1: Permission denied (publickey).
```

- [x] SSH daemon syntax validated (`sshd -t`)
- [x] Ed25519 public key installed with `0600` permissions
- [x] Password authentication disabled (`PasswordAuthentication no`)
- [x] Direct root login blocked (`PermitRootLogin no`)
- [x] Host alias operational (`ssh lab-linux-01`)
- [x] Fallback password attempt rejected with `Permission denied (publickey)`

---

## 5. Troubleshooting & Incident Log (STAR Format)

* **Situation:** During initial verification, client SSH attempts still prompted for password, and `sshd -T` reported `passwordauthentication yes` despite an existing `/etc/ssh/sshd_config.d/01-hardening.` file.
* **Task:** Identify why the hardening directive was not being loaded and ensure key-based login worked before locking down passwords.
* **Action:** 
  1. Ran `ls -la /etc/ssh/sshd_config.d/` and observed the file was named `01-hardening.` without the required `.conf` extension defined in `Include /etc/ssh/sshd_config.d/*.conf`.
  2. Identified that client-side `id_ed25519_lab` had an unknown passphrase, generated fresh dedicated key `id_ed25519_labadmin`, and deployed it to `authorized_keys`.
  3. Renamed file to `/etc/ssh/sshd_config.d/01-hardening.conf` and reloaded the daemon.
* **Result:** `sshd -T` immediately reflected `passwordauthentication no`, key login authenticated instantly, and unauthorized password requests were rejected.
