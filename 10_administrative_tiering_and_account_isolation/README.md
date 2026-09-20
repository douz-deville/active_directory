# 10 - ADMINISTRATIVE TIERING & ACCOUNT ISOLATION

**1. Context & Architecture**
This step implements the **Enterprise Access Model (Tiering)** to prevent credential theft attacks (Pass-the-Hash, Kerberos Delegation abuse, Kerberoasting) and eliminate lateral movement.

* **Principle of Identity Separation:** Daily accounts (`charlie.it`, `m.tech`, etc.) are kept strictly unprivileged and are used only for standard daily operations (email, web, office applications). Dedicated administrative accounts are created for each administrative plane.
* **Tier 0 (Control Plane):** Identity infrastructure, Domain Controllers, PKI. Managed by `adm_t0_*` accounts.
* **Tier 1 (Server Plane):** Member servers, enterprise applications, database servers. Managed by `adm_t1_*` accounts.
* **Tier 2 (Workstation Plane):** End-user devices and helpdesk operations. Managed by `adm_t2_*` accounts.


**2. Organizational Units & Security Groups Deployment**

```powershell
# 1. Create Tiering Sub-OUs under the existing Admins OU
New-ADOrganizationalUnit -Name 'Tier 0' -Path 'OU=Admins,OU=TechCorp,DC=xyz,DC=com'
New-ADOrganizationalUnit -Name 'Tier 1' -Path 'OU=Admins,OU=TechCorp,DC=xyz,DC=com'
New-ADOrganizationalUnit -Name 'Tier 2' -Path 'OU=Admins,OU=TechCorp,DC=xyz,DC=com'

# 2. Create Global Groups for Administrative Roles (AGDLP)
New-ADGroup -Name "GG_T0_Admins" -SamAccountName "GG_T0_Admins" -GroupCategory Security -GroupScope Global `
    -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Domain & Infrastructure Administrators (Tier 0)"

New-ADGroup -Name "GG_T1_Admins" -SamAccountName "GG_T1_Admins" -GroupCategory Security -GroupScope Global `
    -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Member Server Administrators (Tier 1)"

New-ADGroup -Name "GG_T2_Admins" -SamAccountName "GG_T2_Admins" -GroupCategory Security -GroupScope Global `
    -Path "OU=Groups,OU=TechCorp,DC=xyz,DC=com" -Description "Workstation & Helpdesk Support (Tier 2)"

# 3. Nest Tier 0 Global Group into native Domain Admins
Add-ADGroupMember -Identity "Domain Admins" -Members "GG_T0_Admins"

```

**3. Administrative Account Creation & Native Hardening**

```powershell
$Password = ConvertTo-SecureString "P@ssw0rd12345678!" -AsPlainText -Force

# 1. Create Tier 0 Account for Charlie (AD/DC Management)
New-ADUser -Name "Charlie IT (Admin T0)" `
           -SamAccountName "adm_t0_charlie" `
           -UserPrincipalName "adm_t0_charlie@xyz.com" `
           -Path "OU=Tier 0,OU=Admins,OU=TechCorp,DC=xyz,DC=com" `
           -AccountPassword $Password `
           -Enabled $true `
           -PasswordNeverExpires $false `
           -CannotChangePassword $false `
           -Description "Tier 0 Administrative Account - Active Directory Infrastructure Only"

# 2. Create Tier 1 Account for Charlie (Server Management)
New-ADUser -Name "Charlie IT (Admin T1)" `
           -SamAccountName "adm_t1_charlie" `
           -UserPrincipalName "adm_t1_charlie@xyz.com" `
           -Path "OU=Tier 1,OU=Admins,OU=TechCorp,DC=xyz,DC=com" `
           -AccountPassword $Password `
           -Enabled $true `
           -PasswordNeverExpires $false `
           -CannotChangePassword $false `
           -Description "Tier 1 Administrative Account - Member Servers Only"

# 3. Assign Accounts to their Respective Tier Groups
Add-ADGroupMember -Identity "GG_T0_Admins" -Members "adm_t0_charlie"
Add-ADGroupMember -Identity "GG_T1_Admins" -Members "adm_t1_charlie"

# 4. Native Security Controls Enrolment (Protected Users & Kerberos Restriction)
# Add T0 accounts to Protected Users (Disables NTLM, prevents credential caching, forces strict Kerberos)
Add-ADGroupMember -Identity "Protected Users" -Members "adm_t0_charlie", "Administrator"

# Enable Kerberos flag "Account is sensitive and cannot be delegated"
Set-ADUser -Identity "adm_t0_charlie" -AccountNotDelegated $true
Set-ADUser -Identity "Administrator" -AccountNotDelegated $true

```

**4. Final Validated Audit Script (`Audit-Tiering.ps1`)**

```powershell
Write-Host "====================================================" -ForegroundColor Cyan
Write-Host "       AUDIT DE CONFORMITÉ TIERING & ISOLATION      " -ForegroundColor Cyan
Write-Host "====================================================" -ForegroundColor Cyan

