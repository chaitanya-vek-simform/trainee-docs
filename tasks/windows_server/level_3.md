# Windows Server — Level 3 (🔴 Hard)

## 1. IIS as a Reverse Proxy with URL Rewrite & ARR

> **Objective:** Configure IIS as a reverse proxy using Application Request Routing (ARR) and URL Rewrite to route traffic to backend application servers, set up load balancing, and enable SSL offloading.

### 1.1 Prerequisites

```powershell
# Ensure IIS is installed with management tools
Install-WindowsFeature Web-Server -IncludeManagementTools -IncludeAllSubFeature
```

### 1.2 Install Application Request Routing (ARR)

ARR and URL Rewrite are not included with IIS by default. Install them using the Web Platform Installer or direct download:

```powershell
# Download and install ARR 3.0
# Option 1 — Web Platform Installer (if available)
# Option 2 — Direct MSI download
Invoke-WebRequest -Uri "https://download.microsoft.com/download/E/9/8/E9849D6A-020E-47E4-9FD0-A023E99B54EB/requestRouter_amd64.msi" `
    -OutFile "C:\Temp\ARR3.msi"
Start-Process msiexec.exe -ArgumentList "/i C:\Temp\ARR3.msi /quiet /norestart" -Wait

# Download and install URL Rewrite 2.1
Invoke-WebRequest -Uri "https://download.microsoft.com/download/1/2/8/128E2E22-C1B9-44A4-BE2A-5859ED1D4592/rewrite_amd64_en-US.msi" `
    -OutFile "C:\Temp\URLRewrite.msi"
Start-Process msiexec.exe -ArgumentList "/i C:\Temp\URLRewrite.msi /quiet /norestart" -Wait
```

> 💡 After installation, restart IIS: `iisreset`

### 1.3 Enable Proxy Functionality

```powershell
# Enable ARR proxy via appcmd
C:\Windows\System32\inetsrv\appcmd.exe set config -section:system.webServer/proxy /enabled:"True" /commit:apphost

# Or via IIS Manager:
# Server level → Application Request Routing Cache → Server Proxy Settings → Enable proxy ✅
```

### 1.4 Configure Reverse Proxy Rules (Single Backend)

Create a `web.config` in the IIS site root to route all traffic to a backend server:

```powershell
# Create the reverse proxy site directory
New-Item -Path "C:\inetpub\reverseproxy" -ItemType Directory -Force

# Write the URL Rewrite rule
Set-Content -Path "C:\inetpub\reverseproxy\web.config" -Value @"
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="ReverseProxy-Backend" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="http://192.168.1.50:3000/{R:1}" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
"@

# Create the IIS site
New-Website -Name "ReverseProxy" -Port 80 -PhysicalPath "C:\inetpub\reverseproxy" -Force
```

### 1.5 Path-Based Routing (Multiple Backends)

Route different URL paths to different backend servers:

```powershell
Set-Content -Path "C:\inetpub\reverseproxy\web.config" -Value @"
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <!-- Route /api/* to backend API server -->
        <rule name="API-Backend" stopProcessing="true">
          <match url="^api/(.*)" />
          <action type="Rewrite" url="http://192.168.1.51:5000/api/{R:1}" />
        </rule>
        <!-- Route /app/* to backend app server -->
        <rule name="App-Backend" stopProcessing="true">
          <match url="^app/(.*)" />
          <action type="Rewrite" url="http://192.168.1.52:8080/{R:1}" />
        </rule>
        <!-- Default: route to frontend -->
        <rule name="Frontend-Default" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="http://192.168.1.50:3000/{R:1}" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
"@
```

### 1.6 Load Balancing with Server Farms

```powershell
# Create a Server Farm via appcmd
C:\Windows\System32\inetsrv\appcmd.exe set config -section:webFarms /+"[name='WebFarm']" /commit:apphost

# Add backend servers
C:\Windows\System32\inetsrv\appcmd.exe set config -section:webFarms `
    /+"[name='WebFarm'].[address='192.168.1.50']" /commit:apphost
C:\Windows\System32\inetsrv\appcmd.exe set config -section:webFarms `
    /+"[name='WebFarm'].[address='192.168.1.51']" /commit:apphost
C:\Windows\System32\inetsrv\appcmd.exe set config -section:webFarms `
    /+"[name='WebFarm'].[address='192.168.1.52']" /commit:apphost
```

### 1.7 Health Monitoring

Configure health checks in IIS Manager:

- **Server Farms → WebFarm → Health Test**
- Set a URL to monitor: `http://backend:3000/health`
- Interval: 30 seconds
- Timeout: 10 seconds
- Response match: `"status":"ok"`

```powershell
# Configure via appcmd
C:\Windows\System32\inetsrv\appcmd.exe set config -section:webFarms `
    /[name='WebFarm'].applicationRequestRouting.healthCheck.url:"http://localhost:3000/health" `
    /[name='WebFarm'].applicationRequestRouting.healthCheck.interval:"00:00:30" `
    /[name='WebFarm'].applicationRequestRouting.healthCheck.responseMatch:"ok" `
    /commit:apphost
```

### 1.8 SSL Offloading

Terminate SSL at the IIS reverse proxy and forward HTTP to backends:

```powershell
# Import an SSL certificate (PFX)
$CertPassword = ConvertTo-SecureString "CertP@ss!" -AsPlainText -Force
Import-PfxCertificate -FilePath "C:\Certs\proxy.pfx" -CertStoreLocation Cert:\LocalMachine\My -Password $CertPassword

# Bind the certificate to the HTTPS site
$Cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object Subject -like "*proxy.corp.local*"
New-WebBinding -Name "ReverseProxy" -Protocol "https" -Port 443 -HostHeader "proxy.corp.local"
$Binding = Get-WebBinding -Name "ReverseProxy" -Protocol "https"
$Binding.AddSslCertificate($Cert.Thumbprint, "My")
```

Add an HTTPS → HTTP redirect and preserve host headers:

```powershell
# Set to preserve the original Host header
C:\Windows\System32\inetsrv\appcmd.exe set config -section:system.webServer/proxy `
    /preserveHostHeader:"True" /commit:apphost
```

### 1.9 WebSocket Proxying

```powershell
# Enable WebSocket protocol in IIS
Install-WindowsFeature Web-WebSockets

