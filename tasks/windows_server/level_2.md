# Windows Server — Level 2 (🟡 Medium)

## 1. Active Directory Domain Services (AD DS)

> **Objective:** Install AD DS, promote the server to a Domain Controller, create organizational units, users, groups, and configure basic Group Policy.

### 1.1 Install the AD DS Role

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

### 1.2 Promote to Domain Controller (New Forest)

```powershell
# Import the ADDSDeployment module
Import-Module ADDSDeployment

# Create a new forest
Install-ADDSForest `
    -DomainName "corp.local" `
    -DomainNetbiosName "CORP" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns:$true `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "DSRM-P@ss2026!" -AsPlainText -Force) `
    -Force:$true
```

> ⚠️ The server will reboot automatically after promotion. After reboot, log in as `CORP\Administrator`.

### 1.3 Create Organizational Units (OUs)

```powershell
# Create top-level OUs
New-ADOrganizationalUnit -Name "IT" -Path "DC=corp,DC=local"
New-ADOrganizationalUnit -Name "HR" -Path "DC=corp,DC=local"
New-ADOrganizationalUnit -Name "Finance" -Path "DC=corp,DC=local"

# Create a nested OU
New-ADOrganizationalUnit -Name "Servers" -Path "OU=IT,DC=corp,DC=local"
New-ADOrganizationalUnit -Name "Workstations" -Path "OU=IT,DC=corp,DC=local"
```

### 1.4 Create AD Users

```powershell
# Create a single user
New-ADUser -Name "John Smith" `
    -GivenName "John" -Surname "Smith" `
    -SamAccountName "jsmith" `
    -UserPrincipalName "jsmith@corp.local" `
    -Path "OU=IT,DC=corp,DC=local" `
    -AccountPassword (ConvertTo-SecureString "User-P@ss2026!" -AsPlainText -Force) `
    -Enabled $true `
    -ChangePasswordAtLogon $true
```

**Bulk user creation from CSV:**

```powershell
# users.csv format: FirstName,LastName,Username,OU
# John,Smith,jsmith,IT
# Jane,Doe,jdoe,HR

Import-Csv "C:\Scripts\users.csv" | ForEach-Object {
    New-ADUser -Name "$($_.FirstName) $($_.LastName)" `
        -GivenName $_.FirstName -Surname $_.LastName `
        -SamAccountName $_.Username `
        -UserPrincipalName "$($_.Username)@corp.local" `
        -Path "OU=$($_.OU),DC=corp,DC=local" `
        -AccountPassword (ConvertTo-SecureString "Welcome2026!" -AsPlainText -Force) `
        -Enabled $true -ChangePasswordAtLogon $true
}
```

### 1.5 Create AD Groups

```powershell
# Create a Global Security group
New-ADGroup -Name "IT-Admins" `
    -GroupScope Global `
    -GroupCategory Security `
    -Path "OU=IT,DC=corp,DC=local" `
    -Description "IT Administrators group"

# Add users to the group
Add-ADGroupMember -Identity "IT-Admins" -Members "jsmith"

# View group members
Get-ADGroupMember -Identity "IT-Admins"
```

### 1.6 Group Policy Basics

```powershell
# Import the GroupPolicy module
Import-Module GroupPolicy

# Create a new GPO
New-GPO -Name "IT-SecurityPolicy" -Comment "Security settings for IT OU"

# Link the GPO to an OU
New-GPLink -Name "IT-SecurityPolicy" -Target "OU=IT,DC=corp,DC=local"

# View all GPOs
Get-GPO -All | Format-Table DisplayName, GpoStatus

