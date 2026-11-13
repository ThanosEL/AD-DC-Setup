# Windows Active Directory Lab

## Table of Contents

1. [Lab Overview](#1-lab-overview)
2. [VMware Network Setup](#2-vmware-network-setup)
3. [DC01: Static IP & DNS](#3-dc01-static-ip--dns)
4. [Install AD DS & DNS, Promote to Domain Controller](#4-install-ad-ds--dns-promote-to-domain-controller)
5. [Create OUs, Groups & Users](#5-create-ous-groups--users)
6. [Enable SMB Share](#6-enable-smb-share)
7. [SRV01: Server Core Setup](#7-srv01-server-core-setup)
8. [WS01: Windows 11 Setup](#8-ws01-windows-11-setup)
9. [Join Machines to the Domain](#9-join-machines-to-the-domain)
10. [Post-Join Configuration](#10-post-join-configuration)
11. [Password Management](#11-password-management)

---

## 1. Lab Overview

```
              VMware NAT network (VMnet8)
                  192.168.177.0/24
                  Gateway: 192.168.177.2
      ┌──────────────────┼───────────────────┐
      │                  │                   │
┌─────┴─────┐     ┌──────┴─────┐      ┌──────┴─────┐
│   DC01    │     │    WS01    │      │   SRV01    │
│ Win 2025  │     │ Windows 11 │      │ Win 2025   │
│ Desktop   │     │ workstation│      │ Server Core│
│ AD + DNS  │     │            │      │ (CLI only) │
│   .10     │     │    .11     │      │    .12     │
└───────────┘     └────────────┘      └────────────┘
```

| Machine | Role | IP | OS |
|---------|------|----|----|
| DC01 | Domain Controller, DNS, SMB share | 192.168.177.10 | Windows Server 2025 Standard (Desktop Experience) |
| WS01 | Domain workstation | 192.168.177.11 | Windows 11 |
| SRV01 | Domain member server | 192.168.177.12 | Windows Server 2025 Standard (Server Core) |

**Domain:** `lab.local` | **NetBIOS:** `LAB` | **Gateway:** `192.168.177.2`

### Lab accounts (lab only)

| Machine | Account | Password |
|---------|---------|----------|
| DC01 / domain | `LAB\Administrator` (Domain Admin) | `Password1!` |
| DC01 / domain | `LAB\jdoe` (standard user) | `Password1!` |
| SRV01 | local user `thanos` | `Password123` |
| WS01 | local user | `Password1` |

### Server editions explained

- **Standard / Datacenter** is the *edition* (licensing). Identical for this lab.
- **Server Core / Desktop Experience** is the *installation option*. Core has no GUI, Desktop Experience has one.
- In the installer, the entry **without** "(Desktop Experience)" in its name is **Server Core**.

---

## 2. VMware Network Setup

```
Virtual Network Editor (run as Administrator) → Change Settings
  → select VMnet8 (NAT)
  → Subnet IP: 192.168.177.0 / 255.255.255.0
  → NAT Settings → Gateway IP: 192.168.177.2
```

---

## 3. DC01: Static IP & DNS

### Configure Static IP (DC01)

```powershell
# List adapters
Get-NetAdapter

# If the IP is DHCP-assigned, disable DHCP and remove the old config first
Set-NetIPInterface -InterfaceAlias "Ethernet0" -Dhcp Disabled
Remove-NetIPAddress -InterfaceAlias "Ethernet0" -AddressFamily IPv4 -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet0" -DestinationPrefix 0.0.0.0/0 -Confirm:$false -ErrorAction SilentlyContinue

# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.177.10 -PrefixLength 24 -DefaultGateway 192.168.177.2

# Point DNS to itself
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.177.10
```


### Rename Server

```powershell
Rename-Computer -NewName "DC01" -Restart
```

### Verify

```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet0" -AddressFamily IPv4
Test-Connection 8.8.8.8 -Count 2
```

---

## 4. Install AD DS & DNS, Promote to Domain Controller

### Method A: PowerShell (works on Server Core and Desktop Experience)

```powershell
# Install the roles
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools

# Create the new forest (the server reboots automatically)
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -InstallDns `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") `
  -Force
```

### Method B: Server Manager (GUI)

```
Manage → Add Roles and Features
  → Role-based or feature-based installation
  → Select the local server
  → Server Roles:
      ✔ Active Directory Domain Services → Add Features
      ✔ DNS Server → Add Features
  → Confirmation → Install
```

After installation, click **"Promote this server to a domain controller"**:

```
Deployment Configuration  → Add a new forest → Root domain: lab.local
Domain Controller Options → ✔ DNS Server  ✔ Global Catalog
                          → Set DSRM password
Additional Options        → NetBIOS name: LAB
Paths (defaults):
  Database folder  → C:\WINDOWS\NTDS
  Log files folder → C:\WINDOWS\NTDS
  SYSVOL           → C:\WINDOWS\SYSVOL
→ Install
```

After the reboot, log in as `LAB\Administrator`. Its password is the one the local Administrator had before the promotion.


### Post-promotion Setup & Verification

These commands only work **after** the promotion:

```powershell
# DNS forwarders (so the domain can still resolve internet names)
Set-DnsServerForwarder -IPAddress 8.8.8.8,1.1.1.1

# Health checks
Get-ADDomain
Get-ADDomainController
Get-DnsServerZone
dcdiag /v
```

---

## 5. Create OUs, Groups & Users

### Method A: GUI (Active Directory Users and Computers)

```
Server Manager → Tools → Active Directory Users and Computers
```

**1. Create OU**
```
Right-click "lab.local" → New → Organizational Unit
  Name: IT
  ✔ Protect container from accidental deletion
→ OK
```

**2. Create Group**
```
Right-click "OU=IT" → New → Group
  Group name:  IT-Admins
  Group scope: Global
  Group type:  Security
→ OK
```

**3. Create User**
```
Right-click "OU=IT" → New → User
  First name:      John
  Last name:       Doe
  User logon name: jdoe  (→ jdoe@lab.local)
→ Next
  Password:               Password1!
  ☐ User must change password at next logon
  ☑ Password never expires  (lab only)
→ Next → Finish
```

**4. Add User to Group**
```
Right-click "IT-Admins" → Properties → Members → Add
  Type: jdoe → Check Names → OK → OK
```

### Method B: PowerShell (also works on Server Core)

```powershell
# 1. OU
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"

# 2. Group
New-ADGroup -Name "IT-Admins" -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"

# 3. User
New-ADUser -Name "John Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Password1!" -AsPlainText -Force) `
  -Enabled $true `
  -PasswordNeverExpires $true

# 4. Add user to group
Add-ADGroupMember -Identity "IT-Admins" -Members "jdoe"

# 5. Verify
Get-ADUser -Identity jdoe -Properties MemberOf
Get-ADGroupMember -Identity "IT-Admins"
```

---

## 6. Enable SMB Share


### Method A: GUI

```
1. File Explorer → C:\ → New Folder → "Shares"; inside it → New Folder → "Shared"

2. Right-click "Shared" → Properties → Sharing → Advanced Sharing
     ✔ Share this folder
     Share name: Shared
     → Permissions → Everyone → Allow: Full Control → OK

3. Security tab → Edit → Add
     Type: Everyone → Check Names → OK
     ✔ Full Control → Allow → OK

4. (Optional) C:\Shares\Shared\notes.txt → "Backup admin password: Password1!"
```

### Method B: PowerShell

```powershell
New-Item -Path "C:\Shares\Shared" -ItemType Directory
New-SmbShare -Name "Shared" -Path "C:\Shares\Shared" -FullAccess "Everyone"
icacls "C:\Shares\Shared" /grant Everyone:F

# (Optional) bait file with a "leaked" credential
"Backup admin password: Password1!" | Out-File "C:\Shares\Shared\notes.txt"

# Verify
Get-SmbShare
Get-SmbShareAccess -Name "Shared"
```

### Verify from another machine

```powershell
Get-ChildItem \\DC01\Shared
net use Z: \\DC01\Shared
```

---

## 7. SRV01: Server Core Setup

SRV01 is a second Windows Server with **no GUI** (CLI only). It is set up fully here and joined to the domain in [section 9](#9-join-machines-to-the-domain).

### Install Server Core

```
VMware: New VM → Installer disc image (iso) → Windows Server 2025 ISO
        Network adapter: VMnet8
        Suggested: 2 CPU, 2-4 GB RAM, 60 GB disk

At the edition screen choose:
  ✔ Windows Server 2025 Standard Evaluation                         ← Server Core (CLI only)
  ✘ Windows Server 2025 Standard Evaluation (Desktop Experience)    ← has a GUI

Set the Administrator / user password
```

After login you get a command prompt and the **sconfig** menu:

```cmd
powershell      :: open PowerShell
sconfig         :: back to the text menu
```

### Configure Static IP & DNS (SRV01)

```powershell
Get-NetAdapter

# Disable DHCP and clear the old config
Set-NetIPInterface -InterfaceAlias "Ethernet0" -Dhcp Disabled
Remove-NetIPAddress -InterfaceAlias "Ethernet0" -AddressFamily IPv4 -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet0" -DestinationPrefix 0.0.0.0/0 -Confirm:$false -ErrorAction SilentlyContinue

# Static IP
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.177.12 -PrefixLength 24 -DefaultGateway 192.168.177.2

# DNS MUST point to the Domain Controller (needed for the domain join)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.177.10
```


### Change the Hostname

The default name (e.g. `Core-Server-1`) can be changed on its own, or together with the domain join in section 9.

```powershell
Rename-Computer -NewName "SRV01" -Restart
```

### Change a Local User Password

```powershell
net user thanos *
# or
Set-LocalUser -Name "thanos" -Password (Read-Host -AsSecureString "New password")
```

### Enable Ping (ICMP)

```powershell
Set-NetFirewallRule -Name FPS-ICMP4-ERQ-In -Enabled True -Profile Any
```

If the rule does not exist:

```powershell
New-NetFirewallRule -Name ICMPv4-In -DisplayName "Allow ICMPv4 Ping" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow -Profile Any
```

Disable again:

```powershell
Set-NetFirewallRule -Name FPS-ICMP4-ERQ-In -Enabled False
```

### Enable SSH (OpenSSH Server)

```powershell
# Install and start the service
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# Check the service, the firewall rule and the network profile
Get-Service sshd
Get-NetFirewallRule -Name *ssh* | Select Name, Enabled, Profile, Direction
Get-NetConnectionProfile
```

The default rule `OpenSSH-Server-In-TCP` applies only to the **Private** profile. A new network is **Public**, and after the domain join it becomes **Domain**, so in both cases SSH is blocked (`Connection reset`). Allow the rule on all profiles:

```powershell
Set-NetFirewallRule -Name OpenSSH-Server-In-TCP -Profile Any
```

(Optional) change the current network to Private:

```powershell
Set-NetConnectionProfile -InterfaceIndex 5 -NetworkCategory Private
```

Test from another machine:

```powershell
ssh thanos@192.168.177.12
```

### Verify Before Joining the Domain

```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet0" -AddressFamily IPv4
Get-DnsClientServerAddress -InterfaceAlias "Ethernet0" -AddressFamily IPv4
Resolve-DnsName lab.local                       # must return 192.168.177.10
Test-NetConnection 192.168.177.10 -Port 389     # TcpTestSucceeded : True
Test-Connection 192.168.177.10 -Count 2         # needs ICMP enabled on DC01
```

---

## 8. WS01: Windows 11 Setup

### Bypass Sign-in During ISO Installation

```powershell
# Disable adapter to skip the Microsoft account sign-in
netsh interface set interface "Ethernet0" admin=disable

# Re-enable after the local account is created
netsh interface set interface "Ethernet0" admin=enable

# Verify
Get-NetAdapter
ipconfig
```

Alternative: at the OOBE screen press `Shift+F10` and run `OOBE\BYPASSNRO` (the machine restarts and offers "I don't have internet").

### Configure Static IP & DNS (WS01)

```powershell
Get-NetAdapter

Set-NetIPInterface -InterfaceAlias "Ethernet0" -Dhcp Disabled
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.177.11 -PrefixLength 24 -DefaultGateway 192.168.177.2

# Point DNS to the DC
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.177.10
```

### Verify Before Joining

```powershell
nslookup lab.local
Test-NetConnection 192.168.177.10 -Port 389
```

---

## 9. Join Machines to the Domain

### SRV01 (Server Core)

Rename and join in one step:

```powershell
Add-Computer -DomainName "lab.local" -NewName "SRV01" -Credential LAB\Administrator -Restart
```

If the name is already `SRV01`:

```powershell
Add-Computer -DomainName "lab.local" -Credential LAB\Administrator -Restart
```

### WS01 (Windows 11)

```powershell
Add-Computer -DomainName "lab.local" -NewName "WS01" -Credential LAB\Administrator -Restart
```

### Test the credentials without joining

```powershell
net use \\192.168.177.10\IPC$ /user:LAB\Administrator *
net use \\192.168.177.10\IPC$ /delete
```

### Verify the join

On each machine:

```powershell
Get-ComputerInfo | Select-Object CsName, CsDomain, CsDomainRole
```

Expected: `lab.local`, and `MemberServer` (SRV01) or `MemberWorkstation` (WS01).

On DC01:

```powershell
Get-ADComputer -Filter * | Select Name, Enabled, DNSHostName
```

Expected output:

```
Name  Enabled DNSHostName
----  ------- -----------
DC01     True DC01.lab.local
WS01     True WS01.lab.local
SRV01    True SRV01.lab.local
```

The join creates the computer accounts automatically. Nothing else is required on the DC.

---

## 10. Post-Join Configuration

### Allow domain users to log in to SRV01

A normal domain user (`jdoe`) cannot log in to a member server by default. On SRV01:

```powershell
# Remote logon (RDP / interactive)
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "LAB\jdoe"

# PowerShell remoting / WinRM
Enable-PSRemoting -Force
Add-LocalGroupMember -Group "Remote Management Users" -Member "LAB\jdoe"
```

Test:

```powershell
Enter-PSSession -ComputerName SRV01 -Credential LAB\jdoe
```

### Firewall profile changes after the join

After the join the network profile becomes **Domain**. Make sure the rules are not limited to Private:

```powershell
Set-NetFirewallRule -Name OpenSSH-Server-In-TCP -Profile Any
Set-NetFirewallRule -Name FPS-ICMP4-ERQ-In -Enabled True -Profile Any
```

### (Optional) Move computers into an OU

New computers land in the default *Computers* container.

```powershell
Get-ADComputer SRV01 | Move-ADObject -TargetPath "OU=IT,DC=lab,DC=local"
```

### (Optional) Make a domain group local admin on SRV01

Creates a lateral-movement path for later practice:

```powershell
Add-LocalGroupMember -Group "Administrators" -Member "LAB\IT-Admins"
```

### Test from WS01

Log in as `LAB\jdoe` (choose *Other user*), then open `\\DC01\Shared`.

---

## 11. Password Management

### Reset a domain user's password (on DC01)

GUI:

```
Active Directory Users and Computers → right-click the user → Reset Password...
  ☑ Unlock the user's account (if locked)
```

PowerShell:

```powershell
Set-ADAccountPassword -Identity jdoe -Reset -NewPassword (ConvertTo-SecureString "Password1!" -AsPlainText -Force)
Unlock-ADAccount -Identity jdoe
```

The same works for the domain Administrator:

```powershell
Set-ADAccountPassword -Identity Administrator -Reset -NewPassword (ConvertTo-SecureString "Password1!" -AsPlainText -Force)
Unlock-ADAccount -Identity Administrator
```

### Reset a local user's password

On the machine itself, as an administrator:

```powershell
net user USERNAME NEWPASSWORD
# or
Set-LocalUser -Name "USERNAME" -Password (Read-Host -AsSecureString "New password")
```

A reset on DC01 does **not** change **local** accounts of a member machine.

If you forgot the password of a local account on a domain-joined machine, log in there with `LAB\Administrator` and reset it with `net user`. If you have no administrator access, restore a snapshot.