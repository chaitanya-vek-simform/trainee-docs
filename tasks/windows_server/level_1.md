# Windows Server — Level 1 (🟢 Easy)

## 1. Windows Server Initial Setup

> **Objective:** Perform post-installation configuration — verify the OS version, set the hostname, configure a static IP, apply Windows Updates, and enable Remote Desktop.

### 1.1 Check Windows Version

```powershell
# Quick graphical version dialog
winver

# Detailed OS information via PowerShell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsBuildNumber, OsArchitecture
```

You can also use the classic command:

```powershell
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

### 1.2 Set Hostname

```powershell
# View current hostname
hostname

# Rename the computer (requires restart)
Rename-Computer -NewName "WEB-SVR-01" -Force -Restart
```

> ⚠️ The server will reboot automatically after renaming. Plan accordingly.

### 1.3 Configure a Static IP Address

```powershell
# List network adapters
Get-NetAdapter

# Assign a static IP address
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress 192.168.1.10 `
    -PrefixLength 24 `
    -DefaultGateway 192.168.1.1

# Set DNS servers
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses 8.8.8.8, 8.8.4.4
```

**Verify the configuration:**

```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet0"
Get-DnsClientServerAddress -InterfaceAlias "Ethernet0"
```

### 1.4 Windows Update

```powershell
# Install the PSWindowsUpdate module (first time only)
Install-Module PSWindowsUpdate -Force -Confirm:$false

# Import the module
Import-Module PSWindowsUpdate

# Check for available updates
Get-WindowsUpdate

# Install all available updates (auto-reboot if needed)
Install-WindowsUpdate -AcceptAll -AutoReboot
```

### 1.5 Enable Remote Desktop

```powershell
# Enable RDP via registry
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
    -Name "fDenyTSConnections" -Value 0

# Allow RDP through the firewall
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

### 1.6 Verify Everything with `systeminfo`

```powershell
systeminfo
```

| Item to Verify | Where to Look |
|----------------|---------------|
| OS Name & Version | Top of `systeminfo` output |
| Hostname | `Host Name` field |
| Network Config | `Network Card(s)` section |
| Hotfixes Installed | `Hotfix(s)` section |
| Domain / Workgroup | `Domain` field |

---

---

## 2. User and Group Management

> **Objective:** Create and manage local users and groups, set password policies, and control account states.

### 2.1 Create a Local User

```powershell
# Create a user with a secure password
$Password = ConvertTo-SecureString "P@ssw0rd2026!" -AsPlainText -Force
New-LocalUser -Name "devtrainee" -Password $Password -FullName "Dev Trainee" `
    -Description "DevOps trainee account"
```

### 2.2 Change a User's Password

```powershell
$NewPass = ConvertTo-SecureString "NewP@ss2026!" -AsPlainText -Force
Set-LocalUser -Name "devtrainee" -Password $NewPass
```

### 2.3 Create a Local Group

```powershell
New-LocalGroup -Name "DevOpsTeam" -Description "DevOps team members"
```

### 2.4 Add Users to a Group

```powershell
Add-LocalGroupMember -Group "DevOpsTeam" -Member "devtrainee"

# Add a user to the built-in Administrators group
Add-LocalGroupMember -Group "Administrators" -Member "devtrainee"
```

### 2.5 View Users and Groups

```powershell
# List all local users
Get-LocalUser

# List all local groups
Get-LocalGroup

# List members of a specific group
Get-LocalGroupMember -Group "DevOpsTeam"
```

### 2.6 Disable / Enable / Delete Accounts

```powershell
# Disable a user account
Disable-LocalUser -Name "devtrainee"

# Enable a user account
Enable-LocalUser -Name "devtrainee"

# Delete a user
Remove-LocalUser -Name "devtrainee"

# Delete a group
Remove-LocalGroup -Name "DevOpsTeam"
```

### 2.7 Configure Password Policies

```powershell
# View current account/password policies
net accounts

# Set minimum password length and lockout threshold
net accounts /minpwlen:12 /lockoutthreshold:5
```

| Policy | Command |
|--------|---------|
| Min password length | `net accounts /minpwlen:12` |
| Max password age (days) | `net accounts /maxpwage:90` |
| Min password age (days) | `net accounts /minpwage:1` |
| Lockout threshold | `net accounts /lockoutthreshold:5` |
| Lockout duration (min) | `net accounts /lockoutduration:30` |

---

---

## 3. Windows Firewall Configuration

> **Objective:** Manage Windows Defender Firewall profiles and rules using PowerShell.

### 3.1 View Firewall Status

```powershell
# View status of all profiles (Domain, Private, Public)
Get-NetFirewallProfile | Format-Table Name, Enabled
```

### 3.2 Enable / Disable Firewall Profiles

```powershell
# Disable the Public profile (use with caution)
Set-NetFirewallProfile -Profile Public -Enabled False

