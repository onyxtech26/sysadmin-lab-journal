# Lab Step 2.7: Workstation Provisioning & Active Directory Domain Join

**Target Node:** `lab-client-01`  
**Domain:** `lab.local`  
**Operating System:** Windows Server 2022 Standard (Server Core)  
**Role:** Domain Member Workstation / Client Endpoint  

---

## 1. Scenario & Objectives

In an enterprise identity infrastructure, client workstations must be securely onboarded to the Active Directory domain to enable centralized authentication, Kerberos ticket granting, Group Policy enforcement, and access to shared network resources.

In this step, a dedicated Windows client node (`lab-client-01`) is provisioned on `LabNet`, configured for headless remote management via OpenSSH, and joined to the `lab.local` forest root domain via PowerShell.

### Technical Goals:
1. Deploy a lightweight Windows Server Core virtual machine (`lab-client-01`) attached to `LabNet` (`10.10.10.0/24`).
2. Configure IP addressing and point primary DNS resolution to `DC01` (`10.10.10.10`).
3. Deploy and enable OpenSSH Server on the client node and establish host port forwarding (`127.0.0.1:2224 -> Guest:22`).
4. Validate DNS SRV locator resolution for `lab.local`.
5. Join the client machine to Active Directory using `Add-Computer` and commit via automated reboot.
6. Verify domain membership from both the client endpoint and the Domain Controller directory database.

---

## 2. Infrastructure Profile & Remote Access Setup

| Parameter | Allocated Specification | Notes |
| :--- | :--- | :--- |
| **VM Name** | `lab-client-01` | VirtualBox inventory identifier |
| **Operating System** | Windows Server 2022 Standard (Server Core) | Lightweight non-GUI enterprise footprint (~1.2 GB idle RAM) |
| **Assigned IP** | `10.10.10.11/24` (or `10.10.10.20/24`) | Subnet `10.10.10.0/24` |
| **Default Gateway** | `10.10.10.1` | VirtualBox NAT Network Gateway |
| **Primary DNS** | `10.10.10.10` | Resolves AD DS integrated zone on `DC01` |
| **Host SSH Port** | `2224` | Port forwarded: `127.0.0.1:2224 -> Guest:22` |

### 2.1 OpenSSH Deployment on Client
To maintain uniform, headless administration from the physical host workstation across all three nodes:

```powershell
# Install OpenSSH Server capability
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Start and set sshd service to automatic startup
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
```

### 2.2 Hypervisor Port Forwarding Configuration
Executed on the physical host via `VBoxManage`:
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" natnetwork modify --netname "LabNet" --port-forward-4 "SSH-Windows-Client:tcp:[127.0.0.1]:2224:[10.10.10.11]:22"
```

---

## 3. Domain Join Implementation

Executed within PowerShell on `lab-client-01`:

### 3.1 Pre-Join DNS Verification
Before attempting a domain join, endpoint connectivity to AD DNS must be confirmed:
```powershell
Resolve-DnsName lab.local
```
*Validated: Successfully resolved to Domain Controller IP `10.10.10.10`.*

### 3.2 Execute Active Directory Domain Join
```powershell
Add-Computer -DomainName "lab.local" -Credential LAB\Administrator -Restart
```

> **Execution Mechanics:**
> 1. Contacts `10.10.10.10` via LDAP (`TCP 389`) and Kerberos (`TCP 88`).
> 2. Authenticates administrative privileges using domain credential `LAB\Administrator`.
> 3. Creates the Computer object in the default `CN=Computers,DC=lab,DC=local` container.
> 4. Establishes the Netlogon Secure Channel shared secret.
> 5. Triggers an automated system restart to commit the domain security boundary.

---

## 4. Live Verification & Directory Inspection

### 4.1 Client-Side Membership Verification (`lab-client-01`)
```powershell
(Get-CimInstance Win32_ComputerSystem).Domain
(Get-CimInstance Win32_ComputerSystem).PartOfDomain
```

*Live Console Output:*
```text
PS C:\Users\Administrator> (get-ciminstance win32_computersystem).domain
lab.local

PS C:\Users\Administrator> (get-ciminstance win32_computersystem).partofdomain
True
```

### 4.2 Network & Identity Verification (`lab-client-01`)
```powershell
whoami
(Get-NetIPAddress -InterfaceAlias "Ethernet" -AddressFamily IPv4).IPAddress
```

*Live Console Output:*
```text
PS C:\Users\Administrator> whoami
lab-client-01\administrator

PS C:\Users\Administrator> (get-netipaddress -interfacealias "ethernet" -addressfamily IPv4).ipaddress
10.10.10.11
```

### 4.3 Domain Controller Directory Verification (`DC01`)
Executed in PowerShell on `DC01`:
```powershell
Get-ADComputer -Identity "lab-client-01" | Select-Object Name, DNSHostName, Enabled
```

*Live Console Output:*
```text
Name          DNSHostName              Enabled
----          -----------              -------
LAB-CLIENT-01 lab-client-01.lab.local     True
```

### 4.4 Host-to-Client SSH Verification
Tested listener response from host machine terminal:
```powershell
Test-NetConnection -ComputerName 127.0.0.1 -Port 2224
```

*Live Host Terminal Output:*
```text
ComputerName     : 127.0.0.1
RemoteAddress    : 127.0.0.1
RemotePort       : 2224
TcpTestSucceeded : True
```

---

## 5. Architectural & Enterprise Security Insights

1. **The DC Locator Process**:
   * Joining a client to an Active Directory domain relies on resolving DNS SRV records (`_ldap._tcp.dc._msdcs.lab.local`). The client queries its configured DNS server (`10.10.10.10`), locates the nearest DC holding the PDC Emulator role, and initiates LDAP/SMB communication.
2. **Machine Accounts & Secure Channels**:
   * When `lab-client-01` joins `lab.local`, Active Directory generates a unique security principal (`LAB-CLIENT-01$`) in `CN=Computers`. The domain controller and client establish a shared secret (machine password) which is automatically rotated every 30 days to protect the Kerberos secure channel.
3. **Headless Tri-Node Matrix**:
   * With `lab-linux-01` (Port 2222), `DC01` (Port 2223), and `lab-client-01` (Port 2224) all running OpenSSH over NAT forwarding, the administrator can manage Linux, Windows Server Core DC, and Windows client nodes simultaneously from independent host terminal tabs without opening hypervisor graphical consoles.

---

## 6. Checkpoint Summary

- [x] Windows Server 2022 Server Core client VM (`lab-client-01`) provisioned on `LabNet`.
- [x] OpenSSH Server capability installed and verified active on port `2224`.
- [x] DNS resolution to `lab.local` validated against `DC01`.
- [x] Client joined to `lab.local` domain and committed via restart.
- [x] Computer object `LAB-CLIENT-01` confirmed in Active Directory on `DC01`.
- [x] Ready for Step 2.8 (Helpdesk Ticket Simulation: onboarding, group membership, share mapping).