Import-Module ActiveDirectory

# ====================================================
# 1. PROTECTED USERS - TIER 0
# ====================================================
Write-Host "`n[1/4] Vérification du groupe 'Protected Users' (Tier 0)..." -ForegroundColor Yellow

$protMembers = @(
    Get-ADGroupMember -Identity "Protected Users" -ErrorAction SilentlyContinue
)

$t0Members = @(
    Get-ADGroupMember -Identity "GG_T0_Admins" -ErrorAction SilentlyContinue
)

if ($t0Members.Count -eq 0) {
    Write-Host "  - Aucun membre trouvé dans GG_T0_Admins." -ForegroundColor Red
}
else {
    foreach ($user in $t0Members) {$isProtected = $protMembers.DistinguishedName -contains$user.DistinguishedName
        $status = if ($isProtected) { "PASS" } else { "FAIL" }
        $color  = if ($isProtected) { "Green" } else { "Red" }

        Write-Host "  - Compte $($user.SamAccountName) dans Protected Users : [$status]" -ForegroundColor $color
    }
}

# ====================================================
# 2. CANNOT BE DELEGATED - TIER 0
# ====================================================
Write-Host "`n[2/4] Vérification du fanion 'Cannot be delegated'..." -ForegroundColor Yellow

foreach ($user in $t0Members) {
    $adUser = Get-ADUser -Identity $user.SamAccountName -Properties AccountNotDelegated -ErrorAction SilentlyContinue

    if ($adUser) {
        $isNotDelegated = $adUser.AccountNotDelegated
        $status = if ($isNotDelegated) { "PASS" } else { "FAIL" }
        $color  = if ($isNotDelegated) { "Green" } else { "Red" }

        Write-Host "  - Compte $($user.SamAccountName) 'Cannot be delegated' : [$status]" -ForegroundColor $color
    }
}

# ====================================================
# 3. ISOLATION EMAIL - T0 / T1
# ====================================================
Write-Host "`n[3/4] Vérification de l'absence d'adresse e-mail sur T0/T1..." -ForegroundColor Yellow

$allAdmins = @(
    Get-ADGroupMember -Identity "GG_T0_Admins" -ErrorAction SilentlyContinue
    Get-ADGroupMember -Identity "GG_T1_Admins" -ErrorAction SilentlyContinue
)

if ($allAdmins.Count -eq 0) {
    Write-Host "  - Aucun membre trouvé dans GG_T0_Admins ou GG_T1_Admins." -ForegroundColor Red
}
else {
    foreach ($user in$allAdmins) {
        $adUser = Get-ADUser -Identity$user.SamAccountName -Properties EmailAddress -ErrorAction SilentlyContinue

        if ($adUser) {
            $hasNoEmail = [string]::IsNullOrEmpty($adUser.EmailAddress)

            if ($hasNoEmail) {$status = "PASS"
                $color  = "Green"
            }
            else {
                $status = "FAIL (Email détecté)"
                $color  = "Red"
            }

            Write-Host "  - Compte $($user.SamAccountName) sans e-mail : [$status]" -ForegroundColor $color
        }
    }
}

# ====================================================
# 4. COMPTES QUOTIDIENS NON PRIVILEGIES
# ====================================================
Write-Host "`n[4/4] Vérification du statut des comptes quotidiens..." -ForegroundColor Yellow

$dailyUsers = @(
    "charlie.it"
    "m.tech"
    "j.rh"
    "alice.hr"
    "bob.finance"
)

$adminGroups = @(
    "Domain Admins"
    "GG_T0_Admins"
    "GG_T1_Admins"
)

foreach ($u in $dailyUsers) {
    $userObj = Get-ADUser -Identity $u -ErrorAction SilentlyContinue

    if (-not $userObj) {
        Write-Host "  - Compte quotidien $u : [NON TROUVÉ]" -ForegroundColor Yellow
        continue
    }

    $isClean = $true
    $violations = @()

    $userGroups = @(
        Get-ADPrincipalGroupMembership -Identity $userObj -ErrorAction SilentlyContinue
    )

    foreach ($grp in $adminGroups) {
        $isMember = $userGroups.Name -contains $grp

        if ($isMember) {
            $isClean = $false
            $violations += $grp
        }
    }

    if ($isClean) {
        $status = "PASS"
        $color  = "Green"

        Write-Host "  - Compte quotidien $u est non-privilégié : [$status]" -ForegroundColor $color
    }
    else {
        $status = "FAIL"
        $color  = "Red"

        Write-Host "  - Compte quotidien $u est non-privilégié : [$status]" -ForegroundColor $color
        Write-Host "      Groupes privilégiés détectés : $($violations -join ', ')" -ForegroundColor Red
    }
}

Write-Host "`n====================================================" -ForegroundColor Cyan
Write-Host "                 AUDIT TERMINÉ                      " -ForegroundColor Cyan
Write-Host "====================================================" -ForegroundColor Cyan

```