# Re-enable it
Set-NetFirewallProfile -Profile Public -Enabled True

# Enable all profiles at once
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True
```

### 3.3 Create Inbound Rules

```powershell
# Allow HTTP (port 80)
New-NetFirewallRule -DisplayName "Allow HTTP" `
    -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

# Allow HTTPS (port 443)
New-NetFirewallRule -DisplayName "Allow HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# Allow RDP (port 3389)
New-NetFirewallRule -DisplayName "Allow RDP" `
    -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow

# Allow a custom port (e.g., 8080)
New-NetFirewallRule -DisplayName "Allow Custom 8080" `
    -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
```

### 3.4 Create Outbound Rules

```powershell
# Block outbound traffic on port 25 (SMTP)
New-NetFirewallRule -DisplayName "Block SMTP Outbound" `
    -Direction Outbound -Protocol TCP -LocalPort 25 -Action Block
```

### 3.5 View Existing Rules

```powershell
# List all firewall rules (can be very long)
Get-NetFirewallRule | Format-Table DisplayName, Direction, Action, Enabled

# Filter for enabled inbound allow rules
Get-NetFirewallRule -Direction Inbound -Action Allow -Enabled True |
    Format-Table DisplayName, Profile
```

### 3.6 Remove a Rule

```powershell
Remove-NetFirewallRule -DisplayName "Allow Custom 8080"
```

### 3.7 Enable Firewall Logging

```powershell
# Enable logging for the Domain profile
Set-NetFirewallProfile -Profile Domain `
    -LogAllowed True -LogBlocked True `
    -LogFileName "%SystemRoot%\System32\LogFiles\Firewall\pfirewall.log" `
    -LogMaxSizeKilobytes 16384
```

### Common Firewall Rules Reference

| Rule Name | Port | Protocol | Direction |
|-----------|------|----------|-----------|
| HTTP | 80 | TCP | Inbound |
| HTTPS | 443 | TCP | Inbound |
| RDP | 3389 | TCP | Inbound |
| DNS | 53 | TCP/UDP | Outbound |
| WinRM (HTTP) | 5985 | TCP | Inbound |
| WinRM (HTTPS) | 5986 | TCP | Inbound |
| SMB | 445 | TCP | Inbound |
| ICMP (Ping) | — | ICMPv4 | Inbound |

---

---

## 4. Installing and Managing Roles & Features

> **Objective:** Install, verify, and remove Windows Server roles and features using PowerShell and Server Manager.

### 4.1 List Available Roles and Features

```powershell
# List all available roles and features
Get-WindowsFeature

# Filter to only installed features
Get-WindowsFeature | Where-Object Installed -eq $true

# Search for a specific feature
Get-WindowsFeature -Name *Web*
```

### 4.2 Install IIS (Web Server)

```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools -IncludeAllSubFeature
```

### 4.3 Install Other Common Roles

```powershell
# Install DNS Server
Install-WindowsFeature DNS -IncludeManagementTools

# Install DHCP Server
Install-WindowsFeature DHCP -IncludeManagementTools

# Install Active Directory Domain Services
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Install Windows Server Backup
Install-WindowsFeature Windows-Server-Backup
```

### 4.4 Verify Installation

```powershell
# Check if IIS is installed
Get-WindowsFeature Web-Server

# Verify multiple features at once
Get-WindowsFeature Web-Server, DNS, DHCP | Format-Table Name, InstallState
```

### 4.5 Remove a Role or Feature

```powershell
# Remove the DNS role
Uninstall-WindowsFeature DNS -IncludeManagementTools
```

> 💡 **Server Manager (GUI):** You can also install/remove roles through **Server Manager → Manage → Add Roles and Features**. The wizard walks you through the same steps graphically.

### Common Roles & Features Reference

