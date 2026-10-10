# Lab Entry 3.5: Scheduled System Automation with Cron

**Phase:** Phase 3: Linux Systems Administration & Hardening  
**Target Node:** `lab-linux-01` (`10.10.10.11`)  
**Operating System:** Ubuntu Server 22.04 LTS (Headless)  
**Date Completed:** 2026-10-10  

---

## 1. Scenario & Objectives

Enterprise infrastructure requires automated, scheduled maintenance and monitoring tasks without manual sysadmin intervention. Linux utilizes the **`cron`** daemon (`crond` / `cron.service`) to parse user and system job tables and execute scripts at specified calendar intervals.

* **Business Objective:** Implement automated, recurring filesystem storage auditing for user `bob` to ensure capacity limits and consumption trends are logged nightly.
* **Technical Scope:**
  1. Construct a crontab entry adhering to the standard 5-field time specification (`0 1 * * *`).
  2. Install the schedule into `bob`'s user spool file (`/var/spool/cron/crontabs/bob`).
  3. Validate file generation, redirection semantics (`>>`), and POSIX home directory access boundaries.

---

## 2. Cron Time Specification Reference

```text
┌───────────── Minute (0 - 59)
│ ┌─────────── Hour (0 - 23)
│ │ ┌───────── Day of Month (1 - 31)
│ │ │ ┌─────── Month (1 - 12)
│ │ │ │ ┌───── Day of Week (0 - 6, Sunday = 0 or 7)
│ │ │ │ │
0 1 * * *  /bin/command >> /path/to/log
```

---

## 3. Implementation Steps & Commands

### 3.1 Install Crontab via File-Buffered Method
```bash
# 1. Buffer the cron definition into an ephemeral configuration file
echo "0 1 * * * df -h >> /home/bob/disk-usage.log" > /tmp/bobcron

# 2. Install the scheduled job directly into bob's crontab spool
sudo crontab -u bob /tmp/bobcron

# 3. Clean up the temporary definition file
rm /tmp/bobcron
```

> **Design Decision (Buffer File vs. Pipe):**  
> In interactive SSH sessions, piping multiline `echo` commands directly into `crontab -` can trigger line-wrapping errors where long file paths are split onto subsequent lines, causing the cron parser to throw `bad minute`. Writing to an ephemeral file (`/tmp/bobcron`) guarantees atomic, single-line delivery into the spool.

### 3.2 Spool Verification
```bash
# Verify the installed crontab for bob
sudo crontab -u bob -l
```

### 3.3 Manual Execution Test
```bash
# Execute the target command once in the context of user bob
sudo -u bob sh -c "df -h >> /home/bob/disk-usage.log"

# Inspect the generated telemetry log (requires elevated privilege from non-bob accounts)
sudo cat /home/bob/disk-usage.log
```

---

## 4. Verification & Testing

### 4.1 Spool Inspection
```text
sysadmin@lab-linux-01:~$ sudo crontab -u bob -l
0 1 * * * df -h >> /home/bob/disk-usage.log
```

### 4.2 Log Content Audit
```text
sysadmin@lab-linux-01:~$ sudo cat /home/bob/disk-usage.log
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              329M  4.1M  325M   2% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  5.0G  5.7G  47% /
tmpfs                              822M     0  822M   0% /dev/shm
none                               1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
tmpfs                              822M     0  822M   0% /tmp
none                               1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
/dev/sda2                          2.0G  100M  1.7G   6% /boot
none                               1.0M     0  1.0M   0% /run/credentials/systemd-networkd.service
none                               1.0M     0  1.0M   0% /run/credentials/getty@tty1.service
tmpfs                              165M  8.0K  165M   1% /run/user/1000
```

---

## 5. Security & Permission Takeaway

Attempting to read the log directly as standard administrative user `sysadmin`:
```bash
sysadmin@lab-linux-01:~$ cat /home/bob/disk-usage.log
cat: /home/bob/disk-usage.log: Permission denied
```
In Ubuntu Server, user home directories are provisioned with `750` (`drwxr-x---`) permissions owned by `bob:bob`. This establishes a strict multi-tenant security boundary preventing unauthorized local users from reading sensitive user output. Viewing or modifying files inside another user's home directory requires explicit `sudo` elevation.

---

## 6. Checkpoint Summary

- [x] Crontab schedule configured for nightly execution at 1:00 AM (`0 1 * * *`).
- [x] Spool verified under `/var/spool/cron/crontabs/bob`.
- [x] Manual execution successfully generated telemetry file `/home/bob/disk-usage.log`.
- [x] POSIX multi-user home directory boundaries verified.
