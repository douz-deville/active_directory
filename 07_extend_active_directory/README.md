# 07 - Extended Active Directory Infrastructure

This module extends the **TechCorp** Active Directory topology by introducing new Organizational Units (OUs), departmental users, and security groups following the **AGDLP** (Account, Global, Domain Local, Permission) role-based access model.


## 1. Structural Overview

### Organizational Units (OUs)

* **`OU=Finance,OU=Users,OU=TechCorp,DC=xyz,DC=com`**: Dedicated OU for Finance personnel.
* **`OU=Servers,OU=TechCorp,DC=xyz,DC=com`**: Staging OU for member servers.
* **`OU=Admins,OU=TechCorp,DC=xyz,DC=com`**: Staging OU for administrative accounts.
* **`OU=Service Accounts,OU=TechCorp,DC=xyz,DC=com`**: Staging OU for gMSAs and dedicated service accounts.

### AGDLP Role Mapping

| Global Group (GG) | Domain Local Group (DL) | Scope / Target Resource |
| --- | --- | --- |
| `GG_FIN` | `DL_Files_Finance_RW` | Read/Write access to Finance file shares |
| `GG_IT` | `DL_Server_IT` | Administrative rights on member servers |

---

## 2. Deployment Script

```powershell
# ==========================================
# 1. ORGANIZATIONAL UNITS CREATION
# ==========================================
New-ADOrganizationalUnit -Name 'Finance'          -Path 'OU=Users,OU=TechCorp,DC=xyz,DC=com'
New-ADOrganizationalUnit -Name 'Servers'          -Path 'OU=TechCorp,DC=xyz,DC=com'
New-ADOrganizationalUnit -Name 'Admins'           -Path 'OU=TechCorp,DC=xyz,DC=com'
New-ADOrganizationalUnit -Name 'Service Accounts' -Path 'OU=TechCorp,DC=xyz,DC=com'

# ==========================================
# 2. SECURITY GROUPS CREATION (AGDLP)
# ==========================================
# Global Groups
New-ADGroup -Name "GG_FIN" -SamAccountName "GG_FIN" -GroupCategory Security -GroupScope Global `
    -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Members of the Finance team"

# Domain Local Groups
New-ADGroup -Name "DL_Files_Finance_RW" -SamAccountName "DL_Files_Finance_RW" -GroupCategory Security -GroupScope DomainLocal `
    -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Read and Write access to the Finance share"

New-ADGroup -Name "DL_Server_IT" -SamAccountName "DL_Server_IT" -GroupCategory Security -GroupScope DomainLocal `
    -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Administrative access to servers"

# ==========================================
# 3. USER ACCOUNTS CREATION
# ==========================================
$Password = ConvertTo-SecureString "Passw0rd123!" -AsPlainText -Force

# HR User
New-ADUser -Name "Alice HR" -SamAccountName "alice.hr" -UserPrincipalName "alice.hr@xyz.com" `
    -AccountPassword $Password -Enabled $true -Path "OU=HR,OU=Users,OU=TechCorp,DC=xyz,DC=com" -Description "HR user"

# Finance User
New-ADUser -Name "Bob Finance" -SamAccountName "bob.finance" -UserPrincipalName "bob.finance@xyz.com" `
    -AccountPassword $Password -Enabled $true -Path "OU=Finance,OU=Users,OU=TechCorp,DC=xyz,DC=com" -Description "Finance user"

# IT User
New-ADUser -Name "Charlie IT" -SamAccountName "charlie.it" -UserPrincipalName "charlie.it@xyz.com" `
    -AccountPassword $Password -Enabled $true -Path "OU=IT,OU=Users,OU=TechCorp,DC=xyz,DC=com" -Description "IT user"

# ==========================================
# 4. GROUP MEMBERSHIP ASSIGNMENTS
# ==========================================
# Add Users to Global Groups
Add-ADGroupMember -Identity "GG_RH"  -Members "alice.hr"
Add-ADGroupMember -Identity "GG_FIN" -Members "bob.finance"
Add-ADGroupMember -Identity "GG_IT"  -Members "charlie.it"

# Nest Global Groups into Domain Local Groups
Add-ADGroupMember -Identity "DL_Files_Finance_RW" -Members "GG_FIN"
Add-ADGroupMember -Identity "DL_Server_IT" -Members "GG_IT"

```
