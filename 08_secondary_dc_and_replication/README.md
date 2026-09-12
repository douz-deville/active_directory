# 08 - Promotion of the Second Domain Controller (DC2) and AD DS Replication

## 1. Context & Network Architecture

The objective of this step is to ensure high availability of the Active Directory directory service and DNS service for the `xyz.com` domain by adding a second domain controller (`DC2`).

### IP Addressing Topology

* **DC1 (Primary DC):** `192.168.197.10/24`
* **DC2 (Secondary DC):** `192.168.197.11/24`
* **Active Directory Domain:** `xyz.com`


## 2. Prerequisites & DC2 Preparation

Before the promotion, the network configuration of DC2 was validated:

1. **Static IP address:** `192.168.197.11`
2. **Initial Primary DNS:** `192.168.197.11` (Loopback)
3. **Initial Secondary DNS:** `192.168.197.10` (Points to DC1)
4. **Target Computer Name:** `DC2`


## 3. Installation and Promotion

### 3.1 Installing the Roles via PowerShell

```powershell
Install-WindowsFeature -Name AD-Domain-Services, DNS -IncludeManagementTools
```

### 3.2 Promoting the Server to a Replicated Domain Controller

```powershell
Import-Module ADDSDeployment
Install-ADDSDomainController `
    -NoGlobalCatalog:$false `
    -CreateDnsDelegation:$false `
    -Credential (Get-Credential) `
    -DomainName "xyz.com" `
    -InstallDns:$true `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -Force:$true
```


## 4. DNS Redundancy Configuration (Cross-Configuration)

To avoid **DNS Islanding** issues (where a domain controller may become isolated in the event of a restart), the DNS server addresses were configured across both domain controllers:

| Server                     | Primary DNS            | Secondary DNS          |
| :------------------------- | :--------------------- | :--------------------- |
| **DC1** (`192.168.197.10`) | `192.168.197.11` (DC2) | `192.168.197.10` (DC1) |
| **DC2** (`192.168.197.11`) | `192.168.197.10` (DC1) | `192.168.197.11` (DC2) |

### PowerShell Configuration Command

```powershell
Set-DnsClientServerAddress -InterfaceIndex 4 -ServerAddresses ("192.168.197.10", "192.168.197.11")
```
