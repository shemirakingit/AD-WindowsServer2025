# Active Directory Lab — Windows Server 2025

Building a functional Active Directory environment from the ground up: domain controller promotion, OU structure, security groups, GPOs, and day-one help desk operations.

## Overview

This lab simulates the identity and access management foundation of a real enterprise Windows environment. It covers the full lifecycle of an AD deployment — from standing up a domain controller to handling the day-to-day tickets a help desk or sysadmin team would field.

**Environment:** Windows Server 2025 Datacenter, Azure VM (Standard_B2s)
**Domain:** `lab.local`
**Time to complete:** 3–5 hours across multiple sessions
# Watch Full Walkthrough
https://www.loom.com/share/34de67211b6f48ff920eb59e071a8e12

## Skills Demonstrated

- Promoting a Windows Server to a Domain Controller (new forest/domain)
- Designing an Organisational Unit (OU) structure by department
- Creating security groups and implementing role-based access control
- Provisioning user accounts via PowerShell
- Building and linking Group Policy Objects (GPOs)
- Domain-joining a client machine and validating policy application
- Core help desk operations: password resets, account unlocks, offboarding, auditing

## Environment Setup

| Setting | Value |
|---|---|
| Region | East US |
| Image | Windows Server 2025 Datacenter — Gen2 |
| Size | Standard_B2s (2 vCPU, 4GB RAM) |
| OS Disk | Standard SSD |
| Inbound Ports | RDP (3389) |

## Build Steps

### 1. Install Active Directory Domain Services
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-WindowsFeature -Name GPMC
```

### 2. Promote to Domain Controller
Created a new forest and root domain, `lab.local`.
```powershell
Import-Module ADDSDeployment
Install-ADDSForest -DomainName 'lab.local' -DomainNetBiosName 'LAB' `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString 'YourDSRMPassword!' -AsPlainText -Force) `
  -Force:$true
```

### 3. Organisational Units
```powershell
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "DC=lab,DC=local"
```

### 4. Security Groups
```powershell
New-ADGroup -Name "IT_Admins" -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users" -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users" -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
```

### 5. User Accounts
Naming convention: `firstname.lastname`. Users created and mapped to their department's OU and group.

```powershell
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

New-ADUser -Name "alice.chen" -GivenName "Alice" -Surname "Chen" `
  -SamAccountName "alice.chen" -UserPrincipalName "alice.chen@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" -AccountPassword $password -Enabled $true

# ... bob.patel (Finance), carol.jones (HR), david.smith (Sales)

Add-ADGroupMember -Identity "IT_Admins" -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users" -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users" -Members "david.smith"
```

### 6. Group Policy — IT Security Policy
Linked to the IT OU.

| Setting | Value | Purpose |
|---|---|---|
| Minimum password length | 12 | Enforce strong passwords |
| Password complexity | Enabled | Require mixed character types |
| Interactive logon inactivity limit | 900 seconds | Auto-lock after 15 minutes |
| Removable storage access | Deny all | Prevent data exfiltration via USB |

Validated by domain-joining a second VM, moving its computer object into the IT OU, running `gpupdate /force`, and confirming the lock screen policy applied under `alice.chen`.

## Help Desk Operations Practiced

**Password reset (force change at next logon)**
```powershell
Set-ADAccountPassword -Identity "bob.patel" -Reset -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)
Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true
```

**Unlock account**
```powershell
Unlock-ADAccount -Identity "carol.jones"
```

**Offboarding (disable, not delete)**
```powershell
Disable-ADAccount -Identity "david.smith"
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName
```

**Auditing**
```powershell
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} -Properties LastLogonDate | Select-Object Name, LastLogonDate
Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name
```

## Verification

| Check | Command | Expected Result |
|---|---|---|
| DC is running | `Get-ADDomainController` | Returns forest `lab.local` |
| OUs exist | `Get-ADOrganizationalUnit -Filter *` | Lists all 5 OUs |
| Users enabled | `Get-ADUser -Filter {Enabled -eq $true}` | Lists 4 test accounts |
| Group membership | `Get-ADGroupMember -Identity IT_Admins` | Returns `alice.chen` |
| GPO linked | `Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'` | Shows IT Security Policy |

## Issues Encountered & Fixes

- **`New-ADUser` prompting for `Name:`** — caused by running commands line-by-line before `$password` was defined. Fixed by executing the full script block together.
- **Clipboard not syncing over RDP** — the Azure portal's browser-based console has limited clipboard support. Resolved by downloading the RDP file from the portal and connecting via the native Remote Desktop app instead, with clipboard sharing enabled under Local Resources.
- **DNS conflict during promotion** — resolved by setting the NIC's preferred DNS to `127.0.0.1` before promoting the server.

## Lessons Learned

Standing up AD from scratch makes it clear why group-based access control scales the way it does — provisioning and deprovisioning access is a single group membership change, not dozens of individual permission edits. The OU-to-GPO relationship is also the core mechanic behind centralized policy enforcement in every enterprise Windows environment, and directly maps to how Entra ID conditional access and role assignments work in hybrid/cloud setups.

## Tech Stack

`Windows Server 2025` `Active Directory Domain Services` `Group Policy Management` `Azure` `PowerShell`

---
*Part of a hands-on IT infrastructure portfolio series covering Active Directory, Azure, Nessus, Wireshark, ServiceNow, and Terraform.*
