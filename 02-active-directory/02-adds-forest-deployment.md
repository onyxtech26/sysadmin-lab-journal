# Lab Step 2.3: Active Directory Domain Services (AD DS) Installation & Forest Promotion

**Target Node:** `DC01` (`10.10.10.10`)  
**Operating System:** Windows Server 2022 Standard (Server Core)  
**Forest Root Domain:** `lab.local`  
**NetBIOS Domain Name:** `LAB`  
**Role:** Root Domain Controller, Global Catalog (GC), Primary DNS Server  

---

## 1. Scenario & Objectives
Promote `DC01` from a standalone workgroup server into the root Domain Controller for a newly provisioned Active Directory forest (`lab.local`).

### Technical Goals:
1. Install the `AD-Domain-Services` Windows role along with RSAT management binaries via PowerShell.
2. Promote the server and bootstrap a new Active Directory forest.
3. Automatically deploy and integrate Microsoft DNS for SRV record registration.
4. Establish the Directory Services Restore Mode (DSRM) administrative credential.

---

## 2. Pre-Promotion Verification Check
Before executing the role promotion, the following prerequisites were validated on `DC01`:
* **Static IP Binding:** `10.10.10.10/24` verified active on primary network interface.
* **DNS Resolver:** Configured to `127.0.0.1` (loopback).
* **Hostname:** Normalized to `DC01` and committed via reboot.

---

## 3. Implementation Steps & Commands

Entered PowerShell from the Server Core console prompt:
```cmd
powershell
```

### 3.1 Install AD DS Role & Management Tools
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

> **Parameter Rationale:**
> * `AD-Domain-Services`: Deploys the core directory database engine (`ntds.dit`) and directory binaries.
> * `-IncludeManagementTools`: Installs the Active Directory PowerShell module (`Microsoft.ActiveDirectory.Management`) and command-line administrative tools (`dcdiag`, `repadmin`, `ntdsutil`).

### 3.2 Forest Promotion & Domain Creation (PowerShell Splatting)
To eliminate terminal buffer whitespace splitting and parameter binding syntax errors over remote SSH, parameter **splatting** is utilized:

```powershell
# 1. Define DSRM recovery password as secure string
$secPass = ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force

# 2. Define parameters via hashtable (splatting)
$params = @{
    DomainName                    = "lab.local"
    SafeModeAdministratorPassword = $secPass
    Force                         = $true
}

# 3. Execute forest promotion
Install-ADDSForest @params
```

> **Enterprise Architecture Note (Default Role Behaviors):** 
> * **DNS Server:** In Windows Server 2022, creating a new forest root automatically installs and delegates Microsoft DNS by default.
> * **NetBIOS Name:** Automatically derives `LAB` from the leftmost label of `lab.local`.
> * Omitting explicit `-InstallDns` and `-DomainNetbiosName` parameters avoids known DCPromo prerequisite validation exceptions (`DCPromo.General.77`) in Server Core.

> **Parameter Breakdown:**
> * `-DomainName "lab.local"`: Defines the Fully Qualified Domain Name (FQDN) for the forest root.
> * `-DomainNetbiosName "LAB"`: Defines the legacy NetBIOS identifier used for downlevel authentication and user prefixes (`LAB\username`).
> * `-InstallDns:$true`: Automatically installs the DNS Server feature and configures the `lab.local` Active Directory-integrated forward lookup zone.
> * `-SafeModeAdministratorPassword`: Sets the Directory Services Restore Mode (DSRM) administrator password required for offline Active Directory database maintenance or authoritative restore operations.
> * `-Force:$true`: Suppresses interactive confirmation prompts, allowing headless scripted execution.

Upon completion of the command, the server automatically initiated a system restart to configure directory partitions and update security subsystem authorities.

---

## 4. Post-Reboot Verification & Health Checks

Logged back into `DC01` using the newly created domain credentials:
* **User:** `LAB\Administrator`
* **Password:** *(Domain Administrator Password)*

Executed the following PowerShell verification commands:

