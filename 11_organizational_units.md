# 11 - Organizational Units

## Goal

An Organizational Unit (OU) is a container inside Active Directory used to organize users, computers, and groups. OUs are the targeting mechanism for two things:
* Group Policy (GPO) links — GPOs get linked to OUs, not to individual users/computers.
* Delegated administration — in a real environment, you'd delegate "IT-Admins can reset passwords for users in the Sales OU" type permissions at the OU level.
OU structure is genuinely hard to restructure later without breaking existing GPO links and group scopes
Accidental-deletion protection is a real AD best practice. OUs can contain hundreds of objects, and a fat-fingered delete in Active Directory Users and Computers (ADUC) with no protection wipes everything inside instantly with no confirmation beyond the usual Windows prompt. This checkbox forces an extra step (removing the protection flag) before deletion is even possible.

## Objectives

- Design OU structure before creating it 
- Create via ADUC/New-ADOrganizationalUnit
- Enable accidental-deletion protection
- Document the structure

## Steps

### Design the structure first (on paper/in your doc)

I choose to design the AD structure by function/department like the following:
```
lab.local
├── OU=IT
│   ├── OU=Users
│   └── OU=Computers
├── OU=Sales
│   ├── OU=Users
│   └── OU=Computers
├── OU=Servers
├── OU=Service Accounts
└── OU=Security Groups
```
This is because it more directly demonstrates the real-world pattern of "GPO applies to everyone in Sales" being a single link at the OU=Sales level.

For this lab, the structure will be like this:
```
lab.local
├── OU=Corp
│   ├── OU=Users
│   ├── OU=Computers
│   └── OU=Groups
├── OU=Servers          (DC01 stays in Domain Controllers by default)
├── OU=Service Accounts (for WEB01→DB01 SQL login context, WebApp accounts, etc.)
└── OU=DMZ Hosts         (optional - conceptual placeholder, since WEB01 won't actually be domain-joined)
```

### Create the OUs via PowerShell (faster and scriptable)

```powershell
New-ADOrganizationalUnit -Name "Corp" -Path "DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Users" -Path "OU=Corp,DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Computers" -Path "OU=Corp,DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Groups" -Path "OU=Corp,DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Servers" -Path "DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Service Accounts" -Path "DC=lab,DC=local" -ProtectedFromAccidentalDeletion $true
```
Note: -ProtectedFromAccidentalDeletion $true is the default behavior for New-ADOrganizationalUnit even if you omit the flag — but specifying it explicitly is good practice and self-documenting.

![Adding OUs in Powershell](<docs/screenshots/11_organizational_units/adding ou in powershell.png>)

### Or create via ADUC (GUI path)

1. Open Active Directory Users and Computers (dsa.msc)
* Also located in Server Manager → Tools → Active Directory Users and Computers
2. Right-click lab.local → New → Organizational Unit
3. Name it, and note the checkbox "Protect container from accidental deletion" is checked by default — leave it checked
4. Repeat for each OU in your structure, right-clicking the parent OU to nest sub-OUs correctly

![Adding OUs in the GUI](<docs/screenshots/11_organizational_units/adding ou in gui.png>)

### Verify the structure

```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```
This gives you a full list to confirm nesting is correct.
![Verifying AD structure in Powershell](<docs/screenshots/11_organizational_units/verify structure powershell.png>)

### Confirm deletion protection is actually active

```powershell
Get-ADOrganizationalUnit -Identity "OU=Corp,DC=lab,DC=local" -Properties ProtectedFromAccidentalDeletion | Select-Object Name, ProtectedFromAccidentalDeletion
```
Should return True. Repeat for a couple others by changing the identity value as a spot check. 

![Verifying deletion protection in Powershell](<docs/screenshots/11_organizational_units/verify delete protection.png>)

### Test it (optional) — try to delete one and watch it fail

In ADUC, right-click one of the protected OUs → Delete → you should get a message stating it can't be deleted while protection is enabled.

![Testing OU deletion 1](<docs/screenshots/11_organizational_units/test ou deletion 1.png>)
![Testing OU deletion 2](<docs/screenshots/11_organizational_units/test ou deletion 2.png>)

## Resources
Windows Organizational Units: https://learn.microsoft.com/en-us/entra/identity/domain-services/create-ou