# Phase 1: Linux Server Deployment & Network Foundations

**Node:** `lab-linux-01`  
**Operating System:** Ubuntu Server LTS (Headless)  
**Assigned Static IP:** `10.10.10.11/24`  
**Gateway:** `10.10.10.1`  
**DNS:** `8.8.8.8` (Public upstream DNS prior to local DC deployment)  

---

## 1. Objective & Scenario
Deploy the primary Linux server node within `LabNet`. This node serves as:
1. The target for Linux administration, user privilege management, and daemon management.
2. An independent endpoint to validate inter-node network connectivity against the Windows Domain Controller.
3. A headless server managed exclusively via SSH from remote workstations.

---

## 2. Virtual Hardware Profile

| Parameter | Allocated Value | Rationale |
| :--- | :--- | :--- |
| **VM Name** | `lab-linux-01` | Adheres to enterprise naming convention (`lab-<os>-<id>`) |
| **OS Type / Version** | Linux / Ubuntu (64-bit) | Standard LTS enterprise distribution |
| **vCPU** | 2 Processors | Adequate for service concurrency |
| **RAM** | 2048 MB (2 GB) | Minimal memory footprint for headless server |
| **Storage** | 25 GB Dynamic VDI | Thin-provisioned disk to conserve physical host storage |
| **Network Adapter 1** | NAT Network (`LabNet`) | Connected to shared virtual switch `10.10.10.0/24` |

---

## 3. Implementation Steps

### 3.1 Headless OS Installation
1. Booted from the Ubuntu Server ISO.
2. Completed the installer with minimal package profile.
3. **Critical Step:** Selected **OpenSSH Server** during the installation snaps/packages screen to ensure remote management availability without needing guest desktop GUI tools.
4. Set local administrative username and disabled unnecessary server packages.

### 3.2 Static IP Configuration via Netplan
Ubuntu Server manages network interfaces using YAML-based Netplan configs. By default, the installer assigns an ephemeral DHCP lease. For server infrastructure predictability, a static IP was defined.

Located the Netplan configuration at `/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [10.10.10.11/24]
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses: [8.8.8.8]
  version: 2
```

Applied the configuration:
```bash
sudo netplan apply
```

---

## 4. Verification & Testing

### 4.1 Interface Address Verification
```bash
ip a show dev enp0s3
```
*Expected Output:*
```text
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 10.10.10.11/24 brd 10.10.10.255 scope global enp0s3
       valid_lft forever preferred_lft forever
```

### 4.2 Gateway Routing & DNS Check
```bash
# Test default route reachability
ping -c 3 10.10.10.1

# Test upstream external DNS resolution and ICMP
ping -c 3 8.8.8.8
```

### 4.3 Remote SSH Connection (from Host)
Verified direct administration from the host machine terminal:
```bash
ssh <username>@10.10.10.11
```
Successful authentication established remote headless access, removing the need to interact with the VirtualBox graphical window.

---

## 5. Checkpoint Summary
- [x] Ubuntu Server VM deployed with dynamic disk allocation.
- [x] Persistent static IP assigned at `10.10.10.11/24`.
- [x] Routing through virtual gateway `10.10.10.1` verified.
- [x] OpenSSH daemon active and responding.
