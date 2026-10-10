# Incident Response & Troubleshooting Log

This log documents issues, configuration errors, and diagnostic steps encountered across the lab lifecycle, written in **STAR format** (Situation, Task, Action, Result). 

Technical interviewers frequently ask: *"Tell me about a time a system or service didn't work and how you diagnosed it."* This log serves as direct reference material for those discussions.

---

## Incident 01: Host Machine SSH Connection Timeout to `lab-linux-01`

* **Date:** Phase 1 (Step 1.4)
* **Impacted Node:** `lab-linux-01` (`10.10.10.11`)
* **Category:** Network Adapter Configuration / Hypervisor Routing

### Situation
After completing the Ubuntu Server installation and applying static IP `10.10.10.11` via Netplan on the `LabNet` NAT Network, attempting to connect directly from the Windows host terminal (`ssh user@10.10.10.11`) resulted in a connection timeout (`Destination Host Unreachable` / timeout).

### Task
Establish reliable host-to-guest SSH terminal access into both `lab-linux-01` and `DC01` without exposing the VMs to the local physical LAN (Bridged mode) or adding unnecessary secondary virtual adapters.

### Action
1. Analyzed hypervisor network topology: In VirtualBox, a **NAT Network** isolates the guest subnet from host routing. Packets originating from the physical host cannot reach `10.10.10.x` directly without host-side route tables or port translation.
2. Navigated to VirtualBox `Tools -> Network Manager -> NAT Networks -> LabNet -> Port Forwarding`.
3. Created explicit forwarding rules:
   * **Rule 1 (Linux):** Host Port `2222` -> Guest IP `10.10.10.11`, Guest Port `22`.
   * **Rule 2 (Windows Server):** Host IP `127.0.0.1`, Host Port `2223` -> Guest IP `10.10.10.10`, Guest Port `22`.
4. Tested listener response on host via PowerShell: `Test-NetConnection -ComputerName 127.0.0.1 -Port 2222` and `-Port 2223`.

### Result
Both guest virtual machines became instantly accessible from the host terminal using standard port switches:
* Linux: `ssh -p 2222 <username>@127.0.0.1`
* Windows DC: `ssh -p 2223 Administrator@127.0.0.1`
This preserved the isolated lab network boundary while delivering responsive, headless administration across both operating systems.

---

## Incident 02: Active Directory Installation Prerequisite Check Warning (Loopback DNS)

* **Date:** Phase 2 (Step 2.3)
* **Impacted Node:** `DC01` (`10.10.10.10`)
* **Category:** Domain Services / DNS Configuration

### Situation
During preliminary checks before running `Install-ADDSForest`, the network adapter had an upstream public DNS server (`8.8.8.8`) configured from the initial setup instead of local loopback.

### Task
Ensure that promoting the server to a Domain Controller does not cause split-brain DNS resolution or fail the Active Directory pre-promotion validation tests.

### Action
1. Launched `sconfig` -> Option `8` (Network Settings).
2. Pointed the primary DNS server address to `127.0.0.1`.
3. Executed `Install-ADDSForest` with the `-InstallDns:$true` parameter to ensure the integrated DNS service automatically provisions the forward lookup zone `lab.local` and `_msdcs` SRV records locally.

### Result
The AD DS forest installation completed without DNS delegation errors. Upon reboot, running `Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.lab.local" -Type SRV` verified that `DC01` successfully resolved its own Kerberos and LDAP service records.

---

## Incident 03: PowerShell Subexpression Parser Error on Remote SSH Session

* **Date:** Phase 2 (Step 2.3)
* **Impacted Node:** `DC01` (`10.10.10.10`)
* **Category:** Remote Terminal / PowerShell Syntax & Parser

### Situation
When pasting the multiline `Install-ADDSForest` cmdlet into an active SSH terminal session on the Windows host, the command failed with the following parser errors:
```text
At line:5 char:61
+       -SafeModeAdministratorPassword (ConvertTo-SecureString
+                                                             ~
Missing closing ')' in expression.
At line:6 char:37
+   "P@ssw0rd123!" -AsPlainText -Force) `
+                                     ~
Unexpected token ')' in expression or statement.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : MissingEndParenthesisInExpression
```

### Task
Successfully provide a secure `System.Security.SecureString` object to `-SafeModeAdministratorPassword` without encountering terminal buffer splits.

### Action
1. Analyzed root cause: Remote SSH terminal emulators often break lines at carriage returns during interactive multi-line buffer transfers. When an inline subexpression `(...)` spans across line breaks continued with backticks (`` ` ``), the parser attempts to evaluate `(ConvertTo-SecureString` prematurely before receiving the closing parenthesis on the subsequent line.
2. Refactored the command to decouple input object creation from cmdlet execution:
   ```powershell
   # Store the secure string into a standalone variable
   $secPass = ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force

   # Reference variable directly in single execution block
   Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns:$true -SafeModeAdministratorPassword $secPass -Force:$true
   ```

### Result
The cmdlet parsed cleanly without terminal buffer truncation. Active Directory forest provisioning commenced immediately. Documented this best practice for headless PowerShell remoting.

---

## Incident 04: ParameterBindingException on Remote Shell Paste Buffer

* **Date:** Phase 2 (Step 2.3)
* **Impacted Node:** `DC01` (`10.10.10.10`)
* **Category:** PowerShell Automation / Shell Parameter Binding

### Situation
Attempting to run a single-line command with multiple hyphenated switches (`-DomainName ... -DomainNetbiosName ...`) resulted in a `ParameterBindingException`:
```text
Install-ADDSForest : A positional parameter cannot be found that accepts argument '-'.
At line:1 char:5
+     Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LA ...
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidArgument: (:) [Install-ADDSForest], ParameterBindingException
    + FullyQualifiedErrorId : PositionalParameterNotFound,Microsoft.DirectoryServices.Deployment.PowerShell.Commands.InstallADDSForestCommand
```

### Task
Isolate why PowerShell registered a standalone `-` positional parameter and pass all configuration switches reliably into `Install-ADDSForest`.

### Action
1. Investigated root cause: When pasting long command strings into certain SSH/Windows pseudoconsole sessions, terminal word-wrap or encoding can introduce an errant space immediately following a hyphen (e.g., `- DomainName` or ` - Force`), causing PowerShell to interpret the leading dash as an unexpected positional argument rather than a parameter identifier prefix.
2. Refactored the command structure to use **PowerShell Splatting**:
   ```powershell
   $params = @{
       DomainName                    = "lab.local"
       DomainNetbiosName             = "LAB"
       InstallDns                    = $true
       SafeModeAdministratorPassword = $secPass
       Force                         = $true
   }
   Install-ADDSForest @params
   ```
   *By defining the parameters inside a hashtable, keys do not contain hyphens, completely removing the surface area for terminal paste syntax errors.*

### Result
The splatted hashtable passed all parameters into `Install-ADDSForest` without syntax anomalies. Documented parameter splatting as standard operating procedure for all subsequent multi-parameter cmdlets.

---

## Incident 05: DCPromo Prerequisite Failure on Explicit `DomainNetbiosName` (`DCPromo.General.77`)

* **Date:** Phase 2 (Step 2.3)
* **Impacted Node:** `DC01` (`10.10.10.10`)
* **Category:** Active Directory Domain Services / DCPromo Validation

### Situation
Executing `Install-ADDSForest` resulted in consecutive prerequisite check terminations:
```text
Install-ADDSForest : Verification of prerequisites for Domain Controller promotion failed. The specified argument 'DomainNetbiosName' was not recognized.
FullyQualifiedErrorId : Test.VerifyDcPromoCore.DCPromo.General.77

# Upon removing DomainNetbiosName:
Install-ADDSForest : Verification of prerequisites for Domain Controller promotion failed. The specified argument 'InstallDNS' was not recognized.
FullyQualifiedErrorId : Test.VerifyDcPromoCore.DCPromo.General.77
```

### Task
Isolate why the underlying promotion validator rejected explicit `DomainNetbiosName` and `InstallDNS` arguments, and successfully provision the `lab.local` forest with integrated DNS.

### Action
1. Researched the diagnostic ID `Test.VerifyDcPromoCore.DCPromo.General.77`. In Windows Server 2022, passing `-DomainNetbiosName` and `-InstallDns` explicitly into `Install-ADDSForest` can trigger argument-parsing mismatches between the PowerShell wrapper and underlying `dcpromo.dll` validation routines.
2. Analyzed Active Directory forest root default behaviors:
   * **DNS Role:** When building a brand-new forest, Microsoft DNS Server is installed and configured with the root zone automatically by default.
   * **NetBIOS Name:** When omitted, the promotion engine automatically extracts the leftmost label (`LAB`) from `lab.local` and validates it against NetBIOS standards (<= 15 characters, no illegal characters) automatically.
3. Streamlined the parameter hashtable to the strictly required parameters:
   ```powershell
   $params = @{
       DomainName                    = "lab.local"
       SafeModeAdministratorPassword = $secPass
       Force                         = $true
   }
   Install-ADDSForest @params
   ```

### Result
With redundant parameters removed, prerequisite validation passed cleanly without validation exceptions. Documented minimal parameter design for headless forest provisioning.

---

## Incident 06: Promotion Failure Due to Pre-Existing Domain Controller Role (`role: 5 NT5_DC`)

* **Date:** Phase 2 (Step 2.3)
* **Impacted Node:** `DC01` (`10.10.10.10`)
* **Category:** Active Directory Domain Services / Promotion State Machine