# WebSocket connections are automatically proxied if ARR and URL Rewrite are configured.
# Ensure the backend supports WebSocket and the rewrite rule targets the correct backend.
```

### 1.10 Verify and Troubleshoot

```powershell
# Test the reverse proxy
Invoke-WebRequest -Uri "http://localhost" -UseBasicParsing | Select-Object StatusCode
Invoke-WebRequest -Uri "http://localhost/api/health" -UseBasicParsing

# Check ARR error logs
Get-Content "C:\inetpub\logs\FailedReqLogFiles\*.xml" -ErrorAction SilentlyContinue

# Enable Failed Request Tracing for debugging
C:\Windows\System32\inetsrv\appcmd.exe configure trace "ReverseProxy" /enable

# View IIS logs
Get-Content "C:\inetpub\logs\LogFiles\W3SVC*\*.log" -Tail 50
```

---

---

## 2. Windows Server Hardening — Security Checklist

> **Objective:** Implement a comprehensive security hardening process for Windows Server 2022 covering account policies, services, firewall, encryption, and auditing.

### 2.1 Rename the Administrator Account

```powershell
# Rename the built-in Administrator account
Rename-LocalUser -Name "Administrator" -NewName "SrvAdmin2026"

# In Active Directory environment
Rename-ADObject -Identity (Get-ADUser -Identity "Administrator").DistinguishedName -NewName "SrvAdmin2026"
```

### 2.2 Disable the Guest Account

```powershell
Disable-LocalUser -Name "Guest"

# Verify
Get-LocalUser -Name "Guest" | Select-Object Name, Enabled
```

### 2.3 Configure Account Lockout Policies

```powershell
# Set account lockout policy
net accounts /lockoutthreshold:5 /lockoutduration:30 /lockoutwindow:30

# For domain environments, use Group Policy:
# Computer Configuration → Policies → Windows Settings → Security Settings →
# Account Policies → Account Lockout Policy
```

| Policy | Recommended Value |
|--------|-------------------|
| Account lockout threshold | 5 invalid attempts |
| Account lockout duration | 30 minutes |
| Reset lockout counter after | 30 minutes |
| Minimum password length | 14 characters |
| Password complexity | Enabled |
| Maximum password age | 90 days |
| Enforce password history | 24 passwords |

### 2.4 Enable Audit Policies

```powershell
# Enable advanced audit policies
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Logoff" /success:enable
auditpol /set /subcategory:"Account Lockout" /success:enable /failure:enable
auditpol /set /subcategory:"Other Logon/Logoff Events" /success:enable /failure:enable
auditpol /set /subcategory:"Special Logon" /success:enable
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Security Group Management" /success:enable /failure:enable
auditpol /set /subcategory:"Computer Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable

# View current audit policies
auditpol /get /category:*
```

### 2.5 Disable Unnecessary Services

```powershell
# Common services to disable on a hardened server
$ServicesToDisable = @(
    "XblAuthManager",      # Xbox Live Auth Manager
    "XblGameSave",         # Xbox Live Game Save
    "XboxNetApiSvc",       # Xbox Live Networking Service
    "DiagTrack",           # Connected User Experiences and Telemetry
    "dmwappushservice",    # WAP Push Message Routing Service
    "MapsBroker",          # Downloaded Maps Manager
    "RemoteRegistry",      # Remote Registry
    "lfsvc",               # Geolocation Service
    "Fax"                  # Fax
)

foreach ($Service in $ServicesToDisable) {
    $Svc = Get-Service -Name $Service -ErrorAction SilentlyContinue
    if ($Svc) {
        Stop-Service -Name $Service -Force -ErrorAction SilentlyContinue
        Set-Service -Name $Service -StartupType Disabled
        Write-Host "Disabled: $Service" -ForegroundColor Yellow
    }
}
```

### 2.6 Configure Windows Firewall Advanced Rules

```powershell
# Block all inbound by default, allow outbound
Set-NetFirewallProfile -Profile Domain,Private,Public `
    -DefaultInboundAction Block `
    -DefaultOutboundAction Allow `
    -Enabled True

# Allow only essential inbound traffic
New-NetFirewallRule -DisplayName "Allow RDP from Admin Subnet" `
    -Direction Inbound -Protocol TCP -LocalPort 3389 `
    -RemoteAddress 10.0.0.0/24 -Action Allow

New-NetFirewallRule -DisplayName "Allow WinRM HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 5986 `
    -Action Allow

# Block SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force

# Enable firewall logging for all profiles
Set-NetFirewallProfile -Profile Domain,Private,Public `
    -LogAllowed True -LogBlocked True `
    -LogMaxSizeKilobytes 32768
```

### 2.7 Enable BitLocker

```powershell
# Install BitLocker feature
Install-WindowsFeature BitLocker -IncludeManagementTools -Restart

# Enable BitLocker on the OS drive (requires TPM or USB key)
Enable-BitLocker -MountPoint "C:" `
    -EncryptionMethod XtsAes256 `
    -RecoveryPasswordProtector

# View BitLocker status
Get-BitLockerVolume

# Backup recovery key to Active Directory (domain-joined)
Backup-BitLockerKeyProtector -MountPoint "C:" `
    -KeyProtectorId (Get-BitLockerVolume -MountPoint "C:").KeyProtector[1].KeyProtectorId
```

### 2.8 Configure Windows Defender

```powershell
# Ensure Windows Defender is enabled
Set-MpPreference -DisableRealtimeMonitoring $false

# Update definitions
Update-MpSignature

# Configure scan settings
Set-MpPreference -ScanScheduleDay 0 `
    -ScanScheduleTime "02:00:00" `
    -ScanType 2

# Enable cloud-delivered protection
Set-MpPreference -MAPSReporting Advanced
Set-MpPreference -SubmitSamplesConsent SendAllSamples

# Add exclusion paths (for known application directories)
Add-MpPreference -ExclusionPath "C:\AppData\Database"

# Run a quick scan
Start-MpScan -ScanType QuickScan

# View threat history
Get-MpThreatDetection
```

### 2.9 Apply Security Baselines (Security Compliance Toolkit)

Microsoft provides Security Compliance Toolkit (SCT) baselines:

