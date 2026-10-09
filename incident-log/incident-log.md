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
