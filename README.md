# Windows Active Directory — Domain Controller Setup

---

## Table of Contents

- [Static IP & DNS](#1-static-ip--dns)
- [Install AD DS & DNS Roles](#2-install-ad-ds--dns-roles)
- [Add Windows PCs to Domain](#3-add-windows-pcs-to-domain)
- [Create OUs, Groups & Users](#4-create-ous-groups--users)
- [Enable SMB Share](#5-enable-smb-share)

---

## 1. Static IP & DNS

### Configure Static IP (DC01)

```powershell
# List adapters
Get-NetAdapter

# If IP is DHCP-assigned, remove it first
Remove-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.177.136 -Confirm:$false

# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.177.10 -PrefixLength 24 -DefaultGateway 192.168.177.2

# Point DNS to itself (loopback)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1

# (Optional) Set DNS forwarders
Set-DnsServerForwarder -IPAddress 8.8.8.8,1.1.1.1
```

### Rename Server (Optional)

```powershell
Rename-Computer -NewName "DC01" -Restart
```

### Verification & License

```powershell
dcdiag /v
Get-ADDomain
```

```powershell
slmgr /rearm    # Run before 180 days expire — up to 6 times
slmgr /dlv      # Check remaining activation time
```

---

## 2. Install AD DS & DNS Roles

### Via Server Manager (GUI)

```
Manage → Add Roles and Features
  → Role-based or feature-based installation
  → Select the local server
  → Server Roles:
      ✔ Active Directory Domain Services → Add Features
      ✔ DNS Server → Add Features
  → Confirmation → Install
```

### Promote to Domain Controller

After installation, click **"Promote this server to a domain controller"**:

```
Deployment Configuration  → Add a new forest → Root domain: lab.local
Domain Controller Options → ✔ DNS Server  ✔ Global Catalog
                          → Set DSRM password (e.g. Password1!)
Additional Options        → NetBIOS name: LAB
Paths (defaults):
  Database folder  → C:\WINDOWS\NTDS
  Log files folder → C:\WINDOWS\NTDS
  SYSVOL           → C:\WINDOWS\SYSVOL
→ Install
```

---

## 3. Add Windows PCs to Domain

### Bypass Sign-in During ISO Installation

```powershell
# Disable adapter to skip Microsoft account sign-in
netsh interface set interface "Ethernet0" admin=disable

# Re-enable after local account is created
netsh interface set interface "Ethernet0" admin=enable

# Verify
Get-NetAdapter
ipconfig
```

### Configure Static IP & DNS on Workstation (WS01)

```powershell
Get-NetAdapter

New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.177.11 -PrefixLength 24 -DefaultGateway 192.168.177.2

# Point DNS to DC IP
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.177.10

Rename-Computer -NewName "WS01" -Restart
```

### Join Domain

```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

> Credentials prompt: `LAB\Administrator` + DC password

### Verify Domain Join

```powershell
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole
```

---

## 4. Create OUs, Groups & Users

### Method A — GUI (Active Directory Users and Computers)

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

---

### Method B — PowerShell

**1. Create OU**
```powershell
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"
```

**2. Create Group**
```powershell
New-ADGroup -Name "IT-Admins" -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
```

**3. Create User**
```powershell
New-ADUser -Name "John Doe" `
  -SamAccountName "jdoe" `
  -UserPrincipalName "jdoe@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Password1!" -AsPlainText -Force) `
  -Enabled $true `
  -PasswordNeverExpires $true
```

**4. Add User to Group**
```powershell
Add-ADGroupMember -Identity "IT-Admins" -Members "jdoe"
```

**5. Verify**
```powershell
Get-ADUser -Identity jdoe -Properties MemberOf
Get-ADGroupMember -Identity "IT-Admins"
```

### Verification (on client WS01 logged in as `LAB\jdoe`)

```cmd
whoami
whoami /groups
```

---

## 5. Enable SMB Share

### Method A — GUI

**1. Create folder**
```
File Explorer → C:\ → New Folder → "Shares"
Inside C:\Shares → New Folder → "Shared"
```

**2. Enable sharing**
```
Right-click "Shared" → Properties → Sharing → Advanced Sharing
  ✔ Share this folder
  Share name: Shared
  → Permissions
```

**3. Share permissions**
```
Select "Everyone" → Allow: Full Control → OK
```

**4. NTFS permissions**
```
Security tab → Edit → Add
  Type: Everyone → Check Names → OK
  ✔ Full Control → Allow → OK
```

**5. (Optional) Add test file**
```
C:\Shares\Shared\notes.txt → content: "Backup admin password: Password1!"
```

**6. Verify from WS01**
```
File Explorer → \\DC01\Shared
           or → \\192.168.177.10\Shared
```

---

### Method B — PowerShell

**1. Create folder**
```powershell
New-Item -Path "C:\Shares\Shared" -ItemType Directory
```

**2. Create SMB share**
```powershell
New-SmbShare -Name "Shared" -Path "C:\Shares\Shared" -FullAccess "Everyone"
```

**3. NTFS permissions**
```powershell
icacls "C:\Shares\Shared" /grant Everyone:F
```

**4. (Optional) Add test file**
```powershell
"Backup admin password: Password1!" | Out-File "C:\Shares\Shared\notes.txt"
```

**5. Verify locally**
```powershell
Get-SmbShare
Get-SmbShareAccess -Name "Shared"
```

**6. Verify from WS01**
```powershell
Get-ChildItem \\DC01\Shared

# Or map as drive
net use Z: \\DC01\Shared
```
