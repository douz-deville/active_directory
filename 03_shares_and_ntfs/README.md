# 03 File Services and Network / NTFS Permissions

This module covers the creation of a centralized shared folder for the HR department and the fine-grained application of network (SMB) and file system (NTFS) permissions according to the AGDLP method.

### Objectives:

* Create the local folder `C:\Partages\Donnees_RH` and share it under the hidden name `RH$`.
* Configure the SMB share with `Everyone = Full Control` access.
* Disable NTFS inheritance and restrict permissions to the `DL_Partage_RH_RW` group.
* **Expected result:** Julie (`j.rh`) has full modify access, while Marc (`m.tech`) receives an *"Access denied"* message.

### Golden Rule for Active Directory Shares:

* **Share Permissions (SMB):** Open as much as possible (`Everyone = Full Control`).
* **NTFS Permissions (Local Security):** Precisely restricted using domain local security groups (`DL_*`).

## 1. Creating the Folder and SMB Share

```powershell
# 1. Create the local folder on the server (-Force creates parent folders if they do not exist)
New-Item -Path "C:\Partages\Donnees_RH" -ItemType Directory -Force

# 2. Create the SMB network share (Hidden RH$ share)
New-SmbShare -Name "RH$" `
             -Path "C:\Partages\Donnees_RH" `
             -FullAccess "Everyone" `
             -Description "Confidential HR department folder"
```

## 2. Configuring NTFS Permissions (AGDLP Method)

By default, a new folder inherits permissions from the C: drive. Therefore, inheritance must be broken, basic system access must be reset, and exclusive access must be granted to the `DL_Partage_RH_RW` group.

```powershell
# Define the path
$Path = "C:\Partages\Donnees_RH"

# 1. Retrieve the current ACL
$Acl = Get-Acl -Path$Path

# 2. Disable inheritance ($true = Protect the ACL, $false = Remove current inherited rules)
$Acl.SetAccessRuleProtection($true,$false)

# 3. Reset basic system permissions (Administrators & SYSTEM)
$AdminRule = New-Object System.Security.AccessControl.FileSystemAccessRule("BUILTIN\Administrators", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$SystemRule = New-Object System.Security.AccessControl.FileSystemAccessRule("NT AUTHORITY\SYSTEM", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")

$Acl.SetAccessRule($AdminRule)
$Acl.SetAccessRule($SystemRule)

# 4. Add the Domain Local group with Modify permissions (Read, Write, Delete)
$GroupIdentity = "xyz\DL_Partage_RH_RW" # Replace 'xyz' with your domain's NetBIOS name
$Rights = "Modify"                      # "Modify" includes Read, Write, and Delete
$Inheritance = "ContainerInherit, ObjectInherit"
$Propagation = "None"
$Type = "Allow"

$RHAccessRule = New-Object System.Security.AccessControl.FileSystemAccessRule($GroupIdentity, $Rights,$Inheritance, $Propagation,$Type)
$Acl.AddAccessRule($RHAccessRule)

# 5. Apply the new ACL to the folder
Set-Acl -Path $Path -AclObject$Acl
```

## 3. Access Verification

To validate the functionality from the Windows 11 client VM:

```
1. Test with Julie Rh (j.rh):

    - Log in to the client VM with the j.rh account.

    - Open File Explorer and enter: \\DC1\RH$ (replace DC1 with the name of your server).

    - Result: The folder opens. Julie can create, modify, and delete files.

2. Test with Marc Tech (m.tech):

    - Log in to the client VM with the m.tech account.

    - Enter: \\DC1\RH$.

    - Result: An "Access denied" message (Windows Network Error) appears.
```
