# Lab Step 2.1 & 2.2: Windows Server Core Provisioning & Initial Configuration

**Target Node:** `DC01` (originally `lab-dc-01`)  
**Operating System:** Windows Server 2022 Standard Evaluation (Server Core)  
**Assigned Static IP:** `10.10.10.10/24`  
**Default Gateway:** `10.10.10.1`  
**Primary DNS:** `127.0.0.1`  

---

## 1. Objective & Scenario
Provision a dedicated virtual machine running Windows Server 2022 Server Core and perform baseline system provisioning: hostname normalization and static network configuration using the Server Configuration utility (`sconfig`).

---

## 2. Hardware Profile & Provisioning (Step 2.1)

| Parameter | Allocated Specification | Notes |
| :--- | :--- | :--- |
| **VM Name** | `lab-dc-01` | VirtualBox inventory identifier |
| **Operating System** | Windows Server 2022 (64-bit) | Standard Evaluation Edition |
| **Interface Mode** | **Server Core** | Selected non-GUI minimal footprint installation |
| **Memory (RAM)** | 4096 MB (4 GB) | Minimum recommended baseline for AD DS + DNS + DHCP |
| **Processors** | 2 vCPUs | Ensures responsive directory lookups |
| **Storage** | 40 GB Dynamic VDI | Thin-provisioned disk |
| **Network Adapter** | NAT Network (`LabNet`) | Connected to `10.10.10.0/24` |

### Installation Execution:
1. Mounted `windows-server-2022-evaluation.iso`.
2. Booted VM and initiated Windows Setup.
3. Selected: `Windows Server 2022 Standard Evaluation (Server Core)`.
4. Formatted the 40 GB virtual disk and completed base installation.
5. On initial boot, satisfied the mandatory prompt to set a complex local `Administrator` password.

---

## 3. Baseline System Configuration via `sconfig` (Step 2.2)

Upon initial authentication, Server Core presents a single command prompt (`cmd.exe`). Initial host configuration was performed using `sconfig`:

```cmd
sconfig
```

### 3.1 Hostname Renaming
* **Action:** Selected Option `2` (Computer Name).
* **New Name:** `DC01`.
* **Restart:** Executed immediate reboot when prompted to commit NetBIOS and hostname changes.

### 3.2 Static Network Addressing
Post-reboot, re-launched `sconfig` and navigated to Option `8` (Network Settings):
1. Selected the active virtual network adapter index.
2. Selected Option `1` (Set Network Adapter Address):
   * Setting: `S` (Static)
   * Static IP: `10.10.10.10`
   * Subnet Mask: `255.255.255.0` (`/24`)
   * Default Gateway: `10.10.10.1`
3. Selected Option `2` (Set DNS Servers):
   * Primary DNS: `127.0.0.1` (Points to loopback resolver in anticipation of the AD integrated DNS role)
4. Exited `sconfig` (Option `15`) back to the command prompt.

---

## 4. Verification & Validation

Switched from `cmd.exe` to PowerShell:
```cmd
powershell
```

### 4.1 Hostname Verification
```powershell
hostname
# Output: DC01
```

### 4.2 Network Adapter Verification
```powershell
Get-NetIPConfiguration -InterfaceAlias "Ethernet" | Select-Object InterfaceAlias, IPv4Address, IPv4DefaultGateway, DNSServer
```
*Expected Output:*
```text
InterfaceAlias     : Ethernet
IPv4Address        : {10.10.10.10}
IPv4DefaultGateway : {10.10.10.1}
DNSServer          : {127.0.0.1}
```

### 4.3 Default Gateway Reachability
```powershell
Test-Connection -ComputerName 10.10.10.1 -Count 2 -Quiet
# Output: True
```

---

## 5. Checkpoint Summary
- [x] Windows Server 2022 Server Core installed without desktop GUI bloat.
- [x] Hostname permanently updated to `DC01`.
- [x] Static IP configuration applied (`10.10.10.10/24`, Gateway: `10.10.10.1`).
- [x] Local loopback resolver configured (`127.0.0.1`) ready for AD DS promotion.
