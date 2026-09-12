# 09 - SYSTEM & NETWORK BASELINE

**1. Attack Surface Reduction (Roles & Software)**
Audit and cleanup to ensure that no unnecessary secondary components are installed.

```powershell
# Check installed roles (AD DS, DNS, File Services only)
Get-WindowsFeature | Where-Object { $_.Installed -and $_.FeatureType -eq "Role" -and $_.Name -notin @("AD-Domain-Services", "DNS", "FileAndStorage-Services", "Core-File-Server") }

# Third-party software inventory
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*, HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* -ErrorAction SilentlyContinue | Where-Object { $_.DisplayName } | Select-Object DisplayName, DisplayVersion, Publisher

```

**2. Disabling Legacy Services & Protocols**
Disable the Print Spooler, WebClient, Remote Registry, NetBIOS, LLMNR, and SMBv1.

```powershell
# Non-essential services
Stop-Service -Name Spooler, WebClient, RemoteRegistry, LmHosts -ErrorAction SilentlyContinue
Set-Service -Name Spooler, WebClient, RemoteRegistry, LmHosts -StartupType Disabled -ErrorAction SilentlyContinue

# LLMNR
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT" -Name DNSClient -Force -ErrorAction SilentlyContinue
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name "EnableMulticast" -Value 0 -Type DWORD -Force

# NetBIOS (NBT-NS) on all network adapters
Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true } | ForEach-Object { $_.SetTcpipNetbios(2) }

# SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force

```

**3. Authentication & Integrity Hardening**
Enforce network signing and strict NTLMv2 authentication.

```powershell
# Mandatory SMB signing
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force

# LDAP Integrity (2 = Required) & Channel Binding (2 = Always)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "LDAPServerIntegrity" -Value 2 -Type DWORD -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "LdapEnforceChannelBinding" -Value 2 -Type DWORD -Force

# NTLMv2 only — strict mode
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "LmCompatibilityLevel" -Value 5 -Type DWORD -Force

```

**4. Windows Defender Firewall Configuration**
Enable all firewall profiles, systematically block inbound traffic, isolate outbound public traffic, and allow domain/private outbound traffic to guarantee replication stability.

```powershell
# Enable profiles and block inbound traffic
Set-NetFirewallProfile -Profile Domain, Private, Public -Enabled True
Set-NetFirewallProfile -Profile Domain, Private, Public -DefaultInboundAction Block

# Outbound filtering (Block Public, Allow Domain & Private)
Set-NetFirewallProfile -Profile Public -DefaultOutboundAction Block
Set-NetFirewallProfile -Profile Domain, Private -DefaultOutboundAction Allow

# Allow native AD DS traffic
$RuleGroups = @("Active Directory Domain Services", "DNS Service", "Core Networking", "File and Printer Sharing")
foreach ($Group in $RuleGroups) { Enable-NetFirewallRule -DisplayGroup $Group -ErrorAction SilentlyContinue }

# Explicit Outbound AD Rule for Private profile fallback
New-NetFirewallRule -DisplayName "AD-Outbound-Essential" -Direction Outbound -Protocol TCP -RemotePort 53,88,135,389,445,3268,49152-65535 -Action Allow -Profile Private -ErrorAction SilentlyContinue

```

**5. Final Validated Audit Script (`Audit-Baseline.ps1`)**

```powershell
Write-Host "====================================================" -ForegroundColor Cyan
Write-Host "    TIER 0 BASELINE HARDENING COMPLIANCE AUDIT      " -ForegroundColor Cyan
Write-Host "====================================================" -ForegroundColor Cyan

# 1. Services
Write-Host "`n[1/4] Checking Sensitive Services..." -ForegroundColor Yellow
$Services = Get-Service Spooler, WebClient, RemoteRegistry, LmHosts -ErrorAction SilentlyContinue
foreach ($svc in $Services) {
    $isOK = ($svc.Status -eq 'Stopped' -or $svc.StartType -eq 'Disabled')
    $status = if ($isOK) { "PASS" } else { "FAIL" }
    $color = if ($isOK) { "Green" } else { "Red" }
    Write-Host "  - Service $($svc.Name) ($($svc.Status)/$($svc.StartType)) : [$status]" -ForegroundColor $color
}

# 2. SMB & Firewall
Write-Host "`n[2/4] Checking Network Configuration (SMB & Firewall)..." -ForegroundColor Yellow
$Smb = Get-SmbServerConfiguration