| Role / Feature | Feature Name | Purpose |
|----------------|-------------|---------|
| IIS Web Server | `Web-Server` | Host websites and web applications |
| DNS Server | `DNS` | Name resolution |
| DHCP Server | `DHCP` | Automatic IP address assignment |
| AD Domain Services | `AD-Domain-Services` | Directory services & authentication |
| File Server | `FS-FileServer` | File sharing |
| Hyper-V | `Hyper-V` | Virtualization |
| Windows Server Backup | `Windows-Server-Backup` | Backup & recovery |
| .NET Framework 4.8 | `NET-Framework-45-Core` | Application support |

---

---

## 5. IIS Web Server Basics

> **Objective:** Install IIS, create and manage websites, deploy static content, and configure bindings.

### 5.1 Install IIS

```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
```

### 5.2 Verify the Default Website

```powershell
# Test from the server itself
curl http://localhost

# Or use PowerShell
Invoke-WebRequest -Uri http://localhost -UseBasicParsing | Select-Object StatusCode
```

Open a browser and navigate to `http://localhost` — you should see the default IIS welcome page.

### 5.3 IIS Directory Structure

| Path | Description |
|------|-------------|
| `%SystemRoot%\inetpub\wwwroot` | Default website root |
| `%SystemRoot%\inetpub\logs` | IIS log files |
| `%SystemRoot%\System32\inetsrv` | IIS binaries and `appcmd.exe` |
| `%SystemRoot%\System32\inetsrv\config` | `applicationHost.config` (central config) |

### 5.4 Create a New Website

```powershell
# Import the IIS module
Import-Module WebAdministration

# Create the content directory
New-Item -Path "C:\inetpub\mysite" -ItemType Directory -Force

# Create a simple index page
Set-Content -Path "C:\inetpub\mysite\index.html" -Value @"
<!DOCTYPE html>
<html>
<head><title>My Site</title></head>
<body><h1>Hello from My Site!</h1></body>
</html>
"@

# Create the new website
New-Website -Name "MySite" -Port 8080 -PhysicalPath "C:\inetpub\mysite" -Force
```

### 5.5 Configure Bindings

```powershell
# Add an HTTPS binding (requires a certificate)
New-WebBinding -Name "MySite" -Protocol "https" -Port 443 -HostHeader "mysite.local"

# Add a hostname binding on port 80
New-WebBinding -Name "MySite" -Protocol "http" -Port 80 -HostHeader "mysite.local"

# View all bindings for a site
Get-WebBinding -Name "MySite"
```

### 5.6 Start / Stop Websites

```powershell
# Stop a website
Stop-Website -Name "MySite"

# Start a website
Start-Website -Name "MySite"

# Restart the entire IIS service
iisreset
```

### 5.7 Application Pools

```powershell
# List all application pools
Get-IISAppPool

# Create a new application pool
New-WebAppPool -Name "MySitePool"

# Assign a site to an application pool
Set-ItemProperty "IIS:\Sites\MySite" -Name applicationPool -Value "MySitePool"

# Recycle an application pool
Restart-WebAppPool -Name "MySitePool"
```

### Useful IIS PowerShell Commands

| Command | Description |
|---------|-------------|
| `Get-Website` | List all websites |
| `Get-WebBinding` | List bindings for a site |
| `Get-IISAppPool` | List application pools |
| `New-Website` | Create a new website |
| `New-WebBinding` | Add a binding |
| `New-WebAppPool` | Create an application pool |
| `Start-Website` | Start a website |
| `Stop-Website` | Stop a website |
| `Remove-Website` | Delete a website |
| `iisreset` | Restart IIS service |

---

---

## 6. Task Scheduler

> **Objective:** Create, manage, and automate scheduled tasks using PowerShell.

### 6.1 Create a Basic Scheduled Task

```powershell
# Define the action (what to run)
$Action = New-ScheduledTaskAction -Execute "PowerShell.exe" `
    -Argument "-NoProfile -WindowStyle Hidden -File C:\Scripts\cleanup.ps1"

# Define the trigger (when to run)
$Trigger = New-ScheduledTaskTrigger -Daily -At "02:00AM"

# Register the task
Register-ScheduledTask -TaskName "DailyCleanup" `
    -Action $Action -Trigger $Trigger `
    -Description "Runs daily disk cleanup script" `
    -User "SYSTEM" -RunLevel Highest
```

### 6.2 Common Trigger Types