# Generate a GPO report
Get-GPOReport -Name "IT-SecurityPolicy" -ReportType Html -Path "C:\Reports\GPOReport.html"
```

### Useful AD PowerShell Commands

| Command | Description |
|---------|-------------|
| `Get-ADUser -Filter *` | List all AD users |
| `Get-ADUser -Identity jsmith -Properties *` | Detailed user info |
| `Set-ADUser -Identity jsmith -Title "SysAdmin"` | Modify a user attribute |
| `Unlock-ADAccount -Identity jsmith` | Unlock a locked account |
| `Search-ADAccount -LockedOut` | Find locked accounts |
| `Search-ADAccount -AccountDisabled` | Find disabled accounts |
| `Get-ADGroup -Filter *` | List all AD groups |
| `Get-ADOrganizationalUnit -Filter *` | List all OUs |
| `Get-ADDomainController` | Get DC information |
| `Get-ADForest` | Get forest information |

---

---

## 2. DNS Server Configuration

> **Objective:** Install the DNS role, create zones, manage records, configure forwarders, and troubleshoot DNS.

### 2.1 Install the DNS Role

```powershell
Install-WindowsFeature DNS -IncludeManagementTools
```

> 💡 If you promoted a Domain Controller with `-InstallDns:$true`, DNS is already installed.

### 2.2 Create a Forward Lookup Zone

```powershell
# Primary zone
Add-DnsServerPrimaryZone -Name "corp.local" -ZoneFile "corp.local.dns"

# AD-integrated zone (if on a Domain Controller)
Add-DnsServerPrimaryZone -Name "app.corp.local" -ReplicationScope "Domain"
```

### 2.3 Create a Reverse Lookup Zone

```powershell
# For the 192.168.1.0/24 subnet
Add-DnsServerPrimaryZone -NetworkId "192.168.1.0/24" -ReplicationScope "Domain"
```

### 2.4 Add DNS Records

```powershell
# A Record (host to IP)
Add-DnsServerResourceRecordA -Name "webserver" -ZoneName "corp.local" `
    -IPv4Address "192.168.1.50"

# CNAME Record (alias)
Add-DnsServerResourceRecordCName -Name "www" -ZoneName "corp.local" `
    -HostNameAlias "webserver.corp.local"

# MX Record (mail exchange)
Add-DnsServerResourceRecordMX -Name "." -ZoneName "corp.local" `
    -MailExchange "mail.corp.local" -Preference 10

# TXT Record (SPF, verification, etc.)
Add-DnsServerResourceRecord -ZoneName "corp.local" -Name "." `
    -Txt -DescriptiveText "v=spf1 mx -all"

# PTR Record (reverse lookup)
Add-DnsServerResourceRecordPtr -Name "50" -ZoneName "1.168.192.in-addr.arpa" `
    -PtrDomainName "webserver.corp.local"
```

### 2.5 Configure Forwarders

```powershell
# Add external DNS forwarders
Add-DnsServerForwarder -IPAddress 8.8.8.8
Add-DnsServerForwarder -IPAddress 1.1.1.1

# View current forwarders
Get-DnsServerForwarder
```

### 2.6 DNS Troubleshooting

```powershell
# nslookup (classic)
nslookup webserver.corp.local
nslookup 192.168.1.50

# Resolve-DnsName (modern PowerShell)
Resolve-DnsName -Name "webserver.corp.local" -Type A
Resolve-DnsName -Name "corp.local" -Type MX

# Clear DNS client cache
Clear-DnsClientCache

# View DNS client cache
Get-DnsClientCache

# Test DNS server directly
Resolve-DnsName -Name "webserver.corp.local" -Server 192.168.1.10
```

### Common DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| **A** | Maps hostname to IPv4 address | `webserver → 192.168.1.50` |
| **AAAA** | Maps hostname to IPv6 address | `webserver → 2001:db8::50` |
| **CNAME** | Alias for another hostname | `www → webserver.corp.local` |
| **MX** | Mail exchange server | `corp.local → mail.corp.local (priority 10)` |
| **TXT** | Text data (SPF, DKIM, etc.) | `v=spf1 mx -all` |
| **NS** | Name server for a zone | `corp.local → dc01.corp.local` |
| **PTR** | Reverse lookup (IP to hostname) | `192.168.1.50 → webserver.corp.local` |
| **SRV** | Service locator | `_ldap._tcp.corp.local` |
| **SOA** | Start of Authority | Zone metadata (serial, refresh, etc.) |