### Situation
Subsequent executions of `Install-ADDSForest` returned `Test.VerifyDcPromoCore.DCPromo.General.77` stating:
```text
Install-ADDSForest : Verification of prerequisites for Domain Controller promotion failed. The specified argument 'NewDomain' was not recognized.
```

### Task
Analyze internal deployment logging to isolate why the promotion engine rejected valid forest creation arguments.

### Action
1. Queried the last 25 lines of `%systemroot%\debug\dcpromoui.log`:
   ```text
   Enter Computer::GetRole DC01
     role: 5
     NT5_DC
   Enter State::GetRunContext NT5_DC
   Enter State::SetOperation DEMOTE
   ...
   Enter CArgumentsSpec::ValidateArgument NewDomain
   Error: The specified argument 'NewDomain' was not recognized.
   Exit code is 77
   ```
2. Diagnosed root cause:
   * `role: 5` corresponds to `DsRole_RolePrimaryDomainController` (`NT5_DC`).
   * `DC01` had already completed promotion to the forest root Domain Controller in a previous run.
   * Because the host was already a Domain Controller, DCPromo automatically changed operational context to **`DEMOTE`** (`State::SetOperation DEMOTE`).
   * When promotion arguments (`NewDomain`, `InstallDNS`, `DomainNetbiosName`) were passed to an engine in `DEMOTE` mode, DCPromo rejected them as invalid demotion parameters.
3. Verified Active Directory operational health via PowerShell:
   ```powershell
   Get-ADDomain
   Get-ADForest
   (Get-CimInstance Win32_ComputerSystem).DomainRole  # Returned 5 (Primary Domain Controller)
   ```

### Result
Confirmed `DC01` is active and healthy as the root Domain Controller for `lab.local`. Prerequisite check was not failing due to bad configuration, but because the forest was already fully operational. Advanced directly to Step 2.4 (DHCP Server deployment).

---

## Incident 07: Client VM Subnet Isolation Due to Default NAT vs NAT Network

* **Date:** Phase 2 (Step 2.7)
* **Impacted Node:** `lab-client-01`
* **Category:** Hypervisor Networking / Network Virtualization

### Situation
Upon deploying the Windows Server Core client VM, executing `Resolve-DnsName lab.local` failed with `DNS_ERROR_RCODE_NAME_ERROR` (`DNS name does not exist`). Running `Get-NetIPConfiguration` revealed the adapter had acquired IP `10.0.2.15`.

### Task
Connect `lab-client-01` to the shared `LabNet` NAT Network (`10.10.10.0/24`) without reinstalling or recreating the virtual machine.

### Action
1. Diagnosed root cause: VirtualBox defaults new virtual machine adapters to standalone **NAT** (isolated per-VM slirp engine assigning `10.0.2.15`), preventing communication with `DC01` (`10.10.10.10`).
2. Identified the client VM's hardware identifier via CLI:
   ```powershell
   & "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" list vms
   ```
3. Dynamically modified the active network attachment using VirtualBox CLI:
   ```powershell
   & "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" controlvm "908ffb08-4740-4d62-9a7a-2193436386bf" nic1 natnetwork LabNet
   ```
4. Reset the virtual machine (`Restart-Computer`) to initialize the guest socket against the `VBoxNetNAT` daemon.

### Result
`lab-client-01` successfully established Layer 2/3 connectivity on `LabNet`, resolved `DC01.lab.local`, and joined the Active Directory domain cleanly via `Add-Computer`.

---

## Incident 08: `gpresult` Privilege Boundary Exception on Standard Domain User Session

* **Date:** Phase 2 (Step 2.8)
* **Impacted Node:** `lab-client-01`
* **Category:** Group Policy / Security Privileges & RBAC

### Situation
While logged in as standard domain user `LAB\jdoe`, executing `gpresult /r` returned an immediate `ERROR: Access Denied`.

### Task
Inspect applied Group Policy Objects for the logged-in user without granting unnecessary local administrative privileges.

### Action
1. Investigated root cause: Executing `gpresult /r` without scope flags attempts to read both **Computer** and **User** policy settings. Computer policies query machine-level registry paths (`HKLM\Software\Policies`) and WMI namespaces (`ROOT\RSOP\Computer`), which strictly require local Administrator rights.
2. In addition, when connected over **OpenSSH**, Windows establishes a **Network Logon (Logon Type 3)**. Under remote network logons, non-administrative accounts are barred from remotely querying the WMI RSOP provider (`ROOT\RSOP\User`), resulting in `Access Denied` even with `/scope user`.
3. In enterprise administrative workflows, Group Policy verification for non-privileged accounts is evaluated directly from an administrative session targeting the user's security identifier:
   ```cmd
   gpresult /r /user "LAB\jdoe"
   ```