$smb1Pass = (-not $Smb.EnableSMB1Protocol)
$smb1Status = if ($smb1Pass) { "PASS" } else { "FAIL" }
$smb1Color = if ($smb1Pass) { "Green" } else { "Red" }
Write-Host "  - SMBv1 Disabled : [$smb1Status]" -ForegroundColor $smb1Color

$smbSigPass = $Smb.RequireSecuritySignature
$smbSigStatus = if ($smbSigPass) { "PASS" } else { "FAIL" }
$smbSigColor = if ($smbSigPass) { "Green" } else { "Red" }
Write-Host "  - Mandatory SMB Signing : [$smbSigStatus]" -ForegroundColor $smbSigColor

$FwProfiles = Get-NetFirewallProfile
foreach ($p in $FwProfiles) {
    $outExpected = if ($p.Name -eq 'Public') { 'Block' } else { 'Allow' }
    $isOK = (($p.Enabled -eq $true) -and ($p.DefaultInboundAction -eq 'Block') -and ($p.DefaultOutboundAction -eq $outExpected))
    $status = if ($isOK) { "PASS" } else { "FAIL" }
    $color = if ($isOK) { "Green" } else { "Red" }
    Write-Host "  - Firewall Profile $($p.Name) (In:$($p.DefaultInboundAction) / Out:$($p.DefaultOutboundAction)) : [$status]" -ForegroundColor $color
}

# 3. Registry (LLMNR, NTLM, LDAP)
Write-Host "`n[3/4] Checking Registry Configuration..." -ForegroundColor Yellow

$llmnr = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\DNSClient" -Name EnableMulticast -ErrorAction SilentlyContinue).EnableMulticast
$llmnrPass = ($llmnr -eq 0)
$llmnrStatus = if ($llmnrPass) { "PASS" } else { "FAIL" }
$llmnrColor = if ($llmnrPass) { "Green" } else { "Red" }
Write-Host "  - LLMNR Disabled (EnableMulticast=0) : [$llmnrStatus]" -ForegroundColor $llmnrColor

$lsa = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name LmCompatibilityLevel -ErrorAction SilentlyContinue).LmCompatibilityLevel
$lsaPass = ($lsa -eq 5)
$lsaStatus = if ($lsaPass) { "PASS" } else { "FAIL" }
$lsaColor = if ($lsaPass) { "Green" } else { "Red" }
Write-Host "  - Strict NTLMv2 (LmCompatibilityLevel=5) : [$lsaStatus]" -ForegroundColor $lsaColor

$ldapInt = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name LDAPServerIntegrity -ErrorAction SilentlyContinue).LDAPServerIntegrity
$ldapIntPass = ($ldapInt -eq 2)
$ldapIntStatus = if ($ldapIntPass) { "PASS" } else { "FAIL" }
$ldapIntColor = if ($ldapIntPass) { "Green" } else { "Red" }
Write-Host "  - Mandatory LDAP Signing (LDAPServerIntegrity=2) : [$ldapIntStatus]" -ForegroundColor $ldapIntColor

$ldapCb = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name LdapEnforceChannelBinding -ErrorAction SilentlyContinue).LdapEnforceChannelBinding
$ldapCbPass = ($ldapCb -eq 2)
$ldapCbStatus = if ($ldapCbPass) { "PASS" } else { "FAIL" }
$ldapCbColor = if ($ldapCbPass) { "Green" } else { "Red" }
Write-Host "  - Mandatory LDAP Channel Binding (LdapEnforceChannelBinding=2) : [$ldapCbStatus]" -ForegroundColor $ldapCbColor

# 4. Unnecessary Windows Roles
Write-Host "`n[4/4] Checking Attack Surface (Windows Roles)..." -ForegroundColor Yellow
$extraRoles = Get-WindowsFeature | Where-Object { $_.Installed -and $_.FeatureType -eq "Role" -and $_.Name -notin @("AD-Domain-Services", "DNS", "FileAndStorage-Services", "Core-File-Server") }
$rolesPass = ($null -eq $extraRoles -or $extraRoles.Count -eq 0)
$rolesStatus = if ($rolesPass) { "PASS" } else { "FAIL" }
$rolesColor = if ($rolesPass) { "Green" } else { "Red" }
Write-Host "  - No non-essential roles installed : [$rolesStatus]" -ForegroundColor $rolesColor

Write-Host "`n====================================================" -ForegroundColor Cyan
Write-Host "                AUDIT COMPLETE                      " -ForegroundColor Cyan
Write-Host "====================================================" -ForegroundColor Cyan

```