1. Download from the [Microsoft Security Compliance Toolkit](https://www.microsoft.com/en-us/download/details.aspx?id=55319).
2. Extract the baseline ZIP.
3. Apply using the provided PowerShell scripts:

```powershell
# Navigate to the extracted baseline folder
cd "C:\SecurityBaselines\Windows Server 2022"

# Apply the baseline (example)
.\Baseline-LocalInstall.ps1

# Or apply individual GPO settings using LGPO.exe
.\LGPO.exe /g ..\GPOs\{GUID}
```

### 2.10 Regular Security Audits

```powershell
# Check for accounts with no password expiry
Get-LocalUser | Where-Object { $_.PasswordExpires -eq $null -and $_.Enabled } |
    Select-Object Name, PasswordExpires, Enabled

# Find users who haven't logged in for 90+ days (AD)
Search-ADAccount -AccountInactive -TimeSpan 90 -UsersOnly |
    Select-Object Name, LastLogonDate, Enabled

# Check for open ports
Get-NetTCPConnection -State Listen | Sort-Object LocalPort |
    Select-Object LocalAddress, LocalPort, OwningProcess,
        @{N='Process';E={(Get-Process -Id $_.OwningProcess).ProcessName}}

# Review installed software
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName, DisplayVersion, Publisher, InstallDate |
    Sort-Object DisplayName
```

### Security Checklist Summary

| # | Item | Status |
|---|------|--------|
| 1 | Rename Administrator account | ☐ |
| 2 | Disable Guest account | ☐ |
| 3 | Configure account lockout policies | ☐ |
| 4 | Set password complexity & history | ☐ |
| 5 | Enable audit policies | ☐ |
| 6 | Disable unnecessary services | ☐ |
| 7 | Configure firewall (deny inbound by default) | ☐ |
| 8 | Disable SMBv1 | ☐ |
| 9 | Enable BitLocker | ☐ |
| 10 | Configure Windows Defender | ☐ |
| 11 | Apply security baselines | ☐ |
| 12 | Enable NTP time sync | ☐ |
| 13 | Restrict RDP to specific subnets | ☐ |
| 14 | Enable HTTPS-only for management | ☐ |
| 15 | Regular patching schedule | ☐ |
| 16 | Remove unused roles & features | ☐ |
| 17 | Conduct periodic security audits | ☐ |

---

---

## 3. Hyper-V Virtualization

> **Objective:** Install Hyper-V, create and manage virtual machines, configure virtual switches, and work with snapshots and live migration.

### 3.1 Install the Hyper-V Role

```powershell
Install-WindowsFeature Hyper-V -IncludeManagementTools -Restart
```

> ⚠️ Hyper-V requires a 64-bit processor with virtualization extensions (Intel VT-x / AMD-V) enabled in BIOS/UEFI.

### 3.2 Create a Virtual Switch

```powershell
# External switch (connects VMs to the physical network)
New-VMSwitch -Name "ExternalSwitch" `
    -NetAdapterName "Ethernet0" `
    -AllowManagementOS $true

# Internal switch (host ↔ VM communication only)
New-VMSwitch -Name "InternalSwitch" -SwitchType Internal

# Private switch (VM ↔ VM only, no host)
New-VMSwitch -Name "PrivateSwitch" -SwitchType Private

# List all virtual switches
Get-VMSwitch
```

| Switch Type | Host Access | Physical Network | VM ↔ VM |
|-------------|-------------|------------------|---------|
| **External** | ✅ | ✅ | ✅ |
| **Internal** | ✅ | ❌ | ✅ |
| **Private** | ❌ | ❌ | ✅ |

### 3.3 Create a Virtual Machine

```powershell
# Create a Generation 2 VM
New-VM -Name "WEB-VM-01" `
    -MemoryStartupBytes 4GB `
    -NewVHDPath "C:\Hyper-V\VHDs\WEB-VM-01.vhdx" `
    -NewVHDSizeBytes 60GB `
    -SwitchName "ExternalSwitch" `
    -Generation 2 `
    -Path "C:\Hyper-V\VMs"
```

### 3.4 Configure VM Settings

```powershell
# Set CPU count
Set-VMProcessor -VMName "WEB-VM-01" -Count 4

# Configure dynamic memory
Set-VMMemory -VMName "WEB-VM-01" `
    -DynamicMemoryEnabled $true `
    -MinimumBytes 2GB `
    -StartupBytes 4GB `
    -MaximumBytes 8GB

# Attach an ISO for OS installation
Add-VMDvdDrive -VMName "WEB-VM-01" -Path "C:\ISOs\WindowsServer2022.iso"

# Set boot order (DVD first for installation)
$DVDDrive = Get-VMDvdDrive -VMName "WEB-VM-01"
Set-VMFirmware -VMName "WEB-VM-01" -FirstBootDevice $DVDDrive

# Add a second network adapter
Add-VMNetworkAdapter -VMName "WEB-VM-01" -SwitchName "InternalSwitch" -Name "Management"

# Add a second virtual hard disk
New-VHD -Path "C:\Hyper-V\VHDs\WEB-VM-01-Data.vhdx" -SizeBytes 100GB -Dynamic
Add-VMHardDiskDrive -VMName "WEB-VM-01" -Path "C:\Hyper-V\VHDs\WEB-VM-01-Data.vhdx"
```

### 3.5 Manage VM Snapshots (Checkpoints)

```powershell
# Create a checkpoint
Checkpoint-VM -Name "WEB-VM-01" -SnapshotName "Before-IIS-Install"

# List checkpoints
Get-VMCheckpoint -VMName "WEB-VM-01"

# Restore to a checkpoint
Restore-VMCheckpoint -VMName "WEB-VM-01" -Name "Before-IIS-Install" -Confirm:$false

# Remove a checkpoint (keeps current state)
Remove-VMCheckpoint -VMName "WEB-VM-01" -Name "Before-IIS-Install"
```

### 3.6 Start / Stop / Export VMs

```powershell
# Start a VM
Start-VM -Name "WEB-VM-01"

# Graceful shutdown (requires integration services)
Stop-VM -Name "WEB-VM-01" -Force:$false

# Force power off
Stop-VM -Name "WEB-VM-01" -TurnOff

# Save (hibernate) a VM
Save-VM -Name "WEB-VM-01"

# Export a VM (for backup or migration)
Export-VM -Name "WEB-VM-01" -Path "C:\Hyper-V\Exports"

# Import a VM
Import-VM -Path "C:\Hyper-V\Exports\WEB-VM-01\Virtual Machines\*.vmcx" -Copy -GenerateNewId
```

### 3.7 Live Migration Overview

Live Migration allows moving running VMs between Hyper-V hosts with zero downtime:

1. **Prerequisites:**
   - Both hosts must be in the same (or trusted) domain.
   - Shared storage (SAN, SMB, or Cluster Shared Volumes) or use live migration with shared nothing.
   - Identical processor manufacturer (Intel ↔ Intel, AMD ↔ AMD).

2. **Enable Live Migration:**

```powershell
Enable-VMMigration
Set-VMMigrationNetwork -Subnet "10.0.0.0/24"
Set-VMHost -VirtualMachineMigrationAuthenticationType Kerberos
```

3. **Perform a Live Migration:**

```powershell
Move-VM -Name "WEB-VM-01" -DestinationHost "HYPERV-02.corp.local"
```

### 3.8 Nested Virtualization

Run Hyper-V inside a VM (useful for dev/test):

```powershell
# Enable on the host (VM must be stopped)
Set-VMProcessor -VMName "WEB-VM-01" -ExposeVirtualizationExtensions $true

# Enable MAC address spoofing (required for nested VM networking)
Set-VMNetworkAdapter -VMName "WEB-VM-01" -MacAddressSpoofing On
```

### Useful Hyper-V Commands

| Command | Description |
|---------|-------------|
| `Get-VM` | List all VMs |
| `Get-VM -Name "VM" \| Select-Object *` | Detailed VM info |
| `New-VM` | Create a new VM |
| `Start-VM` / `Stop-VM` | Start or stop a VM |
| `Restart-VM` | Restart a VM |
| `Checkpoint-VM` | Create a snapshot |
| `Restore-VMCheckpoint` | Restore from snapshot |
| `Export-VM` / `Import-VM` | Export or import a VM |
| `Get-VMSwitch` | List virtual switches |
| `New-VMSwitch` | Create a virtual switch |
| `Get-VHD` | Get virtual hard disk info |
| `Resize-VHD` | Resize a virtual disk |
| `Measure-VM` | Resource usage statistics |

---

---

## 4. Group Policy Advanced Configuration

> **Objective:** Create and configure advanced GPOs for security settings, software restriction, drive mappings, printer deployment, Windows Update, and PowerShell script deployment.

### 4.1 Create GPOs for Security Settings

```powershell
Import-Module GroupPolicy

# Create a GPO for password and lockout policy
New-GPO -Name "Security-PasswordPolicy" |
    New-GPLink -Target "DC=corp,DC=local"

# Configure via Group Policy Management Console (GPMC):
# Computer Configuration → Policies → Windows Settings →
# Security Settings → Account Policies → Password Policy
```

Key security settings to configure:

| Setting | Path | Recommended Value |
|---------|------|-------------------|
| Min password length | Account Policies → Password Policy | 14 characters |
| Password complexity | Account Policies → Password Policy | Enabled |
| Max password age | Account Policies → Password Policy | 90 days |
| Account lockout threshold | Account Policies → Account Lockout | 5 attempts |
| Lockout duration | Account Policies → Account Lockout | 30 minutes |
| Interactive logon banner | Local Policies → Security Options | Custom warning text |
| Audit logon events | Local Policies → Audit Policy | Success, Failure |

### 4.2 Software Restriction Policies

```powershell
# Create a GPO for software restrictions
New-GPO -Name "Security-AppRestrictions" |
    New-GPLink -Target "OU=Workstations,OU=IT,DC=corp,DC=local"

# Configure via GPMC:
# Computer Configuration → Policies → Windows Settings →
# Security Settings → Software Restriction Policies
# Or use AppLocker:
# Computer Configuration → Policies → Windows Settings →
# Security Settings → Application Control Policies → AppLocker
```

**AppLocker example rules via PowerShell:**

```powershell
# Get current AppLocker policy
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections

# Create a rule that allows executables only from Program Files
$Rule = New-AppLockerPolicy -RuleType Path -RuleNamePrefix "Allow" `
    -User "Everyone" -AllowWindows
```

### 4.3 Drive Mappings via GPO

Configure in GPMC under:

**User Configuration → Preferences → Windows Settings → Drive Maps**

```
Action: Create
Location: \\fileserver\shared
Drive Letter: S:
Label: Shared Drive
Reconnect: ✅
Item-Level Targeting: Security Group = "Domain Users"
```

Alternatively, deploy via logon script:

```powershell
# logon_drives.ps1 — deployed via GPO
New-PSDrive -Name "S" -PSProvider FileSystem -Root "\\fileserver\shared" -Persist
New-PSDrive -Name "P" -PSProvider FileSystem -Root "\\fileserver\projects" -Persist
```

### 4.4 Printer Deployment via GPO

```powershell
# Deploy a printer via GPO (Computer Configuration)
# Computer Configuration → Policies → Windows Settings →
# Deployed Printers → Deploy Printer Connection

# Or using PowerShell on the print server:
# Share the printer
Set-Printer -Name "OfficeHP" -Shared $true -ShareName "OfficeHP"

# Deploy via Group Policy Preferences:
# User Configuration → Preferences → Control Panel Settings → Printers
# Action: Create, Shared Path: \\printserver\OfficeHP
```

### 4.5 Windows Update Configuration via GPO

```powershell
New-GPO -Name "WSUS-UpdatePolicy" |
    New-GPLink -Target "OU=Servers,OU=IT,DC=corp,DC=local"
```

Configure in GPMC:

**Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Update**

| Setting | Value |
|---------|-------|
| Configure Automatic Updates | Enabled — 4 (Auto download and schedule install) |
| Scheduled install day | 0 (Every day) or 1 (Sunday) |
| Scheduled install time | 03:00 |
| Specify intranet update service | `http://wsus-server:8530` |
| No auto-restart during active hours | 06:00 – 22:00 |
| Defer feature updates | 180 days |
| Defer quality updates | 14 days |

### 4.6 PowerShell Script Deployment via GPO

**Startup Script (Computer):**

```
Computer Configuration → Policies → Windows Settings → Scripts → Startup
```

**Logon Script (User):**

```
User Configuration → Policies → Windows Settings → Scripts → Logon
```

```powershell
# Example startup script deployed via GPO
# Save to \\corp.local\SYSVOL\corp.local\Policies\{GPO-GUID}\Machine\Scripts\Startup\

# startup_config.ps1
Set-ExecutionPolicy RemoteSigned -Force -Scope LocalMachine
Enable-PSRemoting -Force -SkipNetworkProfileCheck
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
```

### 4.7 GPO Inheritance and Blocking

```powershell
# View GPO inheritance for an OU
Get-GPInheritance -Target "OU=IT,DC=corp,DC=local"

# Block inheritance on an OU (blocks parent GPOs)
Set-GPInheritance -Target "OU=IT,DC=corp,DC=local" -IsBlocked Yes

# Enforce a GPO (overrides blocking)
Set-GPLink -Name "Security-PasswordPolicy" -Target "DC=corp,DC=local" -Enforced Yes
```

**GPO Processing Order:** Local → Site → Domain → OU (LSDOU). The last applied wins.

### 4.8 Troubleshooting GPOs

```powershell
# Force immediate GPO update
gpupdate /force

# Generate RSoP (Resultant Set of Policy) report
gpresult /r              # Summary to console
gpresult /h C:\Reports\gpresult.html   # HTML report
gpresult /scope computer /h C:\Reports\gpresult_computer.html

# Check GPO replication
repadmin /replsummary

# View Group Policy event logs
Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" -MaxEvents 50
```

### Common GPO Settings Reference

| Category | Setting | Path |
|----------|---------|------|
| Password Policy | Minimum length | Computer → Account Policies |
| Account Lockout | Lockout threshold | Computer → Account Policies |
| Audit Policy | Logon events | Computer → Local Policies |
| User Rights | Log on locally | Computer → Local Policies |
| Firewall | Domain profile | Computer → Admin Templates → Network |
| Windows Update | Auto update config | Computer → Admin Templates → Windows Update |
| Drive Mappings | Map network drives | User → Preferences → Drive Maps |
| Printers | Deploy printers | User → Preferences → Printers |
| Scripts | Startup/Logon scripts | Computer/User → Scripts |
| Desktop | Wallpaper, screen saver | User → Admin Templates → Desktop |

---

---

## 5. Windows Server Clustering (Failover Cluster)

> **Objective:** Set up a Windows Failover Cluster for high availability, including validation, quorum configuration, clustered roles, and failover testing.

### 5.1 Prerequisites and Requirements

| Requirement | Details |
|-------------|---------|
| **Nodes** | Minimum 2 Windows Server 2022 servers (same edition) |
| **Domain** | All nodes must be domain-joined |
| **Network** | At least 2 NICs per node (cluster + heartbeat) |
| **Storage** | Shared storage (SAN/iSCSI) or Storage Spaces Direct |
| **DNS** | Cluster name registered in DNS |
| **Updates** | All nodes at the same patch level |

### 5.2 Install the Failover Clustering Feature

```powershell
# Install on all cluster nodes
$Nodes = @("NODE-01", "NODE-02")
$Nodes | ForEach-Object {
    Invoke-Command -ComputerName $_ -ScriptBlock {
        Install-WindowsFeature Failover-Clustering -IncludeManagementTools
    }
}
```

### 5.3 Validate the Cluster Configuration

```powershell
# Run validation tests (CRITICAL — always validate before creating)
Test-Cluster -Node "NODE-01", "NODE-02" -ReportName "C:\ClusterReports\ValidationReport"

# View the HTML report
Start-Process "C:\ClusterReports\ValidationReport.htm"
```

> ⚠️ Address all **Errors** before proceeding. Warnings should be reviewed but may not block cluster creation.

### 5.4 Create the Cluster

```powershell
# Create the cluster with a static IP
New-Cluster -Name "WEB-CLUSTER" `
    -Node "NODE-01", "NODE-02" `
    -StaticAddress 192.168.1.100 `
    -NoStorage   # Add storage after creation

# Verify
Get-Cluster
Get-ClusterNode
```

### 5.5 Configure Cluster Quorum

```powershell
# Node Majority (for odd number of nodes)
Set-ClusterQuorum -NodeMajority

# Node and Disk Majority (with witness disk)
Set-ClusterQuorum -NodeAndDiskMajority "Cluster Disk 1"

# Node and File Share Majority (recommended for even nodes)
Set-ClusterQuorum -NodeAndFileShareMajority "\\fileserver\clusterwitness"

# Cloud Witness (Azure — recommended for modern deployments)
Set-ClusterQuorum -CloudWitness `
    -AccountName "mystorageaccount" `
    -AccessKey "your-azure-storage-access-key" `
    -Endpoint "core.windows.net"

# View current quorum config
Get-ClusterQuorum
```

| Quorum Mode | Best For |
|-------------|----------|
| Node Majority | Odd number of nodes |
| Node + Disk Majority | Even nodes with shared storage |
| Node + File Share Majority | Even nodes, no shared storage |
| Cloud Witness | Multi-site / Azure hybrid |

### 5.6 Add Clustered Roles and Services

```powershell
# Add a Generic Service role (e.g., a Windows service)
Add-ClusterGenericServiceRole -ServiceName "MyAppService" `
    -Name "MyApp-HA" -StaticAddress 192.168.1.101

# Add a File Server role
Add-ClusterFileServerRole -Name "FS-HA" `
    -Storage "Cluster Disk 1" `
    -StaticAddress 192.168.1.102

# Add a Virtual Machine role (Hyper-V VM high availability)
Add-ClusterVirtualMachineRole -VMName "WEB-VM-01"

# List clustered roles
Get-ClusterGroup
```

### 5.7 Test Failover

```powershell
# Move a clustered role to another node
Move-ClusterGroup -Name "MyApp-HA" -Node "NODE-02"

# Simulate a node failure (drain roles from a node)
Suspend-ClusterNode -Name "NODE-01" -Drain

# Resume the node
Resume-ClusterNode -Name "NODE-01"

# Verify role owner
Get-ClusterGroup | Format-Table Name, OwnerNode, State
```

### 5.8 Cluster-Aware Updating (CAU)

```powershell
# Install CAU role
Install-WindowsFeature RSAT-Clustering-AutomationServer

# Preview applicable updates
Invoke-CauScan -ClusterName "WEB-CLUSTER"

# Apply updates to the cluster (rolling update, one node at a time)
Invoke-CauRun -ClusterName "WEB-CLUSTER" -Force

# Configure self-updating (automatic monthly updates)
Add-CauClusterRole -ClusterName "WEB-CLUSTER" `
    -DaysOfWeek Tuesday `
    -WeeksOfMonth Second `
    -MaxRetriesPerNode 3 `
    -RequireAllNodesOnline
```

### 5.9 Monitoring Cluster Health

```powershell
# Overall cluster health
Get-Cluster | Format-List *

# Node status
Get-ClusterNode | Format-Table Name, State, DynamicWeight

# Resource status
Get-ClusterResource | Format-Table Name, ResourceType, OwnerGroup, State

# Cluster events
Get-ClusterLog -Destination "C:\ClusterLogs" -TimeSpan 60
Get-WinEvent -LogName "Microsoft-Windows-FailoverClustering/Operational" -MaxEvents 20
```

---

---

## 6. Certificate Services (AD CS)

> **Objective:** Deploy an Enterprise Certificate Authority, create and manage certificate templates, configure auto-enrollment, and handle certificate lifecycle management.

### 6.1 Install AD Certificate Services

```powershell
Install-WindowsFeature AD-Certificate -IncludeManagementTools -IncludeAllSubFeature
```

### 6.2 Configure an Enterprise Root CA

```powershell
Install-AdcsCertificationAuthority `
    -CAType EnterpriseRootCA `
    -CACommonName "Corp-Root-CA" `
    -KeyLength 4096 `
    -HashAlgorithmName SHA256 `
    -ValidityPeriod Years `
    -ValidityPeriodUnits 10 `
    -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" `
    -Force
```

> 💡 **Best Practice:** In production, use a two-tier PKI: an offline Root CA and an online Subordinate (Issuing) CA.

### 6.3 Configure Web Enrollment (Optional)

```powershell
Install-AdcsWebEnrollment -Force

# Access at: https://ca-server/certsrv
```

### 6.4 Create Certificate Templates

Certificate templates are managed through the `certtmpl.msc` snap-in or PowerShell:

1. Open **Certificate Authority** management console.
2. Right-click **Certificate Templates** → **Manage**.
3. Duplicate an existing template (e.g., "Web Server").
4. Configure:

| Tab | Setting | Value |
|-----|---------|-------|
| General | Template Name | `Corp-WebServer` |
| General | Validity Period | 2 years |
| Request Handling | Purpose | Signature and encryption |
| Subject Name | Supply in the request | ✅ |
| Security | Enroll permission | Domain Computers, Web Servers group |
| Cryptography | Key Size | 2048 |
| Cryptography | Hash Algorithm | SHA256 |

5. Publish the template:

```powershell
# Add the template to the CA for issuance
certutil -setcatemplates +Corp-WebServer
```

### 6.5 Request and Enroll Certificates

```powershell
# Create an INF file for the certificate request
$InfContent = @"
[Version]
Signature="`$Windows NT$"

[NewRequest]
Subject = "CN=webserver.corp.local"
KeySpec = 1
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
SMIME = FALSE
PrivateKeyArchive = FALSE
UserProtected = FALSE
UseExistingKeySet = FALSE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = CMC
KeyUsage = 0xa0

[EnhancedKeyUsageExtension]
OID=1.3.6.1.5.5.7.3.1 ; Server Authentication

[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=webserver.corp.local&"
_continue_ = "dns=webserver&"
"@

$InfContent | Out-File "C:\Certs\webrequest.inf" -Encoding ASCII

# Generate the request
certreq -new "C:\Certs\webrequest.inf" "C:\Certs\webrequest.req"

# Submit to the CA
certreq -submit -config "CA-SERVER\Corp-Root-CA" "C:\Certs\webrequest.req" "C:\Certs\webserver.cer"

# Accept and install the certificate
certreq -accept "C:\Certs\webserver.cer"
```

### 6.6 Configure Auto-Enrollment via GPO

```powershell
New-GPO -Name "PKI-AutoEnrollment" |
    New-GPLink -Target "DC=corp,DC=local"
```

Configure in GPMC:

**Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client — Auto-Enrollment**

| Setting | Value |
|---------|-------|
| Configuration Model | Enabled |
| Renew expired certificates | ✅ |
| Update certificates that use templates | ✅ |

Also enable for User certificates:

**User Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client — Auto-Enrollment** → Enabled

### 6.7 Manage Certificate Revocation (CRL)

```powershell
# View CRL configuration
certutil -getreg ca\CRLPeriod
certutil -getreg ca\CRLPeriodUnits
certutil -getreg ca\CRLDeltaPeriod

# Publish a new CRL
certutil -CRL

# Revoke a specific certificate (by serial number)
certutil -revoke <SerialNumber> 1
# Reason codes: 0=Unspecified, 1=KeyCompromise, 2=CACompromise,
# 3=AffiliationChanged, 4=Superseded, 5=CessationOfOperation

# View revoked certificates
certutil -view -restrict "Disposition=21" -out "SerialNumber,CommonName,NotAfter"
```

### 6.8 Export / Import Certificates

```powershell
# Export certificate with private key (PFX)
$Cert = Get-ChildItem Cert:\LocalMachine\My |
    Where-Object Subject -like "*webserver.corp.local*"
$PfxPassword = ConvertTo-SecureString "ExportP@ss!" -AsPlainText -Force
Export-PfxCertificate -Cert $Cert -FilePath "C:\Certs\webserver.pfx" -Password $PfxPassword

# Export certificate only (CER)
Export-Certificate -Cert $Cert -FilePath "C:\Certs\webserver.cer"

# Import certificate on another server
Import-PfxCertificate -FilePath "C:\Certs\webserver.pfx" `
    -CertStoreLocation Cert:\LocalMachine\My `
    -Password $PfxPassword
```

### PKI Best Practices

| Practice | Recommendation |
|----------|----------------|
| CA Hierarchy | Two-tier (offline Root CA + online Issuing CA) |
| Key Length | Minimum 2048-bit RSA or 256-bit ECC |
| Hash Algorithm | SHA-256 or stronger |
| CRL Publication | Every 1–7 days; Delta CRL daily |
| Template Permissions | Least-privilege enrollment permissions |
| Backup | Regularly backup CA database and private key |
| Monitoring | Alert on expiring certificates (30/60/90 days) |
| Auto-enrollment | Use GPO for domain computers and users |

---

---

## 7. PowerShell Desired State Configuration (DSC)

> **Objective:** Use DSC to declaratively define server configurations, apply them, verify compliance, and understand push vs. pull deployment modes.

### 7.1 DSC Concepts

| Concept | Description |
|---------|-------------|
| **Configuration** | A PowerShell script block that defines the desired state |
| **Resource** | A module that knows how to configure a specific component (e.g., File, WindowsFeature, Service) |
| **MOF File** | Managed Object Format — compiled output from a configuration |
| **LCM** | Local Configuration Manager — the DSC engine on each node |
| **Push Mode** | Admin pushes configuration to nodes on demand |
| **Pull Mode** | Nodes periodically pull configuration from a DSC Pull Server |

### 7.2 Write a Basic DSC Configuration

```powershell
# WebServerConfig.ps1 — Ensure IIS is installed and running

Configuration WebServerConfig {
    param (
        [string[]]$NodeName = 'localhost'
    )

    Import-DscResource -ModuleName PSDesiredStateConfiguration

    Node $NodeName {

        # Ensure IIS Web Server role is installed
        WindowsFeature IIS {
            Ensure = 'Present'
            Name   = 'Web-Server'
        }

        # Ensure IIS Management Tools are installed
        WindowsFeature IISManagement {
            Ensure    = 'Present'
            Name      = 'Web-Mgmt-Tools'
            DependsOn = '[WindowsFeature]IIS'
        }

        # Ensure ASP.NET 4.5 is installed
        WindowsFeature ASPNET45 {
            Ensure    = 'Present'
            Name      = 'Web-Asp-Net45'
            DependsOn = '[WindowsFeature]IIS'
        }

        # Ensure the W3SVC service is running
        Service W3SVC {
            Name      = 'W3SVC'
            State     = 'Running'
            StartType = 'Automatic'
            DependsOn = '[WindowsFeature]IIS'
        }

        # Ensure the website directory exists
        File WebsiteDir {
            Ensure          = 'Present'
            Type            = 'Directory'
            DestinationPath = 'C:\inetpub\myapp'
        }

        # Ensure an index.html exists
        File IndexPage {
            Ensure          = 'Present'
            Type            = 'File'
            DestinationPath = 'C:\inetpub\myapp\index.html'
            Contents        = '<!DOCTYPE html><html><body><h1>DSC Configured Server</h1></body></html>'
            DependsOn       = '[File]WebsiteDir'
        }

        # Ensure Telnet Client is NOT installed
        WindowsFeature TelnetClient {
            Ensure = 'Absent'
            Name   = 'Telnet-Client'
        }
    }
}
```

### 7.3 Compile and Apply the Configuration (Push Mode)

```powershell
# Step 1: Dot-source the configuration script
. .\WebServerConfig.ps1

# Step 2: Compile (generates MOF files in .\WebServerConfig\)
WebServerConfig -NodeName 'localhost'

# Step 3: Apply the configuration
Start-DscConfiguration -Path .\WebServerConfig -Wait -Verbose -Force
```

### 7.4 Verify Compliance

```powershell
# Test if current state matches desired state
Test-DscConfiguration -Detailed

# Get the current applied configuration
Get-DscConfiguration

# Get LCM state
Get-DscLocalConfigurationManager
```

| Test-DscConfiguration Output | Meaning |
|------------------------------|---------|
| `InDesiredState: True` | Node is compliant |
| `InDesiredState: False` | Node has drifted — resources list shows which |

### 7.5 Configure the Local Configuration Manager (LCM)

```powershell
[DSCLocalConfigurationManager()]
Configuration LCMConfig {
    Node 'localhost' {
        Settings {
            RefreshMode                    = 'Push'
            ConfigurationMode              = 'ApplyAndAutoCorrect'
            ConfigurationModeFrequencyMins = 30
            RebootNodeIfNeeded             = $true
            ActionAfterReboot              = 'ContinueConfiguration'
        }
    }
}

# Compile and apply LCM settings
LCMConfig
Set-DscLocalConfigurationManager -Path .\LCMConfig -Verbose
```

| LCM Setting | Options | Description |
|-------------|---------|-------------|
| `RefreshMode` | Push, Pull, Disabled | How the node gets configuration |
| `ConfigurationMode` | ApplyOnly, ApplyAndMonitor, ApplyAndAutoCorrect | How drift is handled |
| `ConfigurationModeFrequencyMins` | 15–n | How often to check/correct |
| `RebootNodeIfNeeded` | True/False | Auto-reboot if config requires it |

### 7.6 Push vs. Pull Mode

| Aspect | Push Mode | Pull Mode |
|--------|-----------|-----------|
| **Initiation** | Admin pushes manually | Node pulls on a schedule |
| **Server required** | No (just a workstation) | DSC Pull Server or Azure Automation |
| **Best for** | Small environments, testing | Large environments, production |
| **Scalability** | Limited | Highly scalable |
| **Compliance reporting** | Manual | Automatic via pull server |

**Setting up a basic Pull Server:**

```powershell
# Install the DSC Service
Install-WindowsFeature DSC-Service -IncludeManagementTools

# Configure the Pull Server
Configuration PullServer {
    Import-DscResource -ModuleName PSDesiredStateConfiguration
    Import-DscResource -ModuleName xPSDesiredStateConfiguration

    Node 'localhost' {
        xDscWebService PullServer {
            Ensure                  = 'Present'
            EndpointName            = 'PSDSCPullServer'
            Port                    = 8080
            PhysicalPath            = "$env:SystemDrive\inetpub\PSDSCPullServer"
            CertificateThumbPrint   = 'AllowUnencryptedTraffic'
            ModulePath              = "$env:ProgramFiles\WindowsPowerShell\DscService\Modules"
            ConfigurationPath       = "$env:ProgramFiles\WindowsPowerShell\DscService\Configuration"
            RegistrationKeyPath     = "$env:ProgramFiles\WindowsPowerShell\DscService\RegistrationKeys"
            UseSecurityBestPractices = $false
        }
    }
}
```

### 7.7 Practical Example: Full Web Server Configuration with DSC

```powershell
Configuration ProductionWebServer {
    param (
        [string[]]$NodeName = 'localhost',
        [string]$SiteName   = 'ProductionSite',
        [int]$SitePort      = 80
    )

    Import-DscResource -ModuleName PSDesiredStateConfiguration

    Node $NodeName {

        # ---- Roles & Features ----
        WindowsFeature IIS {
            Ensure = 'Present'
            Name   = 'Web-Server'
        }

        WindowsFeature IISMgmt {
            Ensure    = 'Present'
            Name      = 'Web-Mgmt-Tools'
            DependsOn = '[WindowsFeature]IIS'
        }

        WindowsFeature ASPNET {
            Ensure    = 'Present'
            Name      = 'Web-Asp-Net45'
            DependsOn = '[WindowsFeature]IIS'
        }

        WindowsFeature URLAuth {
            Ensure    = 'Present'
            Name      = 'Web-Url-Auth'
            DependsOn = '[WindowsFeature]IIS'
        }

        WindowsFeature RequestFiltering {
            Ensure    = 'Present'
            Name      = 'Web-Filtering'
            DependsOn = '[WindowsFeature]IIS'
        }

        # ---- Remove Unwanted Features ----
        WindowsFeature TelnetClient {
            Ensure = 'Absent'
            Name   = 'Telnet-Client'
        }

        WindowsFeature FTP {
            Ensure = 'Absent'
            Name   = 'Web-Ftp-Server'
        }

        # ---- File System ----
        File SiteDirectory {
            Ensure          = 'Present'
            Type            = 'Directory'
            DestinationPath = "C:\inetpub\$SiteName"
        }

        File LogDirectory {
            Ensure          = 'Present'
            Type            = 'Directory'
            DestinationPath = "C:\Logs\$SiteName"
        }

        File IndexPage {
            Ensure          = 'Present'
            Type            = 'File'
            DestinationPath = "C:\inetpub\$SiteName\index.html"
            Contents        = "<!DOCTYPE html><html><head><title>$SiteName</title></head><body><h1>$SiteName is running!</h1><p>Configured via DSC</p></body></html>"
            DependsOn       = '[File]SiteDirectory'
        }

        # ---- Services ----
        Service IISService {
            Name      = 'W3SVC'
            State     = 'Running'
            StartType = 'Automatic'
            DependsOn = '[WindowsFeature]IIS'
        }

        Service WAS {
            Name      = 'WAS'
            State     = 'Running'
            StartType = 'Automatic'
            DependsOn = '[WindowsFeature]IIS'
        }

        # ---- Firewall (via Script resource) ----
        Script FirewallHTTP {
            GetScript = {
                $Rule = Get-NetFirewallRule -DisplayName "DSC-Allow-HTTP" -ErrorAction SilentlyContinue
                @{ Result = if ($Rule) { "Present" } else { "Absent" } }
            }
            TestScript = {
                $null -ne (Get-NetFirewallRule -DisplayName "DSC-Allow-HTTP" -ErrorAction SilentlyContinue)
            }
            SetScript = {
                New-NetFirewallRule -DisplayName "DSC-Allow-HTTP" `
                    -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
            }
        }

        # ---- Registry: Disable IE Enhanced Security ----
        Registry DisableIEESC {
            Ensure    = 'Present'
            Key       = 'HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}'
            ValueName = 'IsInstalled'
            ValueData = '0'
            ValueType = 'DWord'
        }
    }
}

# Compile
ProductionWebServer -NodeName 'localhost' -SiteName 'ProductionSite' -SitePort 80

# Apply
Start-DscConfiguration -Path .\ProductionWebServer -Wait -Verbose -Force

# Verify
Test-DscConfiguration -Detailed
```

### 7.8 Create Custom DSC Resources — Overview

Custom DSC resources extend DSC with your own logic:

1. **Create module folder:** `$env:ProgramFiles\WindowsPowerShell\Modules\MyDscResource`
2. **Create the resource schema (MOF):**

```
[ClassVersion("1.0.0"), FriendlyName("MyCustomResource")]
class MyCustomResource : OMI_BaseResource
{
    [Key] String Name;
    [Write, ValueMap{"Present","Absent"}, Values{"Present","Absent"}] String Ensure;
    [Write] String Setting;
};
```

3. **Create the resource module (.psm1)** with three mandatory functions:
   - `Get-TargetResource` — Returns current state
   - `Set-TargetResource` — Applies desired state
   - `Test-TargetResource` — Returns `$true` if state matches

4. **Create the module manifest (.psd1)** with `DscResourcesToExport`.

---

## Quick Reference — Essential Windows Server Commands

| Category | Command | Description |
|----------|---------|-------------|
| **System** | `systeminfo` | Display detailed system info |
| **System** | `Get-ComputerInfo` | PowerShell system info |
| **System** | `Rename-Computer -NewName "SRV"` | Change hostname |
| **System** | `Restart-Computer -Force` | Reboot server |
| **Network** | `Get-NetAdapter` | List NICs |
| **Network** | `New-NetIPAddress` | Set static IP |
| **Network** | `Set-DnsClientServerAddress` | Set DNS servers |
| **Network** | `Test-NetConnection -Port 443` | Test port connectivity |
| **Firewall** | `Get-NetFirewallProfile` | Firewall status |
| **Firewall** | `New-NetFirewallRule` | Create firewall rule |
| **Users** | `Get-LocalUser` | List local users |
| **Users** | `New-LocalUser` | Create local user |
| **Users** | `Get-ADUser -Filter *` | List AD users |
| **Users** | `New-ADUser` | Create AD user |
| **Roles** | `Get-WindowsFeature` | List roles/features |
| **Roles** | `Install-WindowsFeature` | Install role/feature |
| **IIS** | `iisreset` | Restart IIS |
| **IIS** | `Get-Website` | List IIS websites |
| **DNS** | `Resolve-DnsName` | DNS lookup |
| **DNS** | `Add-DnsServerResourceRecordA` | Add A record |
| **DHCP** | `Get-DhcpServerv4Scope` | List DHCP scopes |
| **DHCP** | `Get-DhcpServerv4Lease` | View DHCP leases |
| **Disks** | `Get-Disk` / `Get-Volume` | View disks/volumes |
| **Disks** | `Initialize-Disk` | Initialize new disk |
| **AD** | `Get-ADDomain` | Domain info |
| **AD** | `Get-ADDomainController` | DC info |
| **GPO** | `gpupdate /force` | Force GPO refresh |
| **GPO** | `gpresult /r` | GPO results |
| **Cluster** | `Get-Cluster` | Cluster info |
| **Cluster** | `Get-ClusterNode` | List cluster nodes |
| **Hyper-V** | `Get-VM` | List VMs |
| **Hyper-V** | `Start-VM` / `Stop-VM` | Start or stop VM |
| **Certs** | `certutil -store My` | List personal certs |
| **Certs** | `certutil -CRL` | Publish CRL |
| **DSC** | `Start-DscConfiguration` | Apply DSC config |
| **DSC** | `Test-DscConfiguration` | Check compliance |
| **Backup** | `wbadmin get versions` | List backup versions |
| **Logs** | `Get-WinEvent -LogName System` | View system events |
| **Remote** | `Enter-PSSession` | Interactive remote shell |
| **Remote** | `Invoke-Command` | Run command remotely |
| **Services** | `Get-Service` | List services |
| **Services** | `Restart-Service` | Restart a service |
| **Updates** | `Install-WindowsUpdate` | Install updates |

---

> **✅ Checkpoint:** Completing all 7 hard scenarios gives you production-grade skills in IIS reverse proxying, server hardening, Hyper-V, advanced Group Policy, failover clustering, PKI, and DSC. You are now ready for real-world Windows Server infrastructure roles!