---

---

## 3. DHCP Server Configuration

> **Objective:** Install DHCP, create and configure scopes, manage reservations and exclusions, and authorize the server.

### 3.1 Install the DHCP Role

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

### 3.2 Authorize DHCP Server in Active Directory

```powershell
# Required if the server is domain-joined
Add-DhcpServerInDC -DnsName "dc01.corp.local" -IPAddress 192.168.1.10

# Verify authorization
Get-DhcpServerInDC
```

### 3.3 Create a DHCP Scope

```powershell
# Create an IPv4 scope
Add-DhcpServerv4Scope -Name "LAN-Scope" `
    -StartRange 192.168.1.100 `
    -EndRange 192.168.1.200 `
    -SubnetMask 255.255.255.0 `
    -LeaseDuration (New-TimeSpan -Days 8) `
    -State Active
```

### 3.4 Set Scope Options

```powershell
# Set default gateway (router)
Set-DhcpServerv4OptionValue -ScopeId 192.168.1.0 `
    -OptionId 3 -Value 192.168.1.1

# Set DNS servers
Set-DhcpServerv4OptionValue -ScopeId 192.168.1.0 `
    -OptionId 6 -Value 192.168.1.10, 8.8.8.8

# Set DNS domain name
Set-DhcpServerv4OptionValue -ScopeId 192.168.1.0 `
    -OptionId 15 -Value "corp.local"
```

| Option ID | Name | Description |
|-----------|------|-------------|
| 3 | Router | Default gateway |
| 6 | DNS Servers | DNS server addresses |
| 15 | DNS Domain Name | Domain suffix for clients |
| 44 | WINS/NBNS Servers | WINS server addresses |
| 46 | WINS/NBT Node Type | NetBIOS node type |

### 3.5 Create Reservations

```powershell
# Reserve an IP for a specific device by MAC address
Add-DhcpServerv4Reservation -ScopeId 192.168.1.0 `
    -IPAddress 192.168.1.150 `
    -ClientId "00-15-5D-01-02-03" `
    -Name "PrintServer" `
    -Description "Office printer reservation"
```

### 3.6 Create Exclusion Ranges

```powershell
# Exclude addresses from being assigned
Add-DhcpServerv4ExclusionRange -ScopeId 192.168.1.0 `
    -StartRange 192.168.1.100 -EndRange 192.168.1.110
```

### 3.7 Verify DHCP Leases

```powershell
# View active leases
Get-DhcpServerv4Lease -ScopeId 192.168.1.0

# View scope statistics
Get-DhcpServerv4ScopeStatistics -ScopeId 192.168.1.0
```

### 3.8 Backup and Restore DHCP

```powershell
# Backup DHCP configuration
Backup-DhcpServer -Path "C:\DHCPBackup"

# Restore DHCP configuration
Restore-DhcpServer -Path "C:\DHCPBackup"

# Export full DHCP server config (portable XML)
Export-DhcpServer -File "C:\DHCPBackup\dhcpexport.xml" -Leases

# Import on another server
Import-DhcpServer -File "C:\DHCPBackup\dhcpexport.xml" -BackupPath "C:\DHCPBackup\BeforeImport" -Leases
```

---

---

## 4. Windows Server Backup

> **Objective:** Install the backup feature, configure backup policies, perform backups, and understand recovery options.

### 4.1 Install Windows Server Backup

```powershell
Install-WindowsFeature Windows-Server-Backup -IncludeManagementTools
```

### 4.2 Perform a One-Time Backup with `wbadmin`

```powershell
# Backup the C: drive to E: (backup destination)
wbadmin start backup -backupTarget:E: -include:C: -quiet

# Backup specific folders
wbadmin start backup -backupTarget:E: -include:C:\inetpub,C:\Scripts -quiet

# System State backup
wbadmin start systemstatebackup -backupTarget:E: -quiet
```

### 4.3 Schedule Automated Backups

```powershell
# Using wbadmin policy (daily backup at 9 PM)
$Policy = New-WBPolicy
$BackupTarget = New-WBBackupTarget -VolumePath "E:"
Add-WBBackupTarget -Policy $Policy -Target $BackupTarget

# Add volumes to back up
$Volume = Get-WBVolume -VolumePath "C:"
Add-WBVolume -Policy $Policy -Volume $Volume

# Add system state
Add-WBSystemState -Policy $Policy

# Set the schedule (9 PM daily)
Set-WBSchedule -Policy $Policy -Schedule 21:00

# Apply the policy
Set-WBPolicy -Policy $Policy -Force
```

### 4.4 View Backup Status and History

```powershell
# Current backup policy
Get-WBPolicy

# Last backup summary
Get-WBSummary

# Backup job history
Get-WBJob -Previous 5
wbadmin get versions
```

### 4.5 Restore Files from Backup

```powershell
# List available backup versions
wbadmin get versions

# Restore specific files/folders
wbadmin start recovery -version:03/01/2026-21:00 `
    -itemType:File -items:C:\inetpub\wwwroot `
    -recoveryTarget:C:\Restore -quiet
```

### 4.6 Bare Metal Recovery Overview

1. Boot from Windows Server 2022 installation media.
2. Select **Repair your computer** → **Troubleshoot** → **System Image Recovery**.
3. Select the backup location (external drive / network share).
4. Follow the wizard to restore the entire server.

### Backup Best Practices

| Practice | Recommendation |
|----------|----------------|
| **Frequency** | Daily incremental, weekly full |
| **Retention** | Keep at least 30 days of backups |
| **3-2-1 Rule** | 3 copies, 2 different media, 1 offsite |
| **Test Restores** | Quarterly restore tests |
| **System State** | Always include in DC backups |
| **Monitoring** | Check backup logs daily |
| **Documentation** | Document backup schedule & procedures |

---

---

## 5. Remote Server Management

> **Objective:** Configure WinRM, use PowerShell Remoting, manage multiple servers remotely, and leverage RSAT tools.

### 5.1 Enable WinRM

```powershell
# Enable PowerShell Remoting (run on the remote server)
Enable-PSRemoting -Force

# Verify WinRM is running
Get-Service WinRM
Test-WSMan
```

### 5.2 Configure TrustedHosts (Workgroup Environments)

```powershell
# View current TrustedHosts
Get-Item WSMan:\localhost\Client\TrustedHosts

# Add a specific server
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "192.168.1.20" -Force

# Add multiple servers
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "192.168.1.20,192.168.1.21,WEB-SVR-01" -Force

# Trust all hosts (not recommended for production)
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
```

### 5.3 Interactive Remote Session

```powershell
# Enter an interactive session on a remote server
$Cred = Get-Credential
Enter-PSSession -ComputerName "WEB-SVR-01" -Credential $Cred

# You are now on the remote server — run commands as if local
hostname
Get-Service W3SVC
exit   # Exit the remote session
```

### 5.4 Run Commands on Remote Servers

```powershell
# Run a single command remotely
Invoke-Command -ComputerName "WEB-SVR-01" -Credential $Cred -ScriptBlock {
    Get-Service W3SVC
}

# Run a script file remotely
Invoke-Command -ComputerName "WEB-SVR-01" -Credential $Cred -FilePath "C:\Scripts\healthcheck.ps1"
```

### 5.5 Remoting to Multiple Servers

```powershell
$Servers = @("WEB-SVR-01", "WEB-SVR-02", "DB-SVR-01")
$Cred    = Get-Credential

# Check uptime on all servers simultaneously
Invoke-Command -ComputerName $Servers -Credential $Cred -ScriptBlock {
    $OS = Get-CimInstance Win32_OperatingSystem
    [PSCustomObject]@{
        Server = $env:COMPUTERNAME
        Uptime = (Get-Date) - $OS.LastBootUpTime
        FreeMemGB = [math]::Round($OS.FreePhysicalMemory / 1MB, 2)
    }
} | Format-Table Server, Uptime, FreeMemGB
```

### 5.6 Persistent Sessions

```powershell
# Create persistent sessions for efficiency
$Sessions = New-PSSession -ComputerName $Servers -Credential $Cred

