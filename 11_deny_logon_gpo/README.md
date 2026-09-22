# 11 - CROSS-TIER LOGON RESTRICTIONS (DENY LOGON GPO)

## 1. Security Context & Architecture

This step implements the principle of strict privilege isolation (*Enterprise Access Model*). It applies explicit *Deny Logon* restrictions at the Windows security policy level to isolate administrative tiers and prevent lateral movement or vertical privilege escalation.

* **Windows Rule Precedence:** Explicit *Deny* permissions always take precedence over *Allow* permissions.

* **LSASS Protection:** Prevents the injection and persistence of administrative credentials (NTLM hashes, Kerberos tickets) in the volatile memory of lower-tier systems.

* **Multi-Dimensional Blocking:** Applied across the four authentication vectors (Local Interactive, Remote Desktop, Scheduled Tasks, and Windows Services).


## 2. Group Policy Isolation Matrix

| **GPO Name**                   | **Target OUs (Links)**                     | **Restricted Groups**                          | **Applied Restrictions (User Rights)**               |
| ------------------------------ | ------------------------------------------ | ---------------------------------------------- | ---------------------------------------------------- |
| **GPO_Deny_T0_EverywhereElse** | `OU=Servers`, `OU=Workstations`            | `GG_T0_Admins`, `Domain Admins`                | Deny Interactive, Deny RDP, Deny Batch, Deny Service |
| **GPO_Deny_T1_On_T0_T2**       | `OU=Domain Controllers`, `OU=Workstations` | `GG_T1_Admins`                                 | Deny Interactive, Deny RDP, Deny Batch, Deny Service |
| **GPO_Deny_NonT0_On_DCs**      | `OU=Domain Controllers`                    | `Domain Users`, `GG_T1_Admins`, `GG_T2_Admins` | Deny Interactive, Deny RDP                           |


## 3. Graphical Deployment via `gpmc.msc`

### GPO 1: `GPO_Deny_T0_EverywhereElse`

* **Path:** `Computer Configuration` > `Policies` > `Windows Settings` > `Security Settings` > `Local Policies` > `User Rights Assignment`

* **Configured rights** for `xyz\GG_T0_Admins` and `xyz\Domain Admins`:

  * **Deny log on locally** (`SeDenyInteractiveLogonRight`)

  * **Deny log on through Remote Desktop Services** (`SeDenyRemoteInteractiveLogonRight`)

  * **Deny log on as a batch job** (`SeDenyBatchLogonRight`)

  * **Deny log on as a service** (`SeDenyServiceLogonRight`)

* **Links:** `OU=Servers,OU=TechCorp,DC=xyz,DC=com` and `OU=Workstations,OU=TechCorp,DC=xyz,DC=com`.


### GPO 2: `GPO_Deny_T1_On_T0_T2`

* **Path:** Same path as above.

* **Configured rights** for `xyz\GG_T1_Admins`:

  * **Deny log on locally**

  * **Deny log on through Remote Desktop Services**

  * **Deny log on as a batch job**

  * **Deny log on as a service**

* **Links:** `OU=Domain Controllers,DC=xyz,DC=com` and `OU=Workstations,OU=TechCorp,DC=xyz,DC=com`.


### GPO 3: `GPO_Deny_NonT0_On_DCs`

* **Path:** Same path as above.

* **Configured rights** for `xyz\Domain Users`, `xyz\GG_T1_Admins`, and `xyz\GG_T2_Admins`:

  * **Deny log on locally**

  * **Deny log on through Remote Desktop Services**

* **Link:** `OU=Domain Controllers,DC=xyz,DC=com`.


## 4. Audit and Compliance Script (`Audit-DenyLogon.ps1`)

```powershell
Write-Host "====================================================" -ForegroundColor Cyan
Write-Host "       GPO DENY LOGON RESTRICTIONS AUDIT           " -ForegroundColor Cyan
Write-Host "====================================================" -ForegroundColor Cyan

Import-Module GroupPolicy
Import-Module ActiveDirectory

$DomainDN = (Get-ADDomain).DistinguishedName

$ExpectedGPOs = @(
    @{ 
        Name = "GPO_Deny_T0_EverywhereElse"
        Targets = @(
            "OU=Servers,OU=TechCorp,$DomainDN"
            "OU=Workstations,OU=TechCorp,$DomainDN"
        )
        RequiredRights = @(
            "SeDenyInteractiveLogonRight"
            "SeDenyRemoteInteractiveLogonRight"
            "SeDenyBatchLogonRight"
            "SeDenyServiceLogonRight"
        )
    },
    @{ 
        Name = "GPO_Deny_T1_On_T0_T2"
        Targets = @(
            "OU=Domain Controllers,$DomainDN"
            "OU=Workstations,OU=TechCorp,$DomainDN"
        )
        RequiredRights = @(
            "SeDenyInteractiveLogonRight"
            "SeDenyRemoteInteractiveLogonRight"
            "SeDenyBatchLogonRight"
            "SeDenyServiceLogonRight"
        )
    },
    @{ 
        Name = "GPO_Deny_NonT0_On_DCs"
        Targets = @(
            "OU=Domain Controllers,$DomainDN"
        )
        RequiredRights = @(
            "SeDenyInteractiveLogonRight"
            "SeDenyRemoteInteractiveLogonRight"
        )
    }
)

# 1. Check GPO existence and OU links
Write-Host "`n[1/2] Checking GPO existence and OU links..." -ForegroundColor Yellow

foreach ($item in $ExpectedGPOs) {

    $gpoName = $item.Name
    $gpo = Get-GPO -Name $gpoName -ErrorAction SilentlyContinue

    if (-not $gpo) {
        Write-Host "  - GPO '$gpoName': [FAIL - Not Found]" -ForegroundColor Red
        continue
    }

    Write-Host "  - GPO '$gpoName' detected: [PASS]" -ForegroundColor Green

    foreach ($targetOU in $item.Targets) {

        $inheritance = Get-GPInheritance `
            -Target $targetOU `
            -ErrorAction SilentlyContinue

        $isLinked = $inheritance.GpoLinks |
            Where-Object {
                $_.DisplayName -eq $gpoName -and $_.Enabled
            }

        if ($isLinked) {
            Write-Host "    └─ Active link on $targetOU: [PASS]" -ForegroundColor Green
        }
        else {
            Write-Host "    └─ Active link on $targetOU: [FAIL - Not Linked]" -ForegroundColor Red
        }
    }
}

# 2. Validate security policy settings inside each GPO
Write-Host "`n[2/2] Inspecting internal security policy settings (Deny Rules)..." -ForegroundColor Yellow

foreach ($item in $ExpectedGPOs) {

    $gpoName = $item.Name
    $gpo = Get-GPO -Name $gpoName -ErrorAction SilentlyContinue

    if ($gpo) {

        [xml]$report = Get-GPOReport `
            -Name $gpoName `
            -ReportType Xml

        $xmlText = $report.OuterXml

        $allRightsPresent = $true

        foreach ($right in $item.RequiredRights) {

            if ($xmlText -notmatch $right) {
                $allRightsPresent = $false
            }
        }

        if ($allRightsPresent) {
            Write-Host "  - Deny rights configuration for '$gpoName': [PASS]" `
                -ForegroundColor Green
        }
        else {
            Write-Host "  - Deny rights configuration for '$gpoName': [FAIL - Missing Settings]" `
                -ForegroundColor Red
        }
    }
}

Write-Host "`n====================================================" -ForegroundColor Cyan
Write-Host "                  AUDIT COMPLETED                  " -ForegroundColor Cyan
Write-Host "====================================================" -ForegroundColor Cyan
```
