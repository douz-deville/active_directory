# 04 - Deployment and Linking of Group Policy Objects (GPOs)

The objective of this step is to configure and deploy Group Policy Objects (GPOs) to centrally manage user environments and enforce security restrictions on Windows 11 workstations. The implemented policies automate departmental network drive mapping, restrict access to administrative and command-line tools for standard users, and define appropriate exclusions for IT administrator accounts.


## 1. GPO 01: Automatic Network Drive Mapping (`GPO_User_DriveMapping_Direction`)

**Objective:** Automatically map the hidden network share `\\DC1\Direction$` to drive letter **`Z:`** for members of the Direction OU.

### Configuration (`gpmc.msc`)

* **Target:** `User Configuration` > `Preferences` > `Windows Settings` > `Drive Maps`

* **New mapped drive:**

* **Action:** `Update`

* **Location:** `\\DC1\Direction$`

* **Reconnect:** Checked *(Persistence in the `HKCU\Network\Z` registry key)*

* **Drive letter:** `Z:`

* **Link:** `TechCorp > Utilisateurs > Direction`


## 2. GPO 02: Workstation Security and Restrictions (`GPO_User_SecurityRestrictions`)

**Objective:** Reduce the attack surface of Windows 11 client workstations for all standard users.

### A. Blocking Control Panel and Settings

* **Path:** `User Configuration > Administrative Templates > Control Panel`
* **Setting:** **Prohibit access to Control Panel and PC settings** → `Enabled`
* **Effect:** Blocks the execution of `control.exe` and the `ms-settings:` URI.

### B. Restricting Command Interpreters (`cmd`, `powershell`, `wt`)

* **Command Prompt (CMD):**
* **Path:** `User Configuration > Administrative Templates > System`
* **Setting:** **Prevent access to the command prompt** → `Enabled`
* **Option:** *Disable the command prompt script processing also* = `Yes` *(Blocks `cmd.exe`, `.bat`, and `.cmd` files).*

> **Important note (Setting scope):**
> The native *Prevent access to the command prompt* policy targets only `cmd.exe`. It does not block PowerShell (`powershell.exe`) or Windows Terminal (`wt.exe`).

* **Additional restriction (PowerShell & Terminal):**
* **Path:** `User Configuration > Administrative Templates > System`
* **Setting:** **Don't run specified Windows applications** → `Enabled`
* **List of blocked applications:**
* `powershell.exe`
* `powershell_ise.exe`
* `wt.exe` *(Windows Terminal)*

### D. Excluding IT Administrators

To prevent administrative accounts from being affected by the restrictions:

* Isolate administrator accounts in a dedicated OU (`TechCorp > Admins`) that is not targeted by the GPO.
* **Delegation method (Alternative):** In the **Delegation > Advanced** tab of the GPO, add the `GG_IT_Admins` group and select **Deny** for the *Apply Group Policy* permission.