# Run commands on all sessions
Invoke-Command -Session $Sessions -ScriptBlock { Get-WindowsFeature | Where-Object Installed }

# Copy files to remote servers
Copy-Item -Path "C:\Scripts\deploy.ps1" -Destination "C:\Scripts\" -ToSession $Sessions[0]

# Clean up sessions
Remove-PSSession $Sessions
```

### 5.7 RSAT Tools Overview

Remote Server Administration Tools allow management from a workstation:

```powershell
# Install RSAT tools on Windows 10/11 workstation
Get-WindowsCapability -Online -Name "RSAT*" | Add-WindowsCapability -Online

# Install specific RSAT tools
Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.Dns.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.DHCP.Tools~~~~0.0.1.0"
Add-WindowsCapability -Online -Name "Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0"
```

| RSAT Tool | Capability Name | Manages |
|-----------|----------------|---------|
| AD DS Tools | `Rsat.ActiveDirectory.DS-LDS.Tools` | Users, Groups, OUs, AD |
| DNS Tools | `Rsat.Dns.Tools` | DNS Zones & Records |
| DHCP Tools | `Rsat.DHCP.Tools` | DHCP Scopes & Leases |
| Group Policy | `Rsat.GroupPolicy.Management.Tools` | GPOs |
| Server Manager | `Rsat.ServerManager.Tools` | Remote Server Manager |
| Hyper-V Tools | `Rsat.HyperV.Tools` | Virtual Machines |

---

---

## 6. Disk Management and Storage

> **Objective:** Manage physical disks, partitions, volumes, and explore Storage Spaces using PowerShell.

### 6.1 View Disks and Volumes

```powershell
# List all physical disks
Get-Disk

# List all volumes
Get-Volume

# List all partitions
Get-Partition

# Detailed disk info
Get-Disk | Format-Table Number, FriendlyName, Size, PartitionStyle, OperationalStatus
```

### 6.2 Initialize a New Disk

```powershell
# Initialize a new disk as GPT (recommended)
Initialize-Disk -Number 1 -PartitionStyle GPT

# Or as MBR (for legacy compatibility)
Initialize-Disk -Number 1 -PartitionStyle MBR
```

### 6.3 Create a Partition

```powershell
# Create a partition using all available space
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter

# Create a partition with a specific size and drive letter
New-Partition -DiskNumber 1 -Size 50GB -DriveLetter D
```

### 6.4 Format a Volume

```powershell
# Format with NTFS
Format-Volume -DriveLetter D -FileSystem NTFS -NewFileSystemLabel "Data" -Confirm:$false

# Format with ReFS (Resilient File System)
Format-Volume -DriveLetter E -FileSystem ReFS -NewFileSystemLabel "Storage" -Confirm:$false
```

### 6.5 Full Workflow: New Disk → Ready Volume

```powershell
# Complete workflow for a new disk
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -DriveLetter D
Format-Volume -DriveLetter D -FileSystem NTFS -NewFileSystemLabel "AppData" -Confirm:$false
```

### 6.6 Extend / Shrink Volumes

```powershell
# Extend a partition to fill available space
$MaxSize = (Get-PartitionSupportedSize -DriveLetter D).SizeMax
Resize-Partition -DriveLetter D -Size $MaxSize

# Shrink a volume by 10 GB
$CurrentSize = (Get-Partition -DriveLetter D).Size
Resize-Partition -DriveLetter D -Size ($CurrentSize - 10GB)
```

### 6.7 Storage Spaces Basics

Storage Spaces pools multiple physical disks into a single virtual disk:

```powershell
# View available physical disks for pooling
Get-PhysicalDisk -CanPool $true

# Create a Storage Pool
New-StoragePool -FriendlyName "DataPool" `
    -StorageSubSystemFriendlyName "Windows Storage*" `
    -PhysicalDisks (Get-PhysicalDisk -CanPool $true)

