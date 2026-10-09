# Lab Step 2.5: Active Directory OU Hierarchy, Security Groups & User Provisioning

**Target Node:** `DC01` (`10.10.10.10`)  
**Domain:** `lab.local`  
**Role:** Identity & Access Management (IAM), Role-Based Access Control (RBAC)  

---

## 1. Scenario & Objectives
Design and implement a structured Active Directory directory schema that reflects an enterprise departmental hierarchy. Rather than placing all objects into default flat containers (`CN=Users`, `CN=Computers`), objects are organized into dedicated **Organizational Units (OUs)** to enable granular Group Policy inheritance and administrative delegation.

### Technical Goals:
1. Create departmental OUs: `Sales`, `IT`, and `Finance`.
2. Provision Global Security Groups corresponding to departmental functions (`Sales-Team`, `IT-Admins`, `Finance-Team`).
3. Provision initial employee identities (`jdoe`, `jsmith`, `awong`) with secure initial credentials and enabled account states.
4. Establish security group memberships for Role-Based Access Control (RBAC).

---

## 2. Directory Hierarchy Architecture

```text
DC=lab,DC=local
├── OU=Sales
│   ├── User: Jane Doe (jdoe)
│   └── Group: Sales-Team (Members: jdoe)
├── OU=IT
│   ├── User: John Smith (jsmith)
│   └── Group: IT-Admins (Members: jsmith)
├── OU=Finance
│   ├── User: Alice Wong (awong)
│   └── Group: Finance-Team (Members: awong)
└── OU=Domain Controllers
    └── Computer: DC01$
```

---

## 3. Implementation Steps & Commands

Executed within the remote PowerShell SSH session on `DC01`:

### 3.1 Departmental OU Provisioning
```powershell
New-ADOrganizationalUnit -Name "Sales" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance" -Path "DC=lab,DC=local"
```

### 3.2 Security Group Creation (Global Scope)
```powershell
New-ADGroup -Name "Sales-Team" -GroupScope Global -Path "OU=Sales,DC=lab,DC=local"
New-ADGroup -Name "IT-Admins" -GroupScope Global -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance-Team" -GroupScope Global -Path "OU=Finance,DC=lab,DC=local"
```

> **Design Note (Group Scope):** **Global Groups** (`-GroupScope Global`) are used to group accounts with similar job responsibilities or resource needs. Following the AGDLP best practice (Account -> Global -> Domain Local -> Permission), users are nested in Global groups, which later assign permissions to Domain Local groups.

### 3.3 User Account Creation & Group Membership Assignment
```powershell
$userPass = ConvertTo-SecureString "Passw0rd!23" -AsPlainText -Force

# Provision Users
New-ADUser -Name "Jane Doe" -SamAccountName "jdoe" -Path "OU=Sales,DC=lab,DC=local" -AccountPassword $userPass -Enabled $true
New-ADUser -Name "John Smith" -SamAccountName "jsmith" -Path "OU=IT,DC=lab,DC=local" -AccountPassword $userPass -Enabled $true
New-ADUser -Name "Alice Wong" -SamAccountName "awong" -Path "OU=Finance,DC=lab,DC=local" -AccountPassword $userPass -Enabled $true

# Assign Group Memberships
Add-ADGroupMember -Identity "Sales-Team" -Members "jdoe"
Add-ADGroupMember -Identity "IT-Admins" -Members "jsmith"
Add-ADGroupMember -Identity "Finance-Team" -Members "awong"
```

---

## 4. Verification & Live Output

### 4.1 Verify Organizational Units
```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name
```
*Live Console Output:*
```text
Name
----
Domain Controllers
Sales
IT
Finance
```

### 4.2 Verify User Account Status
```powershell
Get-ADUser -Filter * -SearchBase "OU=Sales,DC=lab,DC=local" | Select-Object Name, SamAccountName, Enabled
```
*Live Console Output:*
```text
Name     SamAccountName Enabled
----     -------------- -------
Jane Doe jdoe              True
```

### 4.3 Verify Security Group Membership
```powershell
Get-ADGroupMember -Identity "Sales-Team" | Select-Object Name, SamAccountName
```
*Live Console Output:*
```text
Name     SamAccountName
----     --------------
Jane Doe jdoe
```

---

## 5. Architectural Takeaways

1. **Why OUs Instead of Default Containers**:
   * Objects in the default `CN=Users` container cannot have custom Group Policy Objects (GPOs) linked directly to them. Placing departments into top-level OUs allows targeting policies (e.g., desktop restrictions, password policies, audit rules) specifically to departments.
2. **Principle of Least Privilege**:
   * Departmental isolation ensures security groups (e.g., `Sales-Team`) only access resources assigned to their specific function, avoiding excessive administrative permissions.

---

## 6. Checkpoint Summary
- [x] Three departmental OUs provisioned (`Sales`, `IT`, `Finance`).
- [x] Global security groups established for each department.
- [x] Test accounts created, enabled, and mapped to corresponding functional groups.
- [x] Ready for Step 2.6 (Group Policy Object creation and linking).