| Trigger | PowerShell Command |
|---------|--------------------|
| Daily at 2 AM | `New-ScheduledTaskTrigger -Daily -At "02:00AM"` |
| Weekly on Monday | `New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday -At "06:00AM"` |
| At system startup | `New-ScheduledTaskTrigger -AtStartup` |
| At user logon | `New-ScheduledTaskTrigger -AtLogOn` |
| Once at a specific time | `New-ScheduledTaskTrigger -Once -At "2026-03-15 10:00AM"` |
| Every 5 minutes | `New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 5)` |

### 6.3 View Scheduled Tasks

```powershell
# List all scheduled tasks
Get-ScheduledTask

# Filter by task name
Get-ScheduledTask -TaskName "DailyCleanup"

# View detailed task info
Get-ScheduledTaskInfo -TaskName "DailyCleanup"
```

### 6.4 Enable / Disable Tasks

```powershell
Disable-ScheduledTask -TaskName "DailyCleanup"
Enable-ScheduledTask -TaskName "DailyCleanup"
```

### 6.5 Run a Task Manually

```powershell
Start-ScheduledTask -TaskName "DailyCleanup"
```

### 6.6 Delete a Task

```powershell
Unregister-ScheduledTask -TaskName "DailyCleanup" -Confirm:$false
```

### 6.7 Example: Automated Disk Cleanup Script

Create the cleanup script at `C:\Scripts\cleanup.ps1`:

```powershell
# cleanup.ps1 — Automated disk cleanup script
$LogFile = "C:\Scripts\Logs\cleanup_$(Get-Date -Format 'yyyyMMdd').log"
New-Item -Path (Split-Path $LogFile) -ItemType Directory -Force | Out-Null

"=== Disk Cleanup Started: $(Get-Date) ===" | Out-File $LogFile -Append

# Clear Temp folders
$TempPaths = @(
    "$env:TEMP",
    "C:\Windows\Temp",
    "C:\Windows\Prefetch"
)

foreach ($Path in $TempPaths) {
    if (Test-Path $Path) {
        $FileCount = (Get-ChildItem $Path -Recurse -ErrorAction SilentlyContinue).Count
        Remove-Item "$Path\*" -Recurse -Force -ErrorAction SilentlyContinue
        "Cleared $FileCount items from $Path" | Out-File $LogFile -Append
    }
}

# Clear IIS logs older than 30 days
$IISLogs = "C:\inetpub\logs\LogFiles"
if (Test-Path $IISLogs) {
    Get-ChildItem $IISLogs -Recurse -File |
        Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-30) } |
        Remove-Item -Force
    "Cleared IIS logs older than 30 days" | Out-File $LogFile -Append
}

"=== Disk Cleanup Completed: $(Get-Date) ===" | Out-File $LogFile -Append
```

Then register the scheduled task:

```powershell
$Action  = New-ScheduledTaskAction -Execute "PowerShell.exe" `
    -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\Scripts\cleanup.ps1"
$Trigger = New-ScheduledTaskTrigger -Daily -At "03:00AM"

Register-ScheduledTask -TaskName "AutoDiskCleanup" `
    -Action $Action -Trigger $Trigger `
    -Description "Automated daily disk cleanup" `
    -User "SYSTEM" -RunLevel Highest
```

---

---

## 7. Basic PowerShell for Server Administration

> **Objective:** Learn essential PowerShell cmdlets, pipeline usage, formatting, scripting basics, and the help system for day-to-day server administration.

### 7.1 Essential Cmdlets

| Cmdlet | Description | Example |
|--------|-------------|---------|
| `Get-Process` | List running processes | `Get-Process \| Sort-Object CPU -Descending \| Select -First 10` |
| `Get-Service` | List Windows services | `Get-Service \| Where-Object Status -eq "Running"` |
| `Get-EventLog` | Read classic event logs | `Get-EventLog -LogName System -Newest 20` |
| `Get-WinEvent` | Read modern event logs | `Get-WinEvent -LogName System -MaxEvents 20` |
| `Get-Disk` | List physical disks | `Get-Disk` |
| `Get-Volume` | List volumes | `Get-Volume` |
| `Get-NetAdapter` | List network adapters | `Get-NetAdapter` |
| `Test-Connection` | Ping a host | `Test-Connection google.com -Count 4` |
| `Get-ComputerInfo` | System information | `Get-ComputerInfo` |
| `Restart-Service` | Restart a service | `Restart-Service -Name "W3SVC"` |

### 7.2 Pipeline Basics

The pipeline (`|`) passes objects from one cmdlet to the next:

