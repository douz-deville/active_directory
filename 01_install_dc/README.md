# 01 Installing the Domain Controller

1. Use `sconfig` to:
	- Change the hostname
	- Configure the server's IP address
	- Change the DNS server to our own IP address

2. Install the Active Directory Windows Feature

```shell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

3. Create a new forest

```shell
Install-ADDSForest -DomainName "xyz.com"
```

# 02 Domain Integration

1. Configure the workstation's DNS server to point to DC1

2. Join the Workstation to the Domain

```shell
Add-Computer -DomainName xyz.com -Credential (Get-Credential)

Restart-Computer
```