### 4.1 Domain & Forest Information
```powershell
Get-ADDomain
```
*Live Console Output:*
```text
AllowedDNSSuffixes                 : {}
ChildDomains                       : {}
ComputersContainer                 : CN=Computers,DC=lab,DC=local
DeletedObjectsContainer            : CN=Deleted Objects,DC=lab,DC=local
DistinguishedName                  : DC=lab,DC=local
DNSRoot                            : lab.local
DomainControllersContainer         : OU=Domain Controllers,DC=lab,DC=local
DomainMode                         : Windows2016Domain
DomainSID                          : S-1-5-21-3727344856-3208495336-3448980338
ForeignSecurityPrincipalsContainer : CN=ForeignSecurityPrincipals,DC=lab,DC=local
Forest                             : lab.local
InfrastructureMaster               : DC01.lab.local
LastLogonReplicationInterval       :
LinkedGroupPolicyObjects           : {CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=lab,DC=local}
LostAndFoundContainer              : CN=LostAndFound,DC=lab,DC=local
ManagedBy                          :
Name                               : lab
NetBIOSName                        : LAB
ObjectClass                        : domainDNS
ObjectGUID                         : f1e62338-0848-4118-8828-c0ebe308be16
ParentDomain                       :
PDCEmulator                        : DC01.lab.local
PublicKeyRequiredPasswordRolling   : True
QuotasContainer                    : CN=NTDS Quotas,DC=lab,DC=local
ReadOnlyReplicaDirectoryServers    : {}
ReplicaDirectoryServers            : {DC01.lab.local}
RIDMaster                          : DC01.lab.local
SubordinateReferences              : {DC=ForestDnsZones,DC=lab,DC=local, DC=DomainDnsZones,DC=lab,DC=local, CN=Configuration,DC=lab,DC=local}
SystemsContainer                   : CN=System,DC=lab,DC=local
UsersContainer                     : CN=Users,DC=lab,DC=local
```

```powershell
Get-ADForest
```
*Live Console Output:*
```text
ApplicationPartitions : {DC=ForestDnsZones,DC=lab,DC=local, DC=DomainDnsZones,DC=lab,DC=local}
CrossForestReferences : {}
DomainNamingMaster    : DC01.lab.local
Domains               : {lab.local}
ForestMode            : Windows2016Forest
GlobalCatalogs        : {DC01.lab.local}
Name                  : lab.local
PartitionsContainer   : CN=Partitions,CN=Configuration,DC=lab,DC=local
RootDomain            : lab.local
SchemaMaster          : DC01.lab.local
Sites                 : {Default-First-Site-Name}
SPNSuffixes           : {}
UPNSuffixes           : {}
```
*(All 5 FSMO roles confirmed active on `DC01.lab.local`).*

### 4.2 Active Directory Core Services Status
```powershell
Get-Service adws, dns, ntds, kdc | Select-Object Name, Status, StartType
```
*Expected Output:*
```text
Name Status  StartType
---- ------- ---------
adws Running Automatic  # Active Directory Web Services (PowerShell remoting/API)
dns  Running Automatic  # Microsoft DNS Server daemon
kdc  Running Automatic  # Kerberos Key Distribution Center
ntds Running Automatic  # Active Directory Domain Services (NT Directory Service)
```

### 4.3 DNS SRV Record Registration Check
Active Directory domain members locate domain controllers via DNS SRV records in the `_msdcs` zone:
```powershell
Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.lab.local" -Type SRV
```
*Expected Output:*
```text
Name                                Type TTL Section NameTarget  Port Priority Weight
----                                ---- --- ------- ----------  ---- -------- ------
_ldap._tcp.dc._msdcs.lab.local      SRV  600 Answer  DC01.lab.local 389  0        100   
```

### 4.4 DC Locator Test
```cmd
nltest /dclist:lab.local
```
*Expected Output:*
```text
Get list of DCs in domain 'lab.local' from '\\DC01.lab.local'.
    DC01.lab.local [PDC]
The command completed successfully
```

---

## 5. Key Architecture & Security Takeaways

1. **Integrated DNS vs External DNS**:
   * Active Directory strictly relies on dynamic DNS registrations for Kerberos, LDAP, and Global Catalog lookups. By coupling DNS directly with AD DS (`-InstallDns:$true`), directory replication and secure dynamic updates are natively enabled.
2. **DSRM Password Isolation**:
   * The DSRM password is decoupled from the `LAB\Administrator` domain account. It is stored in the local registry SAM hive on the DC and only activated during Directory Services Repair boot mode.
3. **FSMO Role Placement**:
   * As the first DC in a single-domain forest, `DC01` currently holds all 5 Flexible Single Master Operation (FSMO) roles: Schema Master, Domain Naming Master, PDC Emulator, RID Master, and Infrastructure Master.

---

## 6. Checkpoint Summary
- [x] AD DS role and administration modules installed on Windows Server 2022 Server Core.
- [x] New forest `lab.local` successfully provisioned.
- [x] Domain-integrated DNS initialized and publishing `_msdcs` SRV records.
- [x] Authenticated successfully as `LAB\Administrator`.
- [x] Domain Controller ready for Step 2.4 (DHCP Server deployment and scope configuration).