# Create a Virtual Disk (mirrored for redundancy)
New-VirtualDisk -StoragePoolFriendlyName "DataPool" `
    -FriendlyName "MirroredDisk" `
    -ResiliencySettingName Mirror `
    -Size 100GB

# Initialize, partition, and format the virtual disk
Get-VirtualDisk -FriendlyName "MirroredDisk" | Get-Disk | Initialize-Disk -PartitionStyle GPT
Get-VirtualDisk -FriendlyName "MirroredDisk" | Get-Disk | New-Partition -UseMaximumSize -DriveLetter F
Format-Volume -DriveLetter F -FileSystem ReFS -NewFileSystemLabel "MirroredData" -Confirm:$false
```

### 6.8 Disk Cleanup and Health Monitoring

```powershell
# Check disk health
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, OperationalStatus, HealthStatus, Size

# Check volume free space
Get-Volume | Where-Object DriveLetter |
    Select-Object DriveLetter, FileSystemLabel,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
        @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,2)}},
        @{N='PercentFree';E={[math]::Round(($_.SizeRemaining/$_.Size)*100,1)}}

# Run disk cleanup utility (interactive)
cleanmgr /d C:
```

---

---

## 7. Event Log Management and Monitoring

> **Objective:** Query and filter event logs, create custom views, export logs, configure retention, set up alerts, and use Performance Monitor.

### 7.1 View Event Logs

```powershell
# Classic cmdlet — System log, newest 20 entries
Get-EventLog -LogName System -Newest 20

# Modern cmdlet — recommended for Windows Server 2022
Get-WinEvent -LogName System -MaxEvents 20

# List available log names
Get-WinEvent -ListLog * | Sort-Object RecordCount -Descending | Select-Object -First 20 LogName, RecordCount
```

### 7.2 Filter Events by Level, Source, and Time

```powershell
# Errors from the last 24 hours
Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    Level     = 2          # 1=Critical, 2=Error, 3=Warning, 4=Information
    StartTime = (Get-Date).AddDays(-1)
}

# Events from a specific source
Get-WinEvent -FilterHashtable @{
    LogName      = 'Application'
    ProviderName = 'MSSQLSERVER'
    StartTime    = (Get-Date).AddHours(-6)
}

# Critical and Error events from multiple logs
Get-WinEvent -FilterHashtable @{
    LogName   = 'System', 'Application'
    Level     = 1, 2
    StartTime = (Get-Date).AddDays(-7)
} | Format-Table TimeCreated, LevelDisplayName, ProviderName, Message -Wrap
```

| Level Value | Level Name |
|-------------|------------|
| 1 | Critical |
| 2 | Error |
| 3 | Warning |
| 4 | Informational |
| 5 | Verbose |

### 7.3 Create Custom Event Views (XML Queries)

```powershell
# Advanced XML filter — failed logon attempts (Event ID 4625)
$XmlQuery = @"
<QueryList>
  <Query Id="0" Path="Security">
    <Select Path="Security">
      *[System[(EventID=4625) and TimeCreated[timediff(@SystemTime) &lt;= 86400000]]]
    </Select>
  </Query>
</QueryList>
"@

Get-WinEvent -FilterXml $XmlQuery
```

### 7.4 Export Event Logs

```powershell
# Export to EVTX (native format)
wevtutil epl System "C:\Logs\System_export.evtx"

# Export to CSV using PowerShell
Get-WinEvent -LogName System -MaxEvents 500 |
    Select-Object TimeCreated, LevelDisplayName, ProviderName, Id, Message |
    Export-Csv -Path "C:\Logs\System_events.csv" -NoTypeInformation

# Export to HTML
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2; StartTime=(Get-Date).AddDays(-7)} |
    Select-Object TimeCreated, LevelDisplayName, ProviderName, Message |
    ConvertTo-Html -Title "Critical & Error Events" |
    Out-File "C:\Logs\CriticalErrors.html"
```

### 7.5 Configure Log Size and Retention

