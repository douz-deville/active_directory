# 02 Context: The "TechCorp" Company

The TechCorp company has two departments: Information Technology (IT) and Human Resources (HR), with access to a shared folder and security policies.

## 1. Creating the OU Structure

One main OU: TechCorp.

Inside TechCorp, 3 sub-OUs: Users, Groups, and Computers.

Inside Users, two sub-OUs: IT and HR.

```powershell
New-ADOrganizationalUnit -Name 'TechCorp' -Path 'DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'Users' -Path 'OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'Groups' -Path 'OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'Computers' -Path 'OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'IT' -Path 'OU=Users,OU=TechCorp,DC=xyz,DC=com'

New-ADOrganizationalUnit -Name 'HR' -Path 'OU=Users,OU=TechCorp,DC=xyz,DC=com'
```

## 2. Creating Users

In Users > IT: Marc Tech (m.tech).

In Users > HR: Julie Rh (j.rh).

```powershell
$Password = ConvertTo-SecureString 'Passw0rd' -AsPlainText -Force

New-ADUser -Name 'Marc Tech' -GivenName 'Marc' -Surname 'Tech' -SamAccountName 'm.tech' -UserPrincipalName 'm.tech@xyz.com' -Path 'OU=IT,OU=Users,OU=TechCorp,DC=xyz,DC=com' -AccountPassword $Password -Enabled $true -ChangePasswordAtLogon $false

New-ADUser -Name 'Julie Rh' -GivenName 'Julie' -Surname 'Rh' -SamAccountName 'j.rh' -UserPrincipalName 'j.rh@xyz.com' -Path 'OU=HR,OU=Users,OU=TechCorp,DC=xyz,DC=com' -AccountPassword $Password -Enabled $true -ChangePasswordAtLogon $false
```

#### NB: PowerShell requires a user's password to be provided as a secure string (SecureString)

## 3. Creating Groups

In Groups:

A Global Group named GG_RH (members: j.rh).

A Global Group named GG_IT (members: m.tech).

A Domain Local Group named DL_Partage_RH_RW (Read/Write access to the HR share).

GG_RH as a member of DL_Partage_RH_RW.

1. Creating the Groups

```powershell
New-ADGroup -Name "GG_RH" -SamAccountName "GG_RH" -GroupCategory Security -GroupScope Global -DisplayName "GG_RH" -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Members of the HR team"

New-ADGroup -Name "GG_IT" -SamAccountName "GG_IT" -GroupCategory Security -GroupScope Global -DisplayName "GG_IT" -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Members of the IT team"

New-ADGroup -Name "DL_Partage_RH_RW" -SamAccountName "DL_Partage_RH_RW" -GroupCategory Security -GroupScope DomainLocal -DisplayName "DL_Partage_RH_RW" -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Read and Write access to the HR share"
```

2. Adding Members

```powershell
Add-ADGroupMember -Identity "GG_RH" -Members "j.rh"

Add-ADGroupMember -Identity "GG_IT" -Members "m.tech"

Add-ADGroupMember -Identity "DL_Partage_RH_RW" -Members "GG_RH"
```
