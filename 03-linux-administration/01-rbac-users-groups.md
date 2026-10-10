# Lab Entry 3.1 & 3.2: Linux RBAC, Octal Permissions & Sudoers Delegation

**Phase:** Phase 3: Linux Systems Administration & Hardening  
**Target Node:** `lab-linux-01` (`10.10.10.11`)  
**Operating System:** Ubuntu Server 22.04 LTS (Headless)  
**Date Completed:** 2026-10-10  

---

## 1. Scenario & Objectives

Enterprise systems require strict adherence to the **Principle of Least Privilege (PoLP)** and **Role-Based Access Control (RBAC)**. System administrators must avoid using the root account directly for routine development and service operations.

* **Business Objective:** Onboard an engineering team member (`bob`), associate them with an engineering security group (`devteam`), establish an isolated project directory with strict POSIX permissions, and delegate elevated `sudo` rights without endangering system stability.
* **Technical Scope:**
  1. Provision user `bob` with a home directory and default `/bin/bash` shell.
  2. Provision group `devteam` and append `bob` to secondary group membership.
  3. Deploy `/srv/projects` with octal permission mode `750` (`rwxr-x---`) owned by `bob:devteam`.
  4. Implement modular group `sudo` privileges via `/etc/sudoers.d/devteam` validated with `visudo -c`.

---

## 2. Pre-Configuration State

* **Target IP Address:** `10.10.10.11/24` (Interface `enp0s3`)
* **Remote Management Port:** Host `127.0.0.1:2222` forwarded to Guest `10.10.10.11:22`
* **Administrative Session:** Authenticated as primary local admin (`sysadmin`) over OpenSSH.

---

## 3. Implementation Steps & Commands

### 3.1 User & Group Provisioning
```bash
# 1. Create standard user bob with home directory (-m) and bash shell (-s /bin/bash)
sudo useradd -m -s /bin/bash bob

# 2. Assign password credential to activate account in /etc/shadow
sudo passwd bob

# 3. Create engineering security group
sudo groupadd devteam

# 4. Append bob to devteam group (-a = append, -G = secondary group)
sudo usermod -aG devteam bob
```

### 3.2 Directory Access & Octal Permissions
```bash
# Create shared project root
sudo mkdir -p /srv/projects

# Assign ownership to user 'bob' and group 'devteam'
sudo chown bob:devteam /srv/projects

# Set octal permissions: 7 (User: rwx) 5 (Group: r-x) 0 (Others: ---)
sudo chmod 750 /srv/projects
```

> **Permission Breakdown (`750`):**
> * **Owner (`bob` - `7`):** Read (`4`) + Write (`2`) + Execute/Traverse (`1`) = Full directory control.
> * **Group (`devteam` - `5`):** Read (`4`) + Execute/Traverse (`1`) = Members can list contents and enter directory, but cannot delete or modify existing files.
> * **Others (`World` - `0`):** No permissions (`0`). Prevents unauthorized local processes and standard users from traversing into `/srv/projects`.

### 3.3 Safe Sudoers Delegation via `/etc/sudoers.d/`
```bash
# Delegate full administrative privilege to all devteam members
echo "%devteam ALL=(ALL:ALL) ALL" | sudo tee /etc/sudoers.d/devteam

# Enforce mandatory 0440 read-only permissions for sudoers configuration
sudo chmod 0440 /etc/sudoers.d/devteam

# Perform sanity validation across all sudoers files
sudo visudo -c
```

> **Enterprise Best Practice:**  
> Editing `/etc/sudoers` directly via `nano` or `vim` carries the risk of leaving malformed syntax that invalidates the entire `sudo` binary configuration, immediately locking all non-root users out of privilege escalation. Using modular drop-in files in `/etc/sudoers.d/` validated by `visudo -c` protects against administrative lockouts.

---

## 4. Verification & Testing

### 4.1 Group Membership Verification
```bash
sysadmin@lab-linux-01:~$ id bob
uid=1001(bob) gid=1001(bob) groups=1001(bob),1002(devteam)
```

### 4.2 Sudoers Syntax Validation
```bash
sysadmin@lab-linux-01:~$ sudo visudo -c
/etc/sudoers: parsed OK
```

### 4.3 Interactive Privilege Escalation Verification
Logged into an interactive shell session as `bob` and verified root delegation:
```bash
sysadmin@lab-linux-01:~$ su - bob
Password:
bob@lab-linux-01:~$ sudo whoami
[sudo: authenticate] Password:
root
```

### 4.4 Directory Permissions & File Creation Test
Verified POSIX attributes on `/srv/projects` and tested file creation:
```bash
bob@lab-linux-01:~$ ls -ld /srv/projects
drwxr-x--- 2 bob devteam 4096 Oct 10 11:24 /srv/projects

bob@lab-linux-01:~$ touch /srv/projects/test.txt
bob@lab-linux-01:~$ ls -l /srv/projects
total 0
-rw-rw-r-- 1 bob bob 0 Oct 10 12:02 test.txt
```

---

## 5. Checkpoint Summary

- [x] Standard user `bob` provisioned with bash login shell and initialized `/home/bob`.
- [x] Engineering security group `devteam` created and populated.
- [x] Octal permissions (`750`) enforced on `/srv/projects` with `bob:devteam` ownership.
- [x] Privilege delegation file `/etc/sudoers.d/devteam` deployed and verified error-free.
- [x] Full interactive root escalation verified via `sudo whoami`.
