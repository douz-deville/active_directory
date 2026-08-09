# 01 Installing the Domain Controller

1. Use `sconfig` to:
	- Change the hostname
	-Change the DNS server to our own IP address

2. Install the Active Directory Windows Feature

```shell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

```shell
Get-NetIPAddress
```

# Joining the Workstation to the Domain

```shell
Add-Computer -DomainName xyz.com -Credential (Get-Credential)

Restart-Computer
```
