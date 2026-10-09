# Lab Step 2.4: Windows DHCP Server Installation & Lab Subnet Scoping

**Target Node:** `DC01` (`10.10.10.10`)  
**Operating System:** Windows Server 2022 Standard (Server Core)  
**Role:** DHCP Server, Domain Controller, DNS Server  
**Scope Network:** `10.10.10.0/24` (`LabScope`)  
**Address Pool:** `10.10.10.100` – `10.10.10.200`  

---

## 1. Scenario & Objectives
Deploy and configure the **Microsoft DHCP Server** role on `DC01` to automate dynamic IPv4 addressing for future workstations and lab client virtual machines.

### Technical Goals:
1. Install the `DHCP` server feature and RSAT administrative modules via PowerShell.
2. Define the active IPv4 scope (`LabScope`) carving out `10.10.10.100`–`10.10.10.200` on the `10.10.10.0/24` subnet.
3. Configure DHCP scope options:
   * **Option 003 (Router / Default Gateway):** `10.10.10.1`
   * **Option 006 (DNS Servers):** `10.10.10.10` (Directing all domain lookups to `DC01`)
4. Authorize DHCP in Active Directory and initialize security groups (`netsh dhcp add securitygroups`).

---

## 2. Implementation Steps & Commands

Executed within the remote PowerShell SSH session on `DC01`:

### 2.1 Feature Installation
```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

### 2.2 Create IPv4 Scope (`LabScope`)
```powershell
Add-DhcpServerv4Scope -Name "LabScope" -StartRange 10.10.10.100 -EndRange 10.10.10.200 -SubnetMask 255.255.255.0 -State Active
```

> **Subnet Allocation Design:**
> * `10.10.10.1`: VirtualBox NAT Network Gateway.
> * `10.10.10.2` – `10.10.10.99`: Reserved for statically assigned servers and infrastructure nodes (e.g., `DC01` on `.10`, `lab-linux-01` on `.11`).
> * `10.10.10.100` – `10.10.10.200`: Dynamic pool allocated via DHCP for clients and test nodes.
> * `10.10.10.201` – `10.10.10.254`: Reserved for future expansion, VIPs, or secondary hypervisors.

### 2.3 Configure DHCP Scope Options
```powershell
Set-DhcpServerv4OptionValue -ScopeId 10.10.10.0 -Router 10.10.10.1 -DnsServer 10.10.10.10
```

### 2.4 Authorize and Restart Service
In an Active Directory domain, un-authorized DHCP servers will be prevented from serving addresses to prevent rogue DHCP spoofing.
```powershell
netsh dhcp add securitygroups
Restart-Service dhcpserver
```

---

## 3. Verification & Live Output

### 3.1 Verify IPv4 Scope State
```powershell
Get-DhcpServerv4Scope
```
*Live Console Output:*
```text
ScopeId         SubnetMask      Name           State    StartRange      EndRange        LeaseDuration
-------         ----------      ----           -----    ----------      --------        -------------
10.10.10.0      255.255.255.0   LabScope       Active   10.10.10.100    10.10.10.200    8.00:00:00
```

### 3.2 Verify Scope Option Values
```powershell
Get-DhcpServerv4OptionValue -ScopeId 10.10.10.0
```
*Live Console Output:*
```text
OptionId   Name            Type       Value                VendorClass     UserClass       PolicyName
--------   ----            ----       -----                -----------     ---------       ----------
3          Router          IPv4Add... {10.10.10.1}
6          DNS Servers     IPv4Add... {10.10.10.10}
51         Lease           DWord      {691200}
```

### 3.3 Verify Daemon Operational State
```powershell
Get-Service dhcpserver
```
*Live Console Output:*
```text
Status   Name               DisplayName
------   ----               -----------
Running  dhcpserver         DHCP Server
```

---

## 4. Architectural & Security Takeaways

1. **Option 006 Criticality for Active Directory**:
   * Joining a client to an Active Directory domain relies on resolving DNS SRV records (`_ldap._tcp.dc._msdcs.lab.local`). If DHCP clients receive a public resolver (like `8.8.8.8`) instead of the domain controller (`10.10.10.10`), domain lookups and authentication will fail. Automating this via Option 6 guarantees seamless client onboarding.
2. **Static vs. Dynamic IP Segmentation**:
   * Carving out a specific range (`.100` to `.200`) ensures static servers (`.10`, `.11`) will never suffer IP address conflicts with ephemeral client leases.

---

## 5. Checkpoint Summary
- [x] DHCP Server role installed and running on Windows Server Core.
- [x] `LabScope` provisioned on `10.10.10.0/24` with 101 usable dynamic addresses.
- [x] Default Gateway (`10.10.10.1`) and Domain DNS (`10.10.10.10`) options pushed to all future lease holders.
- [x] Security groups authorized and service verified `Running`.
- [x] Ready for Step 2.5 (OU Design, Users, and Security Groups).
