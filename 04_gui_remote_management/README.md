# 04 - Remote GUI Management & Shares

## 1. RSAT Installation

```powershell
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
```

## 2. Launching the Administration Console

`Win + R` > Enter `dsa.msc`.

## 3. Creating Active Directory Objects

1. **OU:** Right-click on `TechCorp > Users` > **New** > **Organizational Unit** > `Management`.
2. **User:** Right-click on `Management` > **New** > **User**:

* Full name: `Alain Dir`
* Login: `a.dir`

3. **Groups (AGDLP):** Right-click on `TechCorp > Groups` > **New** > **Group**:

* `GG_DIR` (Scope: **Global** / Type: **Security**) -> Add `a.dir` under the *Members* tab.
* `DL_Partage_Direction_RW` (Scope: **Domain Local** / Type: **Security**) -> Add `GG_DIR` under the *Members* tab.

## 4. Creating the Folder and SMB Share

`Win + R` > Enter `compmgmt.msc` (Computer Management).

1. Right-click on **Computer Management (Local)** > **Connect to another computer...** > Enter `DC1`.
2. Navigate to: **System Tools** > **Shared Folders** > **Shares**.
3. Right-click in the empty area > **New Share...**:

* **Folder path:** `C:\Shares\Management`
* **Share name:** `Management$`
* **Share permissions:** *Customize* > `Everyone` -> Check **Full Control**.

## 5. Configuring NTFS Permissions

1. **Breaking inheritance:** Click **Disable inheritance** > *Convert inherited permissions into explicit permissions*.

2. **Cleaning up entries:**

* Remove `Users`.
* Remove `CREATOR OWNER`.
* Keep `SYSTEM` (**Full Control**) and `Administrators` (**Full Control**).

3. **Adding group access:**

* Click **Add** > **Select a principal** > Search for `DL_Partage_Direction_RW`.
* Permissions: Check **Modify**.

## 6. Validating Client Access (TECHCORP\a.dir)

1. `Win + R` > Enter `\\DC1\Management$`.
2. Create a test file `test_ecriture.txt` in the folder to validate write permissions.
