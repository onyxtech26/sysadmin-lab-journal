# Lab Step 2.8: Helpdesk Service Request & Ticket Simulation (Capstone)

**Target Nodes:** `DC01` (`10.10.10.10`) & `lab-client-01` (`10.10.10.11`)  
**Domain:** `lab.local`  
**Identity:** `LAB\jdoe` (Jane Doe, Sales Representative)  
**Role:** End-to-End User Provisioning, Access Delegation & Resource Verification  

---

## 1. Scenario & Service Ticket Description

This capstone exercise simulates a realistic enterprise Tier 1/2 Systems Administration onboarding ticket:

> **Service Ticket #HD-8921:**  
> **Request:** "New hire Jane Doe (`jdoe`) has joined the Sales department. Provision her user identity, assign her to the departmental security group (`Sales-Team`), confirm that departmental Group Policy baseline applies upon login to `lab-client-01`, and map the departmental network share (`\\DC01\Sales` as `S:`) with appropriate write permissions."

### Technical Objectives:
1. Validate Jane Doe's directory identity and security group membership (`Sales-Team`).
2. Provision an enterprise SMB file share (`\\DC01\Sales`) backed by explicit NTFS and Share-level access controls.
3. Grant Jane Doe remote terminal login privileges on `lab-client-01` via local `Remote Management Users`.
4. Authenticate as `LAB\jdoe` over headless SSH into `lab-client-01`.
5. Validate Group Policy evaluation using scoped Resultant Set of Policy (`gpresult`).
6. Map persistent network drive `S:` pointing to `\\DC01\Sales` and verify read/write authorization.

---

## 2. Shared Storage Provisioning (`DC01`)

Executed within PowerShell on `DC01`:

### 2.1 File System & NTFS Access Control List (ACL)
```powershell
# Create storage directory
New-Item -Path "C:\Shares\Sales" -ItemType Directory -Force

# Grant NTFS Modify access to the Sales-Team security group
icacls "C:\Shares\Sales" /grant "LAB\Sales-Team:(OI)(CI)M"
```

*Live Verification Output:*
```text
C:\Shares\Sales LAB\Sales-Team:(OI)(CI)(M)
                NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                BUILTIN\Administrators:(I)(OI)(CI)(F)
                BUILTIN\Users:(I)(OI)(CI)(RX)
```

### 2.2 Provision SMB Network Share
```powershell
$shareParams = @{
    Name         = "Sales"
    Path         = "C:\Shares\Sales"
    FullAccess   = "Administrators"
    ChangeAccess = "LAB\Sales-Team"
}
New-SmbShare @shareParams

# Align share access to allow Authenticated Users through to the NTFS security layer
Grant-SmbShareAccess -Name "Sales" -AccountName "Authenticated Users" -AccessRight Full -Force
```

*Live Verification Output:*
```text
Name  ScopeName AccountName                      AccessControlType AccessRight
----  --------- -----------                      ----------------- -----------
Sales *         BUILTIN\Administrators           Allow             Full
Sales *         LAB\Sales-Team                   Allow             Change
Sales *         NT AUTHORITY\Authenticated Users Allow             Full
```

---

## 3. Client Access & User Authentication (`lab-client-01`)

### 3.1 Delegate Remote Management Access
On `lab-client-01`, standard domain users cannot establish remote interactive PowerShell/SSH sessions by default. The local security group was updated by the local administrator:
```powershell
Add-LocalGroupMember -Group "Remote Management Users" -Member "LAB\jdoe"
```

### 3.2 Authenticate as Jane Doe over SSH
From the physical host workstation terminal:
```powershell
ssh -p 2224 "LAB\jdoe@127.0.0.1"
```

*Live Identity Verification:*
```text
PS C:\Users\jdoe> whoami
lab\jdoe
```

---

## 4. Drive Mapping & Policy Validation (`lab-client-01`)

Executed within Jane Doe's active session:

### 4.1 Map Departmental Network Drive (`S:`)
```powershell
New-PSDrive -Name "S" -PSProvider FileSystem -Root "\\DC01\Sales" -Persist
```

### 4.2 Verify Write and Read Operations
```powershell
"Onboarding ticket simulation successful - Jane Doe" | Out-File "S:\welcome.txt"
Get-Content "S:\welcome.txt"
```

*Live Console Output:*
```text
PS C:\Users\jdoe> "Onboarding ticket simulation successful - Jane Doe" | Out-File "S:\welcome.txt"
PS C:\Users\jdoe> Get-Content "S:\welcome.txt"
Onboarding ticket simulation successful - Jane Doe
```

### 4.3 Group Policy Resultant Set of Policy Evaluation
In an OpenSSH remote session (Network Logon Type 3), non-administrative users are barred from querying the WMI RSOP provider (`ROOT\RSOP\User`). In enterprise operations, the administrator evaluates the user's applied policies from the elevated administrative context:
```cmd
gpresult /r /user "LAB\jdoe"
```
*Validated: Successfully enumerated applied GPOs under Jane Doe's domain user context (`sales-password-policy`, `Default Domain Policy`) without privilege escalation exceptions.*

---

## 5. Architectural Takeaways & Real-World Troubleshooting

1. **Effective Permissions = (Share Permissions ∩ NTFS Permissions)**:
   * Setting restrictive permissions at the SMB Share layer often causes unexpected session rejections before NTFS evaluation begins. Modern enterprise standard practice configures broad access at the share layer (`Authenticated Users: Full Control`), relying entirely on the granular NTFS file system ACLs (`LAB\Sales-Team: Modify`) to enforce the principle of least privilege.
2. **`gpresult` Remote Session & Privilege Boundary**:
   * Running plain `gpresult /r` as a standard non-admin domain user queries both Computer and User scopes, triggering an immediate `Access Denied` error when touching machine WMI/registry keys. Furthermore, over remote OpenSSH sessions (Network Logon Type 3), WMI access is restricted to Administrators. Running `gpresult /r /user "LAB\jdoe"` from the administrator session allows clean verification of the employee's policy state.
3. **Headless Enterprise Workflow Complete**:
   * The complete lifecycle—from bare-metal installation of Windows Server Core, forest creation, DHCP scoping, OU design, GPO deployment, and client onboarding—was executed and verified 100% headless over PowerShell and OpenSSH.

---

## 6. Checkpoint Summary

- [x] Departmental SMB share `\\DC01\Sales` deployed with RBAC permissions.
- [x] Remote access privileges delegated to `LAB\jdoe`.
- [x] Jane Doe authenticated into `lab-client-01` over SSH.
- [x] Persistent network drive `S:` mapped to `\\DC01\Sales`.
- [x] Read/write data verification confirmed (`welcome.txt`).
- [x] **Phase 2 (Active Directory Domain Services) 100% Complete!**
