# Lab Step 2.6: Group Policy Object (GPO) Deployment & OU Linkage

**Target Node:** `DC01` (`10.10.10.10`)  
**Domain:** `lab.local`  
**Role:** Centralized Configuration Management, Policy Inheritance & Enforcement  

---

## 1. Scenario & Objectives

In enterprise environments, system policies, security baselines, and user desktop configurations must be managed uniformly across hundreds or thousands of nodes. Active Directory delivers this through **Group Policy Objects (GPOs)**.

Having established departmental Organizational Units (`Sales`, `IT`, `Finance`) in Step 2.5, this step establishes the Group Policy infrastructure on `DC01` using Server Core command-line tooling, provisions a departmental policy object (`Sales-Password-Policy`), and links it directly to the target Organizational Unit (`OU=Sales`).

### Technical Goals:
1. Deploy the Group Policy Management Console (`GPMC`) binaries and PowerShell provider to Windows Server 2022 Server Core.
2. Provision a new GPO: `Sales-Password-Policy`.
3. Link the policy directly to `OU=Sales,DC=lab,DC=local`.
4. Inspect the directory for Group Policy Container (GPC) registration, GUID assignment, and link precedence order.

---

## 2. Group Policy Architecture & Storage Model

A Group Policy Object is not a monolithic file; it consists of two distinct components synchronized across Active Directory:

```text
Group Policy Object (GPO)
├── Group Policy Container (GPC)
│   └── Stored in Active Directory (LDAP):
│       CN={a68ced0f-ea4f-478f-9a05-7f86cc67c80a},CN=Policies,CN=System,DC=lab,DC=local
│       Contains: GPO attributes, version numbers, status, extension list.
└── Group Policy Template (GPT)
    └── Stored in SYSVOL (File System / DFS-R):
        \\lab.local\sysvol\lab.local\Policies\{a68ced0f-ea4f-478f-9a05-7f86cc67c80a}\
        Contains: Registry settings (.pol files), scripts, security templates, folder redirection rules.
```

### Inheritance & Application Hierarchy (LSDOU):
Group policies are processed in a deterministic order by client endpoints:
1. **L**ocal Policy
2. **S**ite Policy
3. **D**omain Policy
4. **OU** Policy (Parent OUs down to child OUs)

*Later policies overwrite earlier settings unless "Enforced" (No Override) or "Block Inheritance" is explicitly configured.*

---

## 3. Implementation Steps & Commands

Executed within the remote administrative PowerShell session on `DC01`:

### 3.1 Install Group Policy Management Tools
```powershell
Install-WindowsFeature GPMC
```
> **Component Note:** Installs the `GroupPolicy` module, enabling cmdlets such as `New-GPO`, `New-GPLink`, `Get-GPO`, and `Get-GPInheritance` on Server Core.

### 3.2 Provision & Link the GPO in a Single Pipeline
```powershell
New-GPO -Name "Sales-Password-Policy" | New-GPLink -Target "OU=Sales,DC=lab,DC=local"
```
> **Pipeline Mechanics:** `New-GPO` instantiates the policy object in the directory and outputs a `Microsoft.GroupPolicy.Gpo` object. Piping (`|`) directly into `New-GPLink` binds the newly generated GUID to the target OU container immediately without intermediate scripting variables.

---

## 4. Live Verification & Terminal Output

### 4.1 Verify GPO Registration & GUID
```powershell
Get-GPO -Name "Sales-Password-Policy" | Select-Object DisplayName, Id, GpoStatus
```

*Live Console Output:*
```text
DisplayName           Id                                   GpoStatus
-----------           --                                   ---------
sales-password-policy a68ced0f-ea4f-478f-9a05-7f86cc67c80a AllSettingsEnabled
```

### 4.2 Verify OU Linkage & Evaluation Precedence
```powershell
(Get-GPInheritance -Target "OU=Sales,DC=lab,DC=local").GpoLinks | Select-Object DisplayName, Enabled, Enforced, Order
```

*Live Console Output:*
```text
DisplayName           Enabled Enforced Order
-----------           ------- -------- -----
sales-password-policy    True    False     1
```

*Evaluation Confirmed:*
* **DisplayName:** `sales-password-policy`
* **Enabled:** `True` (Policy is active and parsed by endpoints during GP processing cycles)
* **Enforced:** `False` (Standard inheritance applies; downstream sub-OUs can override unless marked Enforced)
* **Order:** `1` (Precedence 1 - highest priority among linked GPOs on this container)

---

## 5. Architectural & Enterprise Security Insights

1. **GPO Password Policies vs. Fine-Grained Password Policies (FGPP)**:
   * **The Traditional Constraint:** Traditional domain password policies configured inside standard GPOs (*Computer Configuration -> Windows Settings -> Security Settings -> Account Policies -> Password Policy*) only take effect domain-wide when applied at the **root domain level** (typically inside the *Default Domain Policy*). If linked to an OU, standard GPO password policies only affect local SAM accounts of computers residing in that OU, not domain user accounts.
   * **The Modern Enterprise Standard:** To enforce distinct, departmental password policies (e.g., stricter complexity and shorter expiration for privileged teams), Active Directory uses **Fine-Grained Password Policies (FGPP)** implemented through **Password Settings Objects (PSOs)** via `New-ADFineGrainedPasswordPolicy`.
   * **Departmental GPO Role:** GPOs linked to departmental OUs primarily control user/computer workstation security baselines (e.g., AppLocker rules, BitLocker recovery escrow, screen lockout timers, mapped drives, and Windows Defender settings).

2. **Headless Server Core vs. RSAT Management**:
   * While GPO creation, linking, and status inspection are performed cleanly via PowerShell on Server Core, configuring deep, multi-branch registry settings is traditionally managed using the graphical **Remote Server Administration Tools (RSAT)** Group Policy Management Console (`gpmc.msc`) connected remotely from an administrative workstation or jump box.

---

## 6. Checkpoint Summary

- [x] Group Policy Management Console (`GPMC`) feature installed on `DC01`.
- [x] Custom GPO `Sales-Password-Policy` created and assigned unique GUID (`a68ced0f-ea4f-478f-9a05-7f86cc67c80a`).
- [x] GPO linked to `OU=Sales,DC=lab,DC=local` with Precedence Order `1` and active status.
- [x] Group Policy inheritance validated via `Get-GPInheritance`.
- [x] Ready for Step 2.7 (Domain-joining a Windows client VM).