### Result
Evaluating `gpresult` from the elevated administrative context successfully parsed Jane Doe's applied policies, SIDs, and security group memberships, resolving the remote WMI privilege boundary without compromising the principle of least privilege.

---

## Incident 09: SMB Share-Level Authorization Denied Despite NTFS Modify Permissions

* **Date:** Phase 2 (Step 2.8)
* **Impacted Node:** `DC01` (`10.10.10.10`)
* **Category:** File Services / SMB vs NTFS Effective Permissions

### Situation
When Jane Doe (`LAB\jdoe`) attempted to write a verification file to the mapped network drive (`"..." | Out-File "S:\welcome.txt"`), PowerShell returned `OpenError: Access to the path 'S:\welcome.txt' is denied`, despite `C:\Shares\Sales` having explicit NTFS Modify permissions granted to `LAB\Sales-Team`.

### Task
Identify why the SMB layer rejected write requests and align share permissions with enterprise least-privilege standards.

### Action
1. Evaluated Windows effective permissions model:
   $$\text{Effective Permission} = \text{Share Permissions} \cap \text{NTFS Permissions}$$
2. Queried current share access on `DC01` (`Get-SmbShareAccess -Name "Sales"`). The share-level ACL was narrowly restricted to `LAB\Sales-Team (Change)` and `Administrators (Full)`, which rejected the incoming SMB session connection before NTFS evaluation took place.
3. Aligned the share with Microsoft enterprise best practices: Granted `Full Control` at the SMB Share layer to `Authenticated Users`, allowing incoming domain sessions to pass through to the underlying NTFS ACL where granular security is enforced:
   ```powershell
   Grant-SmbShareAccess -Name "Sales" -AccountName "Authenticated Users" -AccessRight Full -Force
   ```

### Result
Jane Doe successfully wrote `welcome.txt` to `S:\` and verified file integrity via `Get-Content`, confirming end-to-end RBAC functionality for the Sales department.

---

## Incident 10: `su: Authentication failure` on Newly Provisioned Linux User

* **Date:** Phase 3 (Step 3.1)
* **Impacted Node:** `lab-linux-01` (`10.10.10.11`)
* **Category:** Linux IAM / Account Lifecycle & Authentication

### Situation
Immediately after provisioning standard user account `bob` with `sudo useradd -m -s /bin/bash bob` and assigning group memberships, running `su - bob` resulted in an immediate `su: Authentication failure`.

### Task
Diagnose why authentication was rejected and complete account initialization so the user can log in and execute workloads.

### Action
1. Investigated Linux account creation behavior: The low-level `useradd` utility creates user entries in `/etc/passwd` and creates the home directory when `-m` is specified. However, it does not prompt for or assign a password hash.
2. Inspected `/etc/shadow` structure: When created without an initial password, `useradd` places an exclamation mark (`!`) in the password hash field. This explicitly locks the account, blocking all password-based authentication (`PAM` module `pam_unix.so` automatically rejects attempts).
3. Initialized credentials for the account:
   ```bash
   sudo passwd bob
   ```
   Entered a compliant password to replace the locked token in `/etc/shadow` with a salted SHA-512 crypt hash.

### Result
Account was unlocked immediately. Subsequent login attempts via `su - bob` authenticated cleanly, confirming the account lifecycle was properly finalized.

---

## Incident 11: `sudo: A terminal is required to authenticate` in Non-Interactive Execution

* **Date:** Phase 3 (Step 3.2)
* **Impacted Node:** `lab-linux-01` (`10.10.10.11`)
* **Category:** Linux Security / Sudo Privileges & Terminal Allocation (TTY)

### Situation
When attempting to verify Bob's newly delegated sudo rights from the admin terminal using `su - bob -c "sudo whoami"`, `sudo` aborted execution with the error:
```text
sudo: A terminal is required to authenticate
```

### Task
Safely verify Bob's administrative delegation without compromising system security controls or introducing insecure nopasswd sudoers rules.

### Action
1. Analyzed security mechanism: The `su` command's `-c` flag executes a single command string non-interactively in a child sub-shell without allocating a pseudo-terminal (TTY).
2. By default, `sudo` enforces interactive TTY allocation for password entry (`requiretty` or default PAM authentication) to protect against script-based shoulder-surfing or brute-force injection attacks. When `sudo` detected that standard input was not attached to a TTY, it immediately aborted to prevent password leakage.
3. Transitioned to an interactive user session:
   ```bash
   su - bob
   # Inside interactive session with allocated TTY:
   sudo whoami
   ```

### Result
`sudo` displayed the secure password prompt on the allocated TTY. Upon supplying credentials, `sudo whoami` returned `root`, confirming that `%devteam` sudoers delegation was operating correctly within security compliance boundaries.