```powershell
# View current log settings
wevtutil gl System

# Set maximum log size to 100 MB and enable auto-overwrite
wevtutil sl System /ms:104857600
wevtutil sl Application /ms:104857600

# Using PowerShell
Limit-EventLog -LogName System -MaximumSize 100MB -OverflowAction OverwriteAsNeeded
Limit-EventLog -LogName Application -MaximumSize 100MB -OverflowAction OverwriteAsNeeded
```

### 7.6 Set Up Email Alerts for Critical Events

Create a monitoring script at `C:\Scripts\EventAlert.ps1`:

```powershell
# EventAlert.ps1 — Send email on critical events
$CriticalEvents = Get-WinEvent -FilterHashtable @{
    LogName   = 'System', 'Application'
    Level     = 1, 2
    StartTime = (Get-Date).AddMinutes(-15)
} -ErrorAction SilentlyContinue

if ($CriticalEvents) {
    $Body = $CriticalEvents |
        Select-Object TimeCreated, LevelDisplayName, ProviderName, Message |
        ConvertTo-Html -Title "Server Alert — $env:COMPUTERNAME" |
        Out-String

    $MailParams = @{
        From       = "alerts@corp.local"
        To         = "admin@corp.local"
        Subject    = "⚠️ Critical Events on $env:COMPUTERNAME — $(Get-Date -Format 'yyyy-MM-dd HH:mm')"
        Body       = $Body
        BodyAsHtml = $true
        SmtpServer = "mail.corp.local"
    }
    Send-MailMessage @MailParams
}
```

Schedule it to run every 15 minutes:

```powershell
$Action  = New-ScheduledTaskAction -Execute "PowerShell.exe" `
    -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\Scripts\EventAlert.ps1"
$Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
    -RepetitionInterval (New-TimeSpan -Minutes 15)

Register-ScheduledTask -TaskName "CriticalEventAlert" `
    -Action $Action -Trigger $Trigger `
    -User "SYSTEM" -RunLevel Highest
```

### 7.7 Performance Monitor Basics

```powershell
# Get CPU usage (% Processor Time)
Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 5

# Get available memory
Get-Counter '\Memory\Available MBytes' -SampleInterval 2 -MaxSamples 5

# Get disk read/write per second
Get-Counter '\PhysicalDisk(_Total)\Disk Reads/sec', '\PhysicalDisk(_Total)\Disk Writes/sec'

# Multiple counters in one call
$Counters = @(
    '\Processor(_Total)\% Processor Time',
    '\Memory\Available MBytes',
    '\PhysicalDisk(_Total)\% Disk Time',
    '\Network Interface(*)\Bytes Total/sec'
)
Get-Counter -Counter $Counters -SampleInterval 5 -MaxSamples 3
```

**Save performance data to CSV:**

```powershell
Get-Counter -Counter '\Processor(_Total)\% Processor Time' -SampleInterval 5 -MaxSamples 60 |
    ForEach-Object {
        [PSCustomObject]@{
            Timestamp = $_.Timestamp
            CPUPercent = [math]::Round($_.CounterSamples[0].CookedValue, 2)
        }
    } | Export-Csv "C:\Logs\cpu_usage.csv" -NoTypeInformation -Append
```

| Counter Path | Description |
|-------------|-------------|
| `\Processor(_Total)\% Processor Time` | Overall CPU usage |
| `\Memory\Available MBytes` | Free RAM in MB |
| `\Memory\% Committed Bytes In Use` | Memory utilization % |
| `\PhysicalDisk(_Total)\% Disk Time` | Disk busy percentage |
| `\PhysicalDisk(_Total)\Avg. Disk Queue Length` | I/O queue depth |
| `\Network Interface(*)\Bytes Total/sec` | Network throughput |
| `\TCPv4\Connections Established` | Active TCP connections |

---

> **✅ Checkpoint:** After completing all 7 medium scenarios you should be comfortable with Active Directory, DNS, DHCP, backups, remote management, storage, and event log monitoring. Proceed to the **Hard** scenarios for advanced topics!