```powershell
# Get the top 5 processes by memory usage
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 5 Name, @{N='MemoryMB';E={[math]::Round($_.WorkingSet64/1MB,2)}}

# Find all stopped services
Get-Service | Where-Object { $_.Status -eq "Stopped" }

# Count running services
(Get-Service | Where-Object Status -eq "Running").Count
```

### 7.3 Formatting Output

```powershell
# Format as a table
Get-Service | Format-Table Name, Status, StartType -AutoSize

# Format as a list (detailed)
Get-Service W3SVC | Format-List *

# Export to CSV
Get-Process | Select-Object Name, CPU, WorkingSet64 |
    Export-Csv -Path "C:\Reports\processes.csv" -NoTypeInformation

# Export to HTML
Get-Service | ConvertTo-Html -Property Name, Status |
    Out-File "C:\Reports\services.html"
```

### 7.4 Script Execution Policy

```powershell
# Check current policy
Get-ExecutionPolicy

# Allow running local scripts
Set-ExecutionPolicy RemoteSigned -Force

# Allow all scripts (use only in dev/test)
Set-ExecutionPolicy Unrestricted -Force
```

| Policy | Description |
|--------|-------------|
| `Restricted` | No scripts can run (default on client OS) |
| `AllSigned` | Only scripts signed by a trusted publisher |
| `RemoteSigned` | Local scripts run freely; downloaded scripts need signing |
| `Unrestricted` | All scripts run (warning on downloaded scripts) |
| `Bypass` | Nothing is blocked, no warnings |

### 7.5 Basic Scripting

**Variables:**

```powershell
$ServerName = "WEB-SVR-01"
$MaxRetries = 3
$Services = @("W3SVC", "WAS", "MSSQLSERVER")
```

**If / Else:**

```powershell
$DiskFree = (Get-Volume -DriveLetter C).SizeRemaining / 1GB
if ($DiskFree -lt 10) {
    Write-Warning "Low disk space: $([math]::Round($DiskFree,2)) GB remaining"
} elseif ($DiskFree -lt 20) {
    Write-Host "Disk space is moderate: $([math]::Round($DiskFree,2)) GB remaining" -ForegroundColor Yellow
} else {
    Write-Host "Disk space is healthy: $([math]::Round($DiskFree,2)) GB remaining" -ForegroundColor Green
}
```

**ForEach Loop:**

```powershell
$Services = @("W3SVC", "WAS", "Spooler")
foreach ($Svc in $Services) {
    $Status = (Get-Service -Name $Svc -ErrorAction SilentlyContinue).Status
    Write-Host "$Svc : $Status"
}
```

**Functions:**

```powershell
function Get-ServerHealth {
    param (
        [string]$ComputerName = $env:COMPUTERNAME
    )

    $OS   = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName
    $CPU  = Get-CimInstance Win32_Processor -ComputerName $ComputerName
    $Disk = Get-Volume -DriveLetter C

    [PSCustomObject]@{
        Server       = $ComputerName
        Uptime       = (Get-Date) - $OS.LastBootUpTime
        CPUName      = $CPU.Name
        FreeMemoryGB = [math]::Round($OS.FreePhysicalMemory / 1MB, 2)
        FreeDiskGB   = [math]::Round($Disk.SizeRemaining / 1GB, 2)
    }
}

# Call the function
Get-ServerHealth
```

### 7.6 Help System

```powershell
# Get help for a cmdlet
Get-Help Get-Service

# Get detailed help with examples
Get-Help Get-Service -Examples

# Update the help files (run as Administrator)
Update-Help -Force -ErrorAction SilentlyContinue

# Find commands by keyword
Get-Command *firewall*

# Find commands in a module
Get-Command -Module NetSecurity
```

| Help Command | Description |
|-------------|-------------|
| `Get-Help <cmdlet>` | Basic help |
| `Get-Help <cmdlet> -Detailed` | Detailed help |
| `Get-Help <cmdlet> -Examples` | Usage examples |
| `Get-Help <cmdlet> -Full` | Full documentation |
| `Get-Help <cmdlet> -Online` | Open docs in browser |
| `Get-Command -Verb Get` | Find all "Get-*" commands |
| `Get-Command -Module <module>` | List commands in a module |

---

> **✅ Checkpoint:** After completing all 7 scenarios you should be comfortable with basic Windows Server configuration, user management, firewall rules, IIS hosting, scheduled tasks, and PowerShell fundamentals. Move on to the **Medium** scenarios to level up!
