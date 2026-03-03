[🏠 Home](../README.md) · [Servers](README.md)

# 🪟 IIS (Internet Information Services) — Advanced Configuration & Production Guide

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Architecture, Production Config, Gotchas  
> **Covers:** Reverse Proxy (ARR), SSL/TLS, Static Hosting, ASP.NET, URL Rewrite, Application Pools, Hardening  
> **Platform:** Windows Server 2019/2022 | **Management:** IIS Manager GUI, PowerShell, appcmd.exe  
> **Tip:** Sections build on each other — read sequentially first, then use TOC for reference.

---

## Table of Contents

1. [Architecture & Mental Model](#1-architecture--mental-model)
2. [Installation & Feature Layout](#2-installation--feature-layout)
3. [Core Configuration Anatomy](#3-core-configuration-anatomy)
4. [Sites, Bindings & Virtual Hosts](#4-sites-bindings--virtual-hosts)
5. [Application Pools (AppPools) — The Heart of IIS](#5-application-pools-apppools--the-heart-of-iis)
6. [Static Site Hosting](#6-static-site-hosting)
7. [Reverse Proxy (ARR + URL Rewrite)](#7-reverse-proxy-arr--url-rewrite)
8. [HTTPS & SSL/TLS Certificate Installation](#8-https--ssltls-certificate-installation)
9. [Server-Side Scripting — ASP.NET, PHP, Node.js](#9-server-side-scripting--aspnet-php-nodejs)
10. [URL Rewrite Module](#10-url-rewrite-module)
11. [Load Balancing (ARR Server Farms)](#11-load-balancing-arr-server-farms)
12. [Caching & Performance Tuning](#12-caching--performance-tuning)
13. [Security Hardening](#13-security-hardening)
14. [Logging, Debugging & Monitoring](#14-logging-debugging--monitoring)
15. [Production Gotchas & Tricky Q&A](#15-production-gotchas--tricky-qa)

---

## 1. Architecture & Mental Model

### 1.1 What Is IIS?

IIS (Internet Information Services) is Microsoft's **enterprise-grade web server** built into Windows Server. It's deeply integrated with the Windows ecosystem — Active Directory, .NET, Windows Authentication, Certificate Store, and PowerShell.

### 1.2 Architecture Diagram

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                    IIS ARCHITECTURE                                 │
  │                                                                     │
  │  ┌───────────────────────────────────────┐                          │
  │  │         HTTP.sys (Kernel-Mode)        │  ← Kernel-level HTTP     │
  │  │  • Listens on ports 80/443            │     listener             │
  │  │  • Handles TCP connections            │  ← Routes to correct     │
  │  │  • SSL/TLS termination (kernel)       │     application pool     │
  │  │  • Response caching (kernel)          │  ← Kernel-mode cache     │
  │  │  • Request queuing per AppPool        │     = blazing fast       │
  │  └────────────────┬──────────────────────┘                          │
  │                   │  Passes request to user-mode                    │
  │  ┌────────────────▼──────────────────────┐                          │
  │  │  W3SVC (World Wide Web Service)       │  ← Windows Service       │
  │  │  • Manages worker processes           │     (svc host)           │
  │  │  • Monitors health                    │                          │
  │  │  • Configuration management           │                          │
  │  └────────────────┬──────────────────────┘                          │
  │                   │  Activates worker process                       │
  │  ┌────────────────▼──────────────────────┐                          │
  │  │  Worker Process (w3wp.exe)            │  ← One per AppPool       │
  │  │                                       │                          │
  │  │  ┌─────────────────────────────────┐  │                          │
  │  │  │  IIS Module Pipeline            │  │                          │
  │  │  │  ┌──────┐ ┌──────┐ ┌────────┐   │  │                          │
  │  │  │  │Auth  │→│Static│→│Routing │   │  │                          │
  │  │  │  │Module│ │File  │ │Module  │   │  │                          │
  │  │  │  └──────┘ │Module│ └───┬────┘   │  │                          │
  │  │  │           └──────┘     │        │  │                          │
  │  │  │              ┌─────────▼─────┐  │  │                          │
  │  │  │              │  Handler      │  │  │                          │
  │  │  │              │  (ASP.NET /   │  │  │                          │
  │  │  │              │   PHP / CGI / │  │  │                          │
  │  │  │              │   Static)     │  │  │                          │
  │  │  │              └───────────────┘  │  │                          │
  │  │  └─────────────────────────────────┘  │                          │
  │  └───────────────────────────────────────┘                          │
  └─────────────────────────────────────────────────────────────────────┘
```

### 1.3 HTTP.sys — Why IIS Is Different

Unlike Nginx/Apache (user-mode), IIS delegates connection handling to **HTTP.sys**, a kernel-mode driver.

```
  Nginx / Apache                         IIS
  ┌──────────────────────────┐            ┌──────────────────────────┐
  │  User Mode               │            │  Kernel Mode (Ring 0)    │
  │  ┌────────────────────┐  │            │  ┌────────────────────┐  │
  │  │ nginx / httpd      │  │            │  │ HTTP.sys           │  │
  │  │ • TCP listen       │  │            │  │ • TCP listen       │  │
  │  │ • SSL termination  │  │            │  │ • SSL termination  │  │
  │  │ • Request parsing  │  │            │  │ • Request parsing  │  │
  │  │ • Response caching │  │            │  │ • Response caching │  │
  │  └────────────────────┘  │            │  └────────┬───────────┘  │
  │                          │            │           │              │
  │  Everything in user mode │            │  User Mode (Ring 3)      │
  │  = context switches      │            │  ┌────────▼───────────┐  │
  │  = more overhead         │            │  │ w3wp.exe           │  │
  │                          │            │  │ • App code only    │  │
  │                          │            │  │ • Module pipeline  │  │
  │                          │            │  └────────────────────┘  │
  └──────────────────────────┘            └──────────────────────────┘

  Benefit: HTTP.sys kernel-mode caching can serve responses
  WITHOUT waking up the worker process at all.
  Static file performance can exceed Nginx in some benchmarks.
```

> **DevOps Relevance:** HTTP.sys is also used by ASP.NET Core's Kestrel (when hosted in-process on IIS), Windows Docker containers, and `HttpListener` in .NET. It's not just "the IIS thing" — it's the Windows HTTP platform.

### 1.4 IIS vs Nginx vs Apache — Comparison

| Aspect | IIS | Nginx | Apache |
|--------|-----|-------|--------|
| **Platform** | Windows only | Linux/Windows/macOS | Linux/Windows/macOS |
| **Architecture** | Kernel (HTTP.sys) + user-mode workers | Event-driven workers | Process/thread (MPM) |
| **Config format** | XML (`web.config`, `applicationHost.config`) | Custom text (`nginx.conf`) | Custom text (`httpd.conf`) |
| **GUI management** | ✅ IIS Manager | ❌ | ❌ |
| **AD integration** | ✅ Native (Windows Auth, Kerberos) | ❌ (needs LDAP proxy) | Partial (mod_auth_kerb) |
| **.NET hosting** | ✅ Native (in-process) | Reverse proxy only | Reverse proxy only |
| **PHP hosting** | ✅ FastCGI | ✅ FastCGI (PHP-FPM) | ✅ mod_php / PHP-FPM |
| **Reverse proxy** | ARR module (add-on) | Built-in | mod_proxy (built-in) |
| **Per-dir config** | `web.config` (like .htaccess) | ❌ | `.htaccess` |
| **Cost** | Windows Server license | Free | Free |

---

## 2. Installation & Feature Layout

### 2.1 Installation via Server Manager

```
  Server Manager → Manage → Add Roles and Features
  ┌──────────────────────────────────────────────────────────────┐
  │  Server Roles → Web Server (IIS) ✅                          │
  │                                                              │
  │  Role Services to install:                                   │
  │  ├── Common HTTP Features                                    │
  │  │   ├── ✅ Default Document                                 │
  │  │   ├── ✅ Directory Browsing                               │
  │  │   ├── ✅ HTTP Errors                                      │
  │  │   └── ✅ Static Content                                   │
  │  ├── Health and Diagnostics                                  │
  │  │   ├── ✅ HTTP Logging                                     │
  │  │   └── ✅ Request Monitor                                  │
  │  ├── Performance                                             │
  │  │   ├── ✅ Static Content Compression                       │
  │  │   └── ✅ Dynamic Content Compression                      │
  │  ├── Security                                                │
  │  │   ├── ✅ Request Filtering                                │
  │  │   ├── ✅ Windows Authentication (if needed)               │
  │  │   └── ✅ Basic Authentication (if needed)                 │
  │  ├── Application Development                                 │
  │  │   ├── ✅ ASP.NET 4.8                                      │
  │  │   ├── ✅ .NET Extensibility 4.8                           │
  │  │   ├── ✅ ISAPI Extensions                                 │
  │  │   └── ✅ ISAPI Filters                                    │
  │  └── Management Tools                                        │
  │      ├── ✅ IIS Management Console                           │
  │      └── ✅ IIS Management Scripts and Tools                 │
  └──────────────────────────────────────────────────────────────┘
```

### 2.2 Installation via PowerShell

```powershell
# Install IIS with common features
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Install specific sub-features
Install-WindowsFeature -Name `
    Web-Default-Doc, `
    Web-Static-Content, `
    Web-Http-Logging, `
    Web-Request-Monitor, `
    Web-Stat-Compression, `
    Web-Dyn-Compression, `
    Web-Filtering, `
    Web-Windows-Auth, `
    Web-Net-Ext45, `
    Web-Asp-Net45, `
    Web-ISAPI-Ext, `
    Web-ISAPI-Filter, `
    Web-Mgmt-Console, `
    Web-Scripting-Tools

# Verify installation
Get-WindowsFeature Web-* | Where-Object Installed

# Import IIS PowerShell module
Import-Module WebAdministration
# or (newer):
Import-Module IISAdministration
```

### 2.3 File Layout — Mental Model

```
  C:\inetpub\                              ← IIS root directory
  ├── wwwroot\                             ← Default site document root
  ├── logs\                                ← Log files
  │   └── LogFiles\
  │       └── W3SVC1\                      ← Per-site log folder (SiteID)
  │           └── u_ex260303.log           ← Daily log file
  ├── custerr\                             ← Custom error pages
  └── temp\                                ← Temp files (compression cache)

  C:\Windows\System32\inetsrv\             ← IIS binaries & tools
  ├── appcmd.exe                           ← Command-line admin tool
  ├── config\                              ← Server-level configuration
  │   ├── applicationHost.config           ← Master config (≈ httpd.conf)
  │   ├── administration.config            ← Management delegation
  │   └── redirection.config               ← Shared config redirection
  └── InetMgr.exe                          ← IIS Manager GUI

  Per-site:
  D:\Sites\MySite\                         ← Custom site root (best practice)
  ├── web.config                           ← Per-site config (≈ .htaccess)
  ├── index.html
  └── bin\                                 ← ASP.NET compiled assemblies
```

> **⚠️ Gotcha:** Never edit `applicationHost.config` directly while IIS is running — use IIS Manager, `appcmd.exe`, or PowerShell. Direct edits can corrupt the config if IIS writes at the same time.

### 2.4 Management Tools — Three Ways to Manage

```
  ┌──────────────────────────────────────────────────────────────┐
  │                   IIS Management Stack                       │
  │                                                              │
  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐   │
  │  │ IIS Manager  │  │ PowerShell   │  │ appcmd.exe        │   │
  │  │ (GUI)        │  │              │  │                   │   │
  │  │ inetmgr.exe  │  │ WebAdmin     │  │ Legacy CLI but    │   │
  │  │              │  │ module or    │  │ still very useful │   │
  │  │ Best for:    │  │ IISAdmin     │  │                   │   │
  │  │ • Exploring  │  │ module       │  │ Best for:         │   │
  │  │ • One-off    │  │              │  │ • Quick one-liners│   │
  │  │   changes    │  │ Best for:    │  │ • Scripting       │   │
  │  │ • Learning   │  │ • Automation │  │ • When PS module  │   │
  │  │              │  │ • CI/CD      │  │   unavailable     │   │
  │  │              │  │ • DSC        │  │                   │   │
  │  └──────────────┘  └──────────────┘  └───────────────────┘   │
  │                                                              │
  │  All three modify the same file: applicationHost.config      │
  └──────────────────────────────────────────────────────────────┘
```

---

## 3. Core Configuration Anatomy

### 3.1 Configuration Hierarchy

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  IIS Configuration Hierarchy                                     │
  │                                                                  │
  │  Machine Level (applicationHost.config)                          │
  │  └── Site Level                                                  │
  │      └── Application Level (/app1, /app2)                        │
  │          └── Virtual Directory Level                              │
  │              └── web.config (per-directory overrides)             │
  │                                                                  │
  │  Inheritance: Parent settings cascade down.                      │
  │  web.config can OVERRIDE parent settings                         │
  │  (unless locked by admin in applicationHost.config)              │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.2 applicationHost.config — Structure

```xml
<!-- C:\Windows\System32\inetsrv\config\applicationHost.config -->
<configuration>
    <system.applicationHost>
        <!-- Site definitions, bindings, AppPools -->
        <sites>
            <site name="Default Web Site" id="1">
                <bindings>
                    <binding protocol="http" bindingInformation="*:80:" />
                </bindings>
                <application path="/" applicationPool="DefaultAppPool">
                    <virtualDirectory path="/" physicalPath="C:\inetpub\wwwroot" />
                </application>
            </site>
        </sites>
        <applicationPools>
            <add name="DefaultAppPool" managedRuntimeVersion="v4.0"
                 managedPipelineMode="Integrated" />
        </applicationPools>
    </system.applicationHost>

    <system.webServer>
        <!-- Global server settings (modules, handlers, security) -->
    </system.webServer>
</configuration>
```

### 3.3 web.config — Per-Site/Per-Directory Config

```xml
<!-- D:\Sites\MySite\web.config -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <defaultDocument>
            <files>
                <clear />
                <add value="index.html" />
                <add value="default.aspx" />
            </files>
        </defaultDocument>

        <httpErrors errorMode="Custom">
            <remove statusCode="404" />
            <error statusCode="404" path="/errors/404.html" responseMode="ExecuteURL" />
        </httpErrors>

        <staticContent>
            <mimeMap fileExtension=".json" mimeType="application/json" />
            <mimeMap fileExtension=".woff2" mimeType="font/woff2" />
        </staticContent>
    </system.webServer>
</configuration>
```

> **web.config is IIS's .htaccess** — same concept (per-directory overrides), same power, same performance concern. But unlike Apache, IIS caches and watches `web.config` for changes, auto-reloading without a full server restart.

### 3.4 Configuration Locking

Admins can **lock** settings so `web.config` can't override them:

```xml
<!-- In applicationHost.config — lock down security globally -->
<system.webServer>
    <security lockAllElementsExcept="requestFiltering">
        <requestFiltering>
            <requestLimits maxAllowedContentLength="30000000" />
        </requestFiltering>
    </security>
</system.webServer>
```

```powershell
# Lock a section via PowerShell
Set-WebConfigurationLock -Filter "system.webServer/security/authentication/anonymousAuthentication" -PSPath "IIS:\"

# Unlock a section
Remove-WebConfigurationLock -Filter "system.webServer/security/authentication/anonymousAuthentication" -PSPath "IIS:\"
```

> **DevOps Relevance:** In shared/multi-tenant hosting, you lock sensitive sections (authentication, handlers) at the machine level to prevent applications from overriding security policies via `web.config`.

---

## 4. Sites, Bindings & Virtual Hosts

### 4.1 IIS Site Concepts

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  IIS Object Hierarchy                                           │
  │                                                                 │
  │  Server                                                         │
  │  ├── Site ("Default Web Site")         ← Comparable to vhost    │
  │  │   ├── Binding: *:80:               ← Protocol + IP:Port:Host │
  │  │   ├── Binding: *:443:example.com    (SNI for HTTPS)          │
  │  │   │                                                          │
  │  │   ├── Application: /               ← Root app                │
  │  │   │   └── Virtual Dir: /           ← Physical path           │
  │  │   │       → C:\inetpub\wwwroot                               │
  │  │   │                                                          │
  │  │   ├── Application: /api            ← Sub-application         │
  │  │   │   └── Virtual Dir: /           │  (can have own AppPool) │
  │  │   │       → D:\apps\api            │                         │
  │  │   │                                                          │
  │  │   └── Virtual Dir: /shared         ← Maps to different path  │
  │  │       → E:\shared-files             (no separate AppPool)    │
  │  │                                                              │
  │  ├── Site ("API Site")                                          │
  │  │   └── ...                                                    │
  │  │                                                              │
  │  └── Application Pools                                          │
  │      ├── DefaultAppPool                                         │
  │      ├── ApiAppPool                                             │
  │      └── ...                                                    │
  └─────────────────────────────────────────────────────────────────┘
```

### 4.2 Creating Sites & Bindings

**IIS Manager:** Sites → Add Website...

**PowerShell:**

```powershell
Import-Module WebAdministration

# Create a new site
New-Website -Name "MySite" `
    -PhysicalPath "D:\Sites\MySite" `
    -BindingInformation "*:80:mysite.com" `
    -ApplicationPool "MySitePool"

# Add additional bindings
New-WebBinding -Name "MySite" -Protocol "http" -HostHeader "www.mysite.com" -Port 80
New-WebBinding -Name "MySite" -Protocol "https" -HostHeader "mysite.com" -Port 443 -SslFlags 1

# List all sites and bindings
Get-Website | Format-Table Name, ID, State, PhysicalPath
Get-WebBinding -Name "MySite"
```

**appcmd.exe:**

```cmd
:: Create site
appcmd add site /name:"MySite" /bindings:"http/*:80:mysite.com" /physicalPath:"D:\Sites\MySite"

:: Add binding
appcmd set site /site.name:"MySite" /+bindings.[protocol='https',bindingInformation='*:443:mysite.com']

:: List sites
appcmd list sites

:: Start/Stop
appcmd start site "MySite"
appcmd stop site "MySite"
```

### 4.3 Binding Syntax Deep Dive

```
  Binding format: protocol/IP:Port:HostHeader

  Examples:
  ┌──────────────────────────────────────────────────────────────┐
  │  *:80:                  → All IPs, port 80, any hostname     │
  │  *:80:mysite.com        → All IPs, port 80, host=mysite.com  │
  │  10.0.0.1:80:           → Specific IP, port 80, any host     │
  │  *:443:mysite.com       → HTTPS with SNI (host-based SSL)    │
  │  *:443:                 → HTTPS catch-all (one cert only)    │
  └──────────────────────────────────────────────────────────────┘

  SNI (Server Name Indication):
  ┌──────────────────────────────────────────────────────────────┐
  │  Before SNI: 1 IP = 1 SSL cert (each HTTPS site needed       │
  │              its own IP address)                             │
  │                                                              │
  │  With SNI:   Client sends hostname in TLS ClientHello        │
  │              IIS picks the right cert based on hostname      │
  │              Multiple HTTPS sites share 1 IP ✅              │
  │                                                              │
  │  ⚠️ Requires: IIS 8+ and "Require Server Name Indication"    │
  │     checkbox in binding config (SslFlags = 1)                │
  └──────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** If you have two sites bound to `*:80:` (no host header), only one can start — the other gets a port conflict. Every site on the same port MUST have a unique host header binding, or use a unique IP.

---

## 5. Application Pools (AppPools) — The Heart of IIS

### 5.1 What Is an Application Pool?

An AppPool is an **isolated process boundary** (w3wp.exe) that hosts one or more web applications. It's the equivalent of a container for process isolation, recycling, and resource management.

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    APPLICATION POOL ISOLATION                   │
  │                                                                 │
  │  ┌──────────────────┐  ┌──────────────────┐                     │
  │  │  AppPool: SiteA  │  │  AppPool: SiteB  │                     │
  │  │  ┌────────────┐  │  │  ┌────────────┐  │                     │
  │  │  │ w3wp.exe   │  │  │  │ w3wp.exe   │  │                     │
  │  │  │            │  │  │  │            │  │                     │
  │  │  │ Site A     │  │  │  │ Site B     │  │                     │
  │  │  │ App /api   │  │  │  │ App /blog  │  │                     │
  │  │  │            │  │  │  │            │  │                     │
  │  │  └────────────┘  │  │  └────────────┘  │                     │
  │  │  Identity: svcA  │  │  Identity: svcB  │                     │
  │  │  .NET: v4.0      │  │  .NET: No CLR    │                     │
  │  │  Pipeline: Int.  │  │  Pipeline: Int.  │                     │
  │  └──────────────────┘  └──────────────────┘                     │
  │                                                                 │
  │  If SiteA crashes → SiteB is completely unaffected ✅           │
  │  If SiteA leaks memory → only its AppPool recycles              │
  │  Each AppPool has its own identity (security context)           │
  └─────────────────────────────────────────────────────────────────┘
```

### 5.2 AppPool Settings — Key Properties

| Setting | Default | Recommended | Why |
|---------|---------|-------------|-----|
| **.NET CLR Version** | v4.0 | `v4.0` or `No Managed Code` | `No Managed Code` for non-.NET (PHP, static, proxy) |
| **Pipeline Mode** | Integrated | Integrated | Classic only for legacy ASP apps |
| **Identity** | ApplicationPoolIdentity | Per-app service account | Least-privilege per application |
| **Start Mode** | OnDemand | AlwaysRunning | Eliminates cold-start latency |
| **Idle Timeout** | 20 min | 0 (disabled) for prod | Prevents AppPool shutdown during idle |
| **Regular Recycling** | 1740 min (29h) | Keep or customize | Recycles to clear memory leaks |
| **Rapid-Fail Protection** | 5 failures / 5 min | Tune for your app | Stops AppPool if it keeps crashing |
| **Queue Length** | 1000 | 5000 for high traffic | Requests queued when all threads busy |

### 5.3 Creating & Configuring AppPools

```powershell
Import-Module WebAdministration

# Create AppPool
New-WebAppPool -Name "MySitePool"

# Configure AppPool
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "managedRuntimeVersion" -Value "v4.0"
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "managedPipelineMode" -Value "Integrated"
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "processModel.identityType" -Value "SpecificUser"
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "processModel.userName" -Value "DOMAIN\svc-mysite"
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "processModel.password" -Value "P@ssw0rd"

# Disable idle timeout (production)
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "processModel.idleTimeout" -Value "00:00:00"

# Enable AlwaysRunning (requires Application Initialization feature)
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "startMode" -Value "AlwaysRunning"

# Set recycling schedule (recycle at 3 AM daily instead of every 29 hours)
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "recycling.periodicRestart.time" -Value "00:00:00"
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "recycling.periodicRestart.schedule" -Value @{value="03:00:00"}

# List all AppPools
Get-ChildItem "IIS:\AppPools" | Format-Table Name, State, managedRuntimeVersion
```

**appcmd.exe:**

```cmd
:: Create and configure
appcmd add apppool /name:"MySitePool" /managedRuntimeVersion:"v4.0" /managedPipelineMode:"Integrated"
appcmd set apppool "MySitePool" /processModel.idleTimeout:"00:00:00"
appcmd set apppool "MySitePool" /startMode:"AlwaysRunning"

:: Recycle an AppPool (graceful)
appcmd recycle apppool "MySitePool"

:: List AppPools
appcmd list apppools
```

### 5.4 AppPool Identity — Security Model

```
  ┌──────────────────────────────────────────────────────────────┐
  │  AppPool Identity Types (from least to most privileged):     │
  │                                                              │
  │  1. ApplicationPoolIdentity (default, recommended)           │
  │     → Virtual account: "IIS AppPool\MySitePool"              │
  │     → Automatically created, no password management          │
  │     → Minimal privileges — can read its own site files       │
  │                                                              │
  │  2. NetworkService                                           │
  │     → Built-in: NT AUTHORITY\NETWORK SERVICE                 │
  │     → Can authenticate on the network as the machine         │
  │     → Shared across all pools using it ← security risk       │
  │                                                              │
  │  3. SpecificUser (domain service account)                    │
  │     → e.g., DOMAIN\svc-webapp                                │
  │     → For AD auth, database access, network resources        │
  │     → Use gMSA (Group Managed Service Account) ← best        │
  │                                                              │
  │  4. LocalSystem                                              │
  │     → HIGHEST privilege on the machine ← NEVER use for IIS   │
  └──────────────────────────────────────────────────────────────┘
```

> **Best Practice:** Use `ApplicationPoolIdentity` for isolation. If the app needs network/DB access, use a **gMSA** (no password rotation headache). Never use `LocalSystem`.

---

## 6. Static Site Hosting

### 6.1 Basic Static Site

```powershell
# Create site directory
New-Item -Path "D:\Sites\StaticSite" -ItemType Directory

# Create AppPool (No Managed Code for static sites)
New-WebAppPool -Name "StaticPool"
Set-ItemProperty "IIS:\AppPools\StaticPool" -Name "managedRuntimeVersion" -Value ""

# Create site
New-Website -Name "StaticSite" `
    -PhysicalPath "D:\Sites\StaticSite" `
    -BindingInformation "*:80:static.example.com" `
    -ApplicationPool "StaticPool"
```

### 6.2 web.config for Static Sites

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <!-- Default documents -->
        <defaultDocument>
            <files>
                <clear />
                <add value="index.html" />
            </files>
        </defaultDocument>

        <!-- Disable directory browsing -->
        <directoryBrowse enabled="false" />

        <!-- Add MIME types for modern assets -->
        <staticContent>
            <remove fileExtension=".json" />
            <mimeMap fileExtension=".json" mimeType="application/json" />
            <remove fileExtension=".woff2" />
            <mimeMap fileExtension=".woff2" mimeType="font/woff2" />
            <remove fileExtension=".webp" />
            <mimeMap fileExtension=".webp" mimeType="image/webp" />
            <remove fileExtension=".svg" />
            <mimeMap fileExtension=".svg" mimeType="image/svg+xml" />
            <remove fileExtension=".webmanifest" />
            <mimeMap fileExtension=".webmanifest" mimeType="application/manifest+json" />
        </staticContent>

        <!-- Cache static assets -->
        <staticContent>
            <clientCache cacheControlMode="UseMaxAge" cacheControlMaxAge="30.00:00:00" />
        </staticContent>

        <!-- Compression -->
        <urlCompression doStaticCompression="true" doDynamicCompression="false" />

        <!-- Hide IIS version header -->
        <httpProtocol>
            <customHeaders>
                <remove name="X-Powered-By" />
            </customHeaders>
        </httpProtocol>
    </system.webServer>
</configuration>
```

> **⚠️ Gotcha:** IIS doesn't know about many modern MIME types out of the box (.woff2, .webmanifest, .webp, .json — sometimes). If you get `404.3` errors for static files, it's a **missing MIME type mapping**.

### 6.3 SPA Hosting (React / Vue / Angular)

```xml
<!-- web.config for SPA with client-side routing -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <!-- Redirect to HTTPS -->
                <rule name="HTTPS Redirect" stopProcessing="true">
                    <match url="(.*)" />
                    <conditions>
                        <add input="{HTTPS}" pattern="off" />
                    </conditions>
                    <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
                </rule>

                <!-- SPA fallback: serve index.html for all non-file routes -->
                <rule name="SPA Routes" stopProcessing="true">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll">
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                        <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="/index.html" />
                </rule>
            </rules>
        </rewrite>

        <staticContent>
            <remove fileExtension=".json" />
            <mimeMap fileExtension=".json" mimeType="application/json" />
        </staticContent>
    </system.webServer>
</configuration>
```

> **Requires:** URL Rewrite Module installed (see Section 10).

### 6.4 Custom Error Pages

```xml
<system.webServer>
    <httpErrors errorMode="Custom" existingResponse="Replace">
        <remove statusCode="404" subStatusCode="-1" />
        <error statusCode="404" path="/errors/404.html" responseMode="ExecuteURL" />
        <remove statusCode="500" subStatusCode="-1" />
        <error statusCode="500" path="/errors/500.html" responseMode="ExecuteURL" />
        <remove statusCode="403" subStatusCode="-1" />
        <error statusCode="403" path="/errors/403.html" responseMode="ExecuteURL" />
    </httpErrors>
</system.webServer>
```

**`errorMode` values:**

| Mode | Behavior |
|------|----------|
| `DetailedLocalOnly` | Detailed errors for localhost, custom for remote (default) |
| `Custom` | Custom error pages for everyone (production) |
| `Detailed` | Show detailed errors to everyone (dev only — **NEVER in prod**) |

> **⚠️ Gotcha:** `existingResponse="Replace"` is critical. Without it, if your app returns its own error response (e.g., ASP.NET 500 page), IIS's custom error page won't replace it. The `Replace` mode ensures IIS error pages always win.

---

## 7. Reverse Proxy (ARR + URL Rewrite)

### 7.1 What Is ARR?

ARR (Application Request Routing) is IIS's **reverse proxy and load balancing module**. Unlike Nginx/Apache where reverse proxy is built-in, ARR is an **add-on module**.

```
  ┌──────────────────────────────────────────────────────────────┐
  │               IIS Reverse Proxy Stack                        │
  │                                                              │
  │  URL Rewrite Module    ← Matching & routing rules            │
  │  +                                                           │
  │  ARR (Application      ← Proxying, load balancing,           │
  │   Request Routing)        health checks, caching             │
  │                                                              │
  │  Both are FREE downloads from Microsoft                      │
  │  (Web Platform Installer or direct download)                 │
  └──────────────────────────────────────────────────────────────┘
```

### 7.2 Installing ARR

```powershell
# Method 1: Web Platform Installer (WebPI)
# Download from: https://www.iis.net/downloads/microsoft/application-request-routing

# Method 2: WinGet (Windows Package Manager)
winget install Microsoft.IIS.UrlRewrite
winget install Microsoft.IIS.ApplicationRequestRouting

# Method 3: Direct MSI download + install
# URL Rewrite: https://www.iis.net/downloads/microsoft/url-rewrite
# ARR: https://www.iis.net/downloads/microsoft/application-request-routing

# After installation, enable proxy in ARR:
# IIS Manager → Server Node → Application Request Routing Cache → Server Proxy Settings
# ✅ Enable proxy
```

**Enable ARR proxy via appcmd:**

```cmd
:: Enable the ARR proxy
appcmd set config -section:system.webServer/proxy /enabled:"True" /commit:apphost
```

### 7.3 Basic Reverse Proxy

```xml
<!-- web.config at the site root -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <rule name="Reverse Proxy to Backend" stopProcessing="true">
                    <match url="(.*)" />
                    <action type="Rewrite" url="http://localhost:3000/{R:1}" />
                    <serverVariables>
                        <set name="HTTP_X_FORWARDED_HOST" value="{HTTP_HOST}" />
                        <set name="HTTP_X_REAL_IP" value="{REMOTE_ADDR}" />
                        <set name="HTTP_X_FORWARDED_PROTO" value="{CACHE_URL}" />
                    </serverVariables>
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
</configuration>
```

### 7.4 Reverse Proxy Architecture — IIS vs Nginx/Apache

```
  Nginx/Apache Reverse Proxy:
  ┌────────────────────────────┐
  │  proxy_pass / ProxyPass    │  ← One directive, one destination
  │  Built-in, always ready    │
  └────────────────────────────┘

  IIS Reverse Proxy:
  ┌────────────────────────────────────────────────────────────┐
  │  Step 1: Install URL Rewrite Module                        │
  │  Step 2: Install ARR Module                                │
  │  Step 3: Enable ARR proxy at server level                  │
  │  Step 4: Create URL Rewrite inbound rule                   │
  │  Step 5: (Optional) Create outbound rule for response      │
  │          header rewriting                                  │
  │                                                            │
  │  More complex setup, but more powerful pattern matching    │
  └────────────────────────────────────────────────────────────┘
```

### 7.5 Multi-Service Reverse Proxy

```xml
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <!-- API requests → Backend API -->
                <rule name="API Proxy" stopProcessing="true">
                    <match url="^api/(.*)" />
                    <action type="Rewrite" url="http://localhost:5000/api/{R:1}" />
                </rule>

                <!-- Auth requests → Auth service -->
                <rule name="Auth Proxy" stopProcessing="true">
                    <match url="^auth/(.*)" />
                    <action type="Rewrite" url="http://localhost:5001/auth/{R:1}" />
                </rule>

                <!-- WebSocket upgrade -->
                <rule name="WebSocket Proxy" stopProcessing="true">
                    <match url="^ws/(.*)" />
                    <conditions>
                        <add input="{HTTP_UPGRADE}" pattern="websocket" />
                    </conditions>
                    <action type="Rewrite" url="http://localhost:8080/ws/{R:1}" />
                </rule>

                <!-- Everything else → Frontend SPA -->
                <rule name="Frontend Fallback" stopProcessing="true">
                    <match url="(.*)" />
                    <conditions>
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="http://localhost:3000/{R:1}" />
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
</configuration>
```

### 7.6 Outbound Rewrite Rules (Response Header Rewriting)

The equivalent of Apache's `ProxyPassReverse` — rewrite response headers:

```xml
<rewrite>
    <outboundRules>
        <!-- Rewrite Location headers in redirects -->
        <rule name="Rewrite Location Header" preCondition="IsRedirection">
            <match serverVariable="RESPONSE_LOCATION" pattern="http://localhost:3000/(.*)" />
            <action type="Rewrite" value="https://{HTTP_HOST}/{R:1}" />
        </rule>

        <!-- Remove internal headers -->
        <rule name="Remove Internal Headers">
            <match serverVariable="RESPONSE_X_POWERED_BY" pattern=".*" />
            <action type="Rewrite" value="" />
        </rule>

        <preConditions>
            <preCondition name="IsRedirection">
                <add input="{RESPONSE_STATUS}" pattern="^30[12]" />
            </preCondition>
        </preConditions>
    </outboundRules>
</rewrite>
```

> **⚠️ Gotcha:** IIS outbound rules must explicitly allow setting server variables. Go to: URL Rewrite → View Server Variables → Add: `RESPONSE_LOCATION`, `HTTP_X_FORWARDED_HOST`, etc. Without this whitelist, rules silently fail.

### 7.7 Preserving Client IP Through ARR

```xml
<!-- applicationHost.config or server-level web.config -->
<system.webServer>
    <proxy preserveHostHeader="true" />
    <!-- This sends the original Host header to the backend -->
</system.webServer>
```

```powershell
# Enable preserve host header via PowerShell
Set-WebConfigurationProperty -Filter "system.webServer/proxy" -Name "preserveHostHeader" -Value "True" -PSPath "IIS:\"
```

ARR automatically sets `X-Forwarded-For` with the client IP. Backend apps should read this header.

---

## 8. HTTPS & SSL/TLS Certificate Installation

### 8.1 Certificate Sources

```
  ┌──────────────────────────────────────────────────────────────┐
  │           IIS Certificate Installation Sources               │
  │                                                              │
  │  1. Windows Certificate Store    ← IIS reads certs from here │
  │     └── Personal (My) store                                  │
  │                                                              │
  │  2. Let's Encrypt (win-acme)    ← Free, automated            │
  │  3. AD Certificate Services     ← Internal/corp certs        │
  │  4. Commercial CA (DigiCert)    ← Purchased certs            │
  │  5. Self-Signed                 ← Dev/testing only           │
  └──────────────────────────────────────────────────────────────┘
```

### 8.2 Let's Encrypt with win-acme (Free, Automated)

```powershell
# Download win-acme (ACME client for Windows/IIS)
# https://www.win-acme.com/

# Run the interactive wizard
.\wacs.exe

# Automated command:
.\wacs.exe --target iis --siteid 1 --installation iis --host example.com,www.example.com

# win-acme will:
# 1. Create HTTP-01 challenge in IIS site
# 2. Obtain cert from Let's Encrypt
# 3. Import cert into Windows Certificate Store
# 4. Bind cert to IIS site
# 5. Create scheduled task for auto-renewal
```

### 8.3 Manual Certificate Installation

**Step 1: Generate CSR (Certificate Signing Request)**

```
  IIS Manager → Server Node → Server Certificates → Create Certificate Request
  ┌──────────────────────────────────────────────────────────────┐
  │  Common Name:    example.com                                 │
  │  Organization:   My Company                                  │
  │  Org Unit:       IT                                          │
  │  City:           Ahmedabad                                   │
  │  State:          Gujarat                                     │
  │  Country:        IN                                          │
  │  Bit length:     2048 (minimum) or 4096                      │
  │  Output file:    C:\certs\example.com.csr                    │
  └──────────────────────────────────────────────────────────────┘
```

**Step 2: Submit CSR to CA, receive .cer/.crt file**

**Step 3: Complete Certificate Request in IIS**

```
  IIS Manager → Server Certificates → Complete Certificate Request
  ┌──────────────────────────────────────────────────────────────┐
  │  Certificate file:  C:\certs\example.com.cer                 │
  │  Friendly name:     example.com (2026)                       │
  │  Certificate store: Personal                                 │
  └──────────────────────────────────────────────────────────────┘
```

**Step 4: Bind to Site**

```powershell
# Bind SSL certificate to site
New-WebBinding -Name "MySite" -Protocol "https" -Port 443 -HostHeader "example.com" -SslFlags 1

# Assign certificate (by thumbprint)
$cert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*example.com*" }
$binding = Get-WebBinding -Name "MySite" -Protocol "https"
$binding.AddSslCertificate($cert.Thumbprint, "My")
```

### 8.4 Import PFX Certificate (Common Scenario)

```powershell
# Import PFX to Windows Certificate Store
$password = ConvertTo-SecureString -String "PfxPassword123" -AsPlainText -Force
Import-PfxCertificate -FilePath "C:\certs\example.com.pfx" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -Password $password

# Verify
Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*example.com*" }
```

### 8.5 SSL/TLS Hardening

```powershell
# Method 1: Use IIS Crypto tool (GUI — recommended for simplicity)
# Download: https://www.nartac.com/Products/IISCrypto

# Method 2: Registry changes (PowerShell)

# Disable TLS 1.0
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server" -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server" -Name "Enabled" -Value 0 -Type DWord

# Disable TLS 1.1
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server" -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server" -Name "Enabled" -Value 0 -Type DWord

# Enable TLS 1.2 (usually already enabled)
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server" -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server" -Name "Enabled" -Value 1 -Type DWord

# Enable TLS 1.3 (Windows Server 2022+)
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Server" -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Server" -Name "Enabled" -Value 1 -Type DWord

# REQUIRES REBOOT to take effect!
Restart-Computer
```

> **⚠️ Gotcha:** SSL/TLS protocol settings in Windows are **system-wide** (registry-based), not per-site like Nginx/Apache. Disabling TLS 1.0 affects ALL applications — IIS, RDP, LDAPS, SQL Server, WinRM. Test thoroughly before changing in production.

### 8.6 HTTP to HTTPS Redirect

```xml
<!-- web.config -->
<system.webServer>
    <rewrite>
        <rules>
            <rule name="HTTP to HTTPS" stopProcessing="true">
                <match url="(.*)" />
                <conditions>
                    <add input="{HTTPS}" pattern="off" />
                </conditions>
                <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
            </rule>
        </rules>
    </rewrite>
</system.webServer>
```

### 8.7 HSTS (HTTP Strict Transport Security)

```xml
<!-- IIS 10.0+ (Windows Server 2019+) native HSTS -->
<system.webServer>
    <hsts enabled="true" max-age="63072000" includeSubDomains="true" preload="true" redirectHttpToHttps="true" />
</system.webServer>

<!-- Older IIS: Use custom headers -->
<system.webServer>
    <httpProtocol>
        <customHeaders>
            <add name="Strict-Transport-Security" value="max-age=63072000; includeSubDomains; preload" />
        </customHeaders>
    </httpProtocol>
</system.webServer>
```

> **⚠️ Gotcha:** On IIS versions before 10.0, the native `<hsts>` element doesn't exist. Use custom headers instead. But custom headers are sent on **both** HTTP and HTTPS responses — HSTS should only be on HTTPS. Use URL Rewrite outbound rules to conditionally add it only for HTTPS.

---

## 9. Server-Side Scripting — ASP.NET, PHP, Node.js

### 9.1 Architecture — IIS Hosting Models

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                IIS Application Hosting Models                    │
  │                                                                  │
  │  ┌─────────────────────┐  ┌──────────────────────────────────┐   │
  │  │ In-Process Hosting  │  │  Out-of-Process (Reverse Proxy)  │   │
  │  │                     │  │                                  │   │
  │  │  ┌───────────────┐  │  │  ┌──────┐     ┌──────────────┐   │   │
  │  │  │  w3wp.exe     │  │  │  │ IIS  │────►│ Kestrel /    │   │   │
  │  │  │  ┌─────────┐  │  │  │  │(ARR) │     │ Node.js /    │   │   │
  │  │  │  │ ASP.NET │  │  │  │  └──────┘     │ PHP-CGI /    │   │   │
  │  │  │  │ Core    │  │  │  │               │ Python       │   │   │
  │  │  │  │ module  │  │  │  │               └──────────────┘   │   │
  │  │  │  └─────────┘  │  │  │                                  │   │
  │  │  └───────────────┘  │  │  Separate process                │   │
  │  │                     │  │  IIS = reverse proxy only        │   │
  │  │  App runs INSIDE    │  │                                  │   │
  │  │  IIS worker process │  │  Better isolation                │   │
  │  │  Best performance   │  │  Can run non-.NET languages      │   │
  │  └─────────────────────┘  └──────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────┘
```

### 9.2 ASP.NET Core (Modern .NET — Recommended)

```powershell
# Install ASP.NET Core Hosting Bundle
# Download from: https://dotnet.microsoft.com/download/dotnet
# This installs the ASP.NET Core Module (ANCM) for IIS

# Verify installation
Get-WebGlobalModule | Where-Object { $_.Name -like "*AspNet*" }
```

**In-Process hosting (best performance):**

```xml
<!-- web.config generated by dotnet publish -->
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <location path="." inheritInChildApplications="false">
        <system.webServer>
            <handlers>
                <add name="aspNetCore" path="*" verb="*"
                     modules="AspNetCoreModuleV2" resourceType="Unspecified" />
            </handlers>
            <aspNetCore processPath="dotnet" arguments=".\MyApp.dll"
                        stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout"
                        hostingModel="inprocess" />
            <!--                      ^^^^^^^^^ key setting -->
        </system.webServer>
    </location>
</configuration>
```

**Out-of-Process hosting (more isolation):**

```xml
<aspNetCore processPath="dotnet" arguments=".\MyApp.dll"
            stdoutLogEnabled="true" stdoutLogFile=".\logs\stdout"
            hostingModel="outofprocess">
    <environmentVariables>
        <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
    </environmentVariables>
</aspNetCore>
```

```
  In-Process vs Out-of-Process:
  ┌──────────────────────────────────────────────────────────────┐
  │  In-Process:                                                 │
  │  • App runs INSIDE w3wp.exe                                  │
  │  • No inter-process communication overhead                   │
  │  • ~4x faster than out-of-process                            │
  │  • Limited to one app per AppPool                            │
  │  • If app crashes, worker process crashes                    │
  │                                                              │
  │  Out-of-Process:                                             │
  │  • IIS proxies to Kestrel (separate dotnet.exe process)      │
  │  • More overhead (HTTP proxy between IIS and Kestrel)        │
  │  • Multiple apps can share an AppPool                        │
  │  • Better isolation (app crash ≠ worker crash)               │
  │  • Required for sidecar patterns                             │
  └──────────────────────────────────────────────────────────────┘
```

### 9.3 ASP.NET Framework (Legacy — 4.x)

```powershell
# Ensure .NET Framework 4.8 features are installed
Install-WindowsFeature Net-Framework-45-ASPNET
Install-WindowsFeature Web-Asp-Net45
```

```powershell
# Create AppPool with .NET 4.0 CLR
New-WebAppPool -Name "LegacyPool"
Set-ItemProperty "IIS:\AppPools\LegacyPool" -Name "managedRuntimeVersion" -Value "v4.0"
Set-ItemProperty "IIS:\AppPools\LegacyPool" -Name "managedPipelineMode" -Value "Integrated"
```

### 9.4 PHP on IIS

```
  IIS PHP Hosting Options:
  ┌──────────────────────────────────────────────────────────────┐
  │  1. FastCGI (recommended)                                    │
  │     → php-cgi.exe runs as persistent FastCGI process         │
  │     → Much faster than plain CGI                             │
  │     → IIS manages the process pool                           │
  │                                                              │
  │  2. WinCache (optional PHP accelerator)                      │
  │     → Opcode cache (like OPcache on Linux)                   │
  │     → File cache, user cache                                 │
  │                                                              │
  │  Note: PHP-FPM is Linux-only.                                │
  │  On Windows, IIS manages PHP processes via FastCGI handler.  │
  └──────────────────────────────────────────────────────────────┘
```

```powershell
# Install PHP via Web Platform Installer or manually:
# 1. Download PHP for Windows: https://windows.php.net/download/
# 2. Extract to C:\PHP
# 3. Copy php.ini-production to php.ini
# 4. Enable required extensions in php.ini

# Register PHP with IIS FastCGI
appcmd set config /section:system.webServer/fastCGI /+[fullPath='C:\PHP\php-cgi.exe']

# Map .php extension to FastCGI handler
appcmd set config /section:system.webServer/handlers /+[name='PHP-FastCGI',path='*.php',verb='*',modules='FastCgiModule',scriptProcessor='C:\PHP\php-cgi.exe',resourceType='File']
```

```xml
<!-- web.config for PHP site -->
<configuration>
    <system.webServer>
        <defaultDocument>
            <files>
                <clear />
                <add value="index.php" />
                <add value="index.html" />
            </files>
        </defaultDocument>

        <handlers>
            <add name="PHP-FastCGI" path="*.php" verb="*"
                 modules="FastCgiModule"
                 scriptProcessor="C:\PHP\php-cgi.exe"
                 resourceType="File" />
        </handlers>

        <!-- Rewrite for pretty URLs (WordPress, Laravel) -->
        <rewrite>
            <rules>
                <rule name="WordPress" stopProcessing="true">
                    <match url=".*" />
                    <conditions>
                        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                        <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="index.php" />
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
</configuration>
```

### 9.5 Node.js on IIS (iisnode or Reverse Proxy)

**Method 1: iisnode (in-process — legacy, less common now)**

```xml
<!-- web.config with iisnode -->
<configuration>
    <system.webServer>
        <handlers>
            <add name="iisnode" path="server.js" verb="*" modules="iisnode" />
        </handlers>
        <rewrite>
            <rules>
                <rule name="NodeJS">
                    <match url="/*" />
                    <action type="Rewrite" url="server.js" />
                </rule>
            </rules>
        </rewrite>
        <iisnode nodeProcessCommandLine="C:\Program Files\nodejs\node.exe"
                 watchedFiles="web.config;*.js"
                 loggingEnabled="true"
                 logDirectory="iisnode" />
    </system.webServer>
</configuration>
```

**Method 2: Reverse Proxy via ARR (recommended)**

```xml
<!-- Reverse proxy to Node.js process -->
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <rule name="Node Reverse Proxy" stopProcessing="true">
                    <match url="(.*)" />
                    <action type="Rewrite" url="http://localhost:3000/{R:1}" />
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
</configuration>
```

> **Modern recommendation:** Use ARR reverse proxy to Node.js running as a Windows Service (via NSSM or `node-windows`). This gives you better process management, logging, and restart capability than iisnode.

---

## 10. URL Rewrite Module

### 10.1 Rule Anatomy

```xml
<rule name="Rule Name" stopProcessing="true">
    <!-- 1. MATCH: What URL pattern to match -->
    <match url="^old-page$" />

    <!-- 2. CONDITIONS: Additional tests (optional) -->
    <conditions logicalGrouping="MatchAll">
        <add input="{HTTP_HOST}" pattern="^example\.com$" />
        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
    </conditions>

    <!-- 3. ACTION: What to do -->
    <action type="Redirect" url="https://example.com/new-page" redirectType="Permanent" />
</rule>
```

### 10.2 Action Types

| Action Type | HTTP Behavior | Use Case |
|-------------|--------------|----------|
| `Rewrite` | Internal rewrite (client doesn't see) | Reverse proxy, SPA fallback |
| `Redirect` | 301/302 redirect (client sees new URL) | URL migration, HTTP→HTTPS |
| `CustomResponse` | Return specific status code + body | Block requests, maintenance mode |
| `AbortRequest` | Drop connection silently | Block bots/attacks |
| `None` | No action (use with `stopProcessing="false"`) | Set server variables only |

### 10.3 Common URL Rewrite Patterns

```xml
<rewrite>
    <rules>
        <!-- Force HTTPS -->
        <rule name="HTTPS Redirect" stopProcessing="true">
            <match url="(.*)" />
            <conditions>
                <add input="{HTTPS}" pattern="off" />
            </conditions>
            <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
        </rule>

        <!-- Force WWW -->
        <rule name="Add WWW" stopProcessing="true">
            <match url="(.*)" />
            <conditions>
                <add input="{HTTP_HOST}" pattern="^example\.com$" />
            </conditions>
            <action type="Redirect" url="https://www.example.com/{R:1}" redirectType="Permanent" />
        </rule>

        <!-- Remove trailing slash -->
        <rule name="Remove Trailing Slash" stopProcessing="true">
            <match url="(.*)/$" />
            <conditions>
                <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
            </conditions>
            <action type="Redirect" url="{R:1}" redirectType="Permanent" />
        </rule>

        <!-- Block specific User-Agents -->
        <rule name="Block Bad Bots" stopProcessing="true">
            <match url=".*" />
            <conditions>
                <add input="{HTTP_USER_AGENT}" pattern="(curl|wget|python|scrapy)" />
            </conditions>
            <action type="AbortRequest" />
        </rule>

        <!-- Maintenance mode -->
        <rule name="Maintenance Mode" stopProcessing="true">
            <match url=".*" />
            <conditions>
                <add input="{REMOTE_ADDR}" pattern="^10\.0\.0\.1$" negate="true" />
            </conditions>
            <action type="CustomResponse" statusCode="503"
                    statusReason="Maintenance" statusDescription="Site is under maintenance" />
        </rule>
    </rules>
</rewrite>
```

### 10.4 Regex Capture Groups & Back-References

```
  URL match pattern:  ^blog/(\d{4})/(\d{2})/(.+)$
                           {R:1}    {R:2}   {R:3}

  Request: /blog/2026/03/my-post
  {R:0} = blog/2026/03/my-post  (full match)
  {R:1} = 2026                   (first capture group)
  {R:2} = 03                     (second capture group)
  {R:3} = my-post                (third capture group)

  Condition input:    {HTTP_HOST}  pattern: ^(www\.)?(.+)$
                                            {C:1}   {C:2}

  {C:0} = www.example.com
  {C:1} = www.
  {C:2} = example.com
```

---

## 11. Load Balancing (ARR Server Farms)

### 11.1 Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   IIS + ARR Load Balancing                  │
  │                                                             │
  │                    ┌─────────────┐                          │
  │                    │   Client    │                          │
  │                    └──────┬──────┘                          │
  │                           │                                 │
  │                    ┌──────▼──────┐                          │
  │                    │  IIS + ARR  │                          │
  │                    │ (Server     │                          │
  │                    │  Farm)      │                          │
  │                    └──┬───┬───┬──┘                          │
  │                       │   │   │                             │
  │              ┌────────┘   │   └────────┐                    │
  │              ▼            ▼            ▼                    │
  │         ┌─────────┐ ┌─────────┐ ┌─────────┐                 │
  │         │Server 1 │ │Server 2 │ │Server 3 │                 │
  │         │ (IIS)   │ │ (IIS)   │ │ (IIS)   │                 │
  │         │Weight:50│ │Weight:50│ │Weight:50│                 │
  │         └─────────┘ └─────────┘ └─────────┘                 │
  │                                                             │
  │    Health Checks: ARR pings /health on each server          │
  │    Sticky Sessions: Cookie or URL-based affinity            │
  │    Algorithms: Round Robin, Weighted, Least Requests        │
  └─────────────────────────────────────────────────────────────┘
```

### 11.2 Creating a Server Farm

```
  IIS Manager → Server Farms → Create Server Farm
  ┌──────────────────────────────────────────────────────────────┐
  │  Farm Name: MyBackendFarm                                    │
  │                                                              │
  │  Servers:                                                    │
  │  ├── 10.0.0.10 (port 80, weight 100)                         │
  │  ├── 10.0.0.11 (port 80, weight 100)                         │
  │  └── 10.0.0.12 (port 80, weight 50)  ← lower weight          │
  │                                                              │
  │  "Create URL Rewrite rule for this farm?" → YES              │
  └──────────────────────────────────────────────────────────────┘
```

**PowerShell (using appcmd):**

```cmd
:: Create server farm
appcmd set config -section:webFarms /+[name='MyBackendFarm'] /commit:apphost

:: Add servers to farm
appcmd set config -section:webFarms /+[name='MyBackendFarm'].[address='10.0.0.10'] /commit:apphost
appcmd set config -section:webFarms /+[name='MyBackendFarm'].[address='10.0.0.11'] /commit:apphost
appcmd set config -section:webFarms /+[name='MyBackendFarm'].[address='10.0.0.12'] /commit:apphost
```

### 11.3 Load Balancing Algorithms

| Algorithm | Description |
|-----------|-------------|
| **Weighted Round Robin** | Distributes by weight (default) |
| **Weighted Total Traffic** | Routes to server with least total bytes transferred |
| **Least Current Request** | Routes to server with fewest active requests |
| **Least Response Time** | Routes to server with fastest response time |

### 11.4 Health Checks

```
  ARR Health Checks:
  ┌──────────────────────────────────────────────────────────────┐
  │  URL Test: GET /health HTTP/1.1                              │
  │  Interval: 30 seconds                                        │
  │  Timeout: 10 seconds                                         │
  │  Healthy Status: 200                                         │
  │  Retry: 3 failures → mark unhealthy                          │
  │  Recovery: 3 successes → mark healthy again                  │
  └──────────────────────────────────────────────────────────────┘
```

### 11.5 Client Affinity (Sticky Sessions)

```
  ARR supports two affinity modes:
  ┌──────────────────────────────────────────────────────────────┐
  │  1. Client Affinity Cookie (ARRAffinity)                     │
  │     → ARR sets a cookie with the backend server ID           │
  │     → Subsequent requests route to the same server           │
  │     → Cookie name: ARRAffinity (default)                     │
  │                                                              │
  │  2. Host Name Affinity                                       │
  │     → Routes based on the hostname in the request            │
  │     → Useful for multi-tenant SaaS                           │
  └──────────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** The `ARRAffinity` cookie is not encrypted by default. Attackers can decode it to discover backend server names/IPs. Use `ARRAffinitySameSite` and `ARRAffinitySecure` settings, or better yet, externalize session state (Redis, SQL) and disable affinity.

---

## 12. Caching & Performance Tuning

### 12.1 Output Caching

```xml
<!-- web.config -->
<system.webServer>
    <caching>
        <profiles>
            <!-- Cache static HTML for 1 hour -->
            <add extension=".html" policy="CacheForTimePeriod"
                 kernelCachePolicy="CacheForTimePeriod"
                 duration="01:00:00" />

            <!-- Cache API responses based on query string -->
            <add extension=".aspx" policy="CacheForTimePeriod"
                 kernelCachePolicy="DontCache"
                 duration="00:05:00"
                 varyByQueryString="id,page" />
        </profiles>
    </caching>
</system.webServer>
```

### 12.2 Kernel-Mode Caching (HTTP.sys)

```
  ┌───────────────────────────────────────────────────────────────┐
  │  Kernel-Mode Cache (HTTP.sys)                                 │
  │                                                               │
  │  Request → HTTP.sys → Cache HIT? ──Yes──► Return response     │
  │                         │                 (w3wp never wakes!) │
  │                         No                                    │
  │                         │                                     │
  │                         ▼                                     │
  │                    w3wp.exe processes request                 │
  │                    Response cached in HTTP.sys for next time  │
  │                                                               │
  │  This is FASTER than Nginx/Apache caching because:            │
  │  - No user-mode context switch                                │
  │  - No process wakeup                                          │
  │  - Direct kernel → network stack → client                     │
  └───────────────────────────────────────────────────────────────┘
```

> **DevOps Relevance:** For static content and cacheable API responses, IIS kernel-mode cache can serve 100K+ req/sec on modest hardware. This is why benchmarks sometimes show IIS beating Nginx for static files.

### 12.3 Compression

```xml
<system.webServer>
    <!-- Enable compression -->
    <urlCompression doStaticCompression="true" doDynamicCompression="true" />

    <httpCompression>
        <dynamicTypes>
            <add mimeType="application/json" enabled="true" />
            <add mimeType="application/xml" enabled="true" />
            <add mimeType="text/*" enabled="true" />
        </dynamicTypes>
        <staticTypes>
            <add mimeType="text/*" enabled="true" />
            <add mimeType="application/javascript" enabled="true" />
            <add mimeType="image/svg+xml" enabled="true" />
        </staticTypes>
    </httpCompression>
</system.webServer>
```

```powershell
# Ensure compression features are installed
Install-WindowsFeature Web-Stat-Compression
Install-WindowsFeature Web-Dyn-Compression
```

### 12.4 Connection Limits & Queue Tuning

```powershell
# AppPool queue length (default 1000)
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "queueLength" -Value 5000

# Max worker processes per AppPool (Web Garden)
# WARNING: Web Garden ≠ performance gain for most apps!
# Only use for CPU-bound stateless apps
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "processModel.maxProcesses" -Value 1  # Keep at 1 unless you know what you're doing
```

> **⚠️ Gotcha: Web Gardens.** Setting `maxProcesses > 1` creates multiple w3wp.exe instances for one AppPool. This sounds like Nginx workers but is NOT the same. Each w3wp.exe has its own in-memory state (session, cache, static variables). Web Gardens break in-memory sessions, SignalR, and most real-world apps. Only use for truly stateless, CPU-bound scenarios.

---

## 13. Security Hardening

### 13.1 Remove Server Headers

```xml
<!-- web.config -->
<system.webServer>
    <httpProtocol>
        <customHeaders>
            <remove name="X-Powered-By" />
            <remove name="X-AspNet-Version" />
            <remove name="X-AspNetMvc-Version" />
            <remove name="Server" />
        </customHeaders>
    </httpProtocol>

    <!-- Remove Server header (IIS 10+) -->
    <security>
        <requestFiltering removeServerHeader="true" />
    </security>
</system.webServer>

<!-- Remove X-AspNet-Version globally -->
<!-- In machine-level web.config or applicationHost.config -->
<system.web>
    <httpRuntime enableVersionHeader="false" />
</system.web>
```

> **⚠️ Gotcha:** The `Server: Microsoft-IIS/10.0` header is notoriously hard to remove completely. `removeServerHeader="true"` works in IIS 10+ but requires the Request Filtering module. For older IIS, use URL Rewrite outbound rules or a third-party module (URLScan).

### 13.2 Security Headers

```xml
<system.webServer>
    <httpProtocol>
        <customHeaders>
            <!-- Remove fingerprinting headers -->
            <remove name="X-Powered-By" />

            <!-- Add security headers -->
            <add name="X-Frame-Options" value="SAMEORIGIN" />
            <add name="X-Content-Type-Options" value="nosniff" />
            <add name="X-XSS-Protection" value="1; mode=block" />
            <add name="Referrer-Policy" value="strict-origin-when-cross-origin" />
            <add name="Permissions-Policy" value="camera=(), microphone=(), geolocation=()" />
            <add name="Content-Security-Policy" value="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" />
        </customHeaders>
    </httpProtocol>
</system.webServer>
```

### 13.3 Request Filtering (Built-in WAF)

```xml
<system.webServer>
    <security>
        <requestFiltering>
            <!-- Max request size (default 30MB for IIS, 28.6MB for ASP.NET) -->
            <requestLimits maxAllowedContentLength="52428800"
                          maxUrl="4096"
                          maxQueryString="2048" />

            <!-- Block dangerous file extensions -->
            <fileExtensions allowUnlisted="true">
                <add fileExtension=".config" allowed="false" />
                <add fileExtension=".cs" allowed="false" />
                <add fileExtension=".vb" allowed="false" />
                <add fileExtension=".mdf" allowed="false" />
                <add fileExtension=".ldf" allowed="false" />
            </fileExtensions>

            <!-- Block dangerous URL sequences -->
            <denyUrlSequences>
                <add sequence=".." />
                <add sequence="//" />
                <add sequence="\\" />
                <add sequence=":" />
            </denyUrlSequences>

            <!-- Block specific HTTP verbs -->
            <verbs allowUnlisted="true">
                <add verb="TRACE" allowed="false" />
                <add verb="OPTIONS" allowed="false" />
            </verbs>

            <!-- Hidden segments -->
            <hiddenSegments>
                <add segment="web.config" />
                <add segment="bin" />
                <add segment="App_Code" />
                <add segment="App_Data" />
                <add segment=".git" />
                <add segment=".env" />
            </hiddenSegments>
        </requestFiltering>
    </security>
</system.webServer>
```

### 13.4 Authentication Methods

```
  ┌──────────────────────────────────────────────────────────────┐
  │            IIS Authentication Methods                        │
  │                                                              │
  │  Anonymous             → No auth (public sites)              │
  │  Basic                 → Username/password (Base64, needs    │
  │                          HTTPS!)                             │
  │  Windows (Negotiate)   → Kerberos/NTLM (intranet apps)       │
  │  Digest                → Hashed credentials (legacy)         │
  │  Client Certificate    → Mutual TLS (mTLS)                   │
  │  Forms                 → Custom login page (ASP.NET)         │
  │  OpenID Connect/OAuth  → Via middleware (ASP.NET Core)       │
  └──────────────────────────────────────────────────────────────┘
```

```powershell
# Enable Windows Authentication for an intranet site
Set-WebConfigurationProperty -Filter "/system.webServer/security/authentication/windowsAuthentication" `
    -Name "enabled" -Value "True" -PSPath "IIS:\Sites\IntranetSite"

# Disable Anonymous Authentication
Set-WebConfigurationProperty -Filter "/system.webServer/security/authentication/anonymousAuthentication" `
    -Name "enabled" -Value "False" -PSPath "IIS:\Sites\IntranetSite"
```

> **DevOps Relevance:** IIS's native Windows Authentication with Kerberos is a killer feature for intranet apps — seamless single sign-on (SSO) for domain-joined machines. This is impossible to replicate natively in Nginx/Apache without complex LDAP/Kerberos proxy setups.

### 13.5 IP Address Restrictions

```xml
<system.webServer>
    <security>
        <ipSecurity allowUnlisted="true">
            <!-- Block specific IPs -->
            <add ipAddress="192.168.1.100" allowed="false" />

            <!-- Block a subnet -->
            <add ipAddress="10.0.0.0" subnetMask="255.0.0.0" allowed="false" />
        </ipSecurity>
    </security>
</system.webServer>
```

```powershell
# Install IP Security feature
Install-WindowsFeature Web-IP-Security
```

---

## 14. Logging, Debugging & Monitoring

### 14.1 IIS Logging

```
  IIS Log Location: C:\inetpub\logs\LogFiles\W3SVC{SiteID}\
  Default Format: W3C Extended Log Format

  Example log line:
  2026-03-03 10:15:32 10.0.0.1 GET /api/users - 443 - 192.168.1.50
  Mozilla/5.0... https://example.com/dashboard 200 0 0 125

  Fields:
  date time s-ip cs-method cs-uri-stem cs-uri-query s-port cs-username
  c-ip cs(User-Agent) cs(Referer) sc-status sc-substatus sc-win32-status
  time-taken
```

### 14.2 Configuring Logging

```powershell
# Set logging fields
Set-WebConfigurationProperty -Filter "system.applicationHost/sites/siteDefaults/logFile" `
    -Name "logExtFileFlags" `
    -Value "Date,Time,ClientIP,UserName,SiteName,ServerIP,Method,UriStem,UriQuery,HttpStatus,Win32Status,BytesSent,BytesRecv,TimeTaken,ServerPort,UserAgent,Referer,HttpSubStatus"

# Per-site log file path
Set-ItemProperty "IIS:\Sites\MySite" -Name "logFile.directory" -Value "D:\Logs\IIS"

# JSON logging (IIS 10+ with Enhanced Logging module)
# Or use log parsing tools: Log Parser, GoAccess, ELK
```

### 14.3 Failed Request Tracing (FREB — The Secret Weapon)

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Failed Request Tracing (FREB)                               │
  │                                                              │
  │  What:  Captures detailed trace of a request from start      │
  │         to finish — every module, handler, auth step,        │
  │         rewrite rule, and error.                             │
  │                                                              │
  │  Output: XML file that opens in a browser as an interactive  │
  │          trace timeline.                                     │
  │                                                              │
  │  When to use:                                                │
  │  • 500 errors with no useful info in logs                    │
  │  • Debugging URL Rewrite rules                               │
  │  • Authentication failures                                   │
  │  • Slow requests (set time threshold)                        │
  │  • Understanding the IIS module pipeline                     │
  └──────────────────────────────────────────────────────────────┘
```

```powershell
# Install Failed Request Tracing
Install-WindowsFeature Web-Http-Tracing

# Enable for a site
Set-ItemProperty "IIS:\Sites\MySite" -Name "traceFailedRequestsLogging.enabled" -Value "True"
Set-ItemProperty "IIS:\Sites\MySite" -Name "traceFailedRequestsLogging.directory" -Value "D:\Logs\FREB"
Set-ItemProperty "IIS:\Sites\MySite" -Name "traceFailedRequestsLogging.maxLogFiles" -Value 50
```

```xml
<!-- web.config: Trace 500 errors and slow requests -->
<system.webServer>
    <tracing>
        <traceFailedRequests>
            <add path="*">
                <traceAreas>
                    <add provider="WWW Server" areas="Authentication,Security,Filter,StaticFile,CGI,Compression,Cache,RequestNotifications,Module,Rewrite" verbosity="Verbose" />
                    <add provider="ASP" areas="" verbosity="Verbose" />
                    <add provider="ASPNET" areas="Infrastructure,Module,Page,AppServices" verbosity="Verbose" />
                </traceAreas>
                <failureDefinitions statusCodes="400-599" timeTaken="00:00:30" />
            </add>
        </traceFailedRequests>
    </tracing>
</system.webServer>
```

> **FREB is the single most powerful debugging tool in IIS.** It's like having `strace` + `tcpdump` + `mod_rewrite trace` all in one. If you're debugging any IIS issue, enable FREB first.

### 14.4 Performance Monitor Counters

```powershell
# Key IIS performance counters
Get-Counter -ListSet "Web Service" | Select-Object -ExpandProperty Counter | Select-Object -First 20

# Monitor in real-time
Get-Counter '\Web Service(_Total)\Current Connections',
            '\Web Service(_Total)\Total Method Requests/sec',
            '\ASP.NET\Requests Current',
            '\ASP.NET\Request Execution Time',
            '\Process(w3wp*)\% Processor Time',
            '\Process(w3wp*)\Working Set' -Continuous -SampleInterval 5
```

| Counter | What It Tells You |
|---------|-------------------|
| `Current Connections` | Active connections now |
| `Total Method Requests/sec` | Request throughput |
| `Requests Current` | ASP.NET requests in pipeline |
| `Request Execution Time` | Time to process request (ms) |
| `w3wp % Processor Time` | CPU usage of worker process |
| `w3wp Working Set` | Memory usage of worker process |

---

## 15. Production Gotchas & Tricky Q&A

### Q1: My IIS returns 500.19 — what broke?

**Answer:** 500.19 means **configuration error** (malformed web.config or locked section).

```
  ┌──────────────────────────────────────────────────────────────┐
  │  500.19 Checklist:                                           │
  │                                                              │
  │  1. Malformed web.config XML                                 │
  │     → Check for typos, unclosed tags, BOM characters         │
  │     → Open in browser to verify XML is well-formed           │
  │                                                              │
  │  2. Configuration section locked at a parent level           │
  │     → Error message says "This configuration section         │
  │       cannot be used at this path"                           │
  │     → Fix: Unlock section in applicationHost.config          │
  │       or use appcmd:                                         │
  │       appcmd unlock config -section:system.webServer/handlers│
  │                                                              │
  │  3. Missing module referenced in web.config                  │
  │     → web.config references a module not installed           │
  │     → Install the module or remove the reference             │
  │                                                              │
  │  4. Duplicate config entries                                 │
  │     → Use <remove> before <add> for mime types, handlers     │
  │     → Or use <clear /> to start fresh                        │
  └──────────────────────────────────────────────────────────────┘
```

> **The duplicate entry gotcha is the #1 cause.** Example: You add a MIME type in `web.config` that already exists in `applicationHost.config`. Solution: `<remove fileExtension=".json" />` before `<mimeMap fileExtension=".json" ... />`.

---

### Q2: 500.19 "Cannot read configuration file" — permissions?

```
  AppPool Identity needs READ access to:
  ┌─────────────────────────────────────────────────────────────────────┐
  │  1. The site's physical path (D:\Sites\MySite\)                     │
  │  2. All parent directories up to the drive root                     │
  │  3. web.config files in those directories                           │
  │                                                                     │
  │  Fix:                                                               │
  │  icacls "D:\Sites\MySite" /grant "IIS AppPool\MySitePool":R         │
  │  icacls "D:\Sites\MySite" /grant "IIS AppPool\MySitePool":(OI)(CI)R │
  │                                                                     │
  │  ⚠️ Common mistake: granting IUSR access but not the                │
  │  AppPool identity. In IIS 7.5+, the AppPool identity is             │
  │  what reads config, not IUSR.                                       │
  └─────────────────────────────────────────────────────────────────────┘
```

---

### Q3: Why does my site work on localhost but return 503 remotely?

```
  503 = Service Unavailable = AppPool is STOPPED
  
  Check:
  1. Is the AppPool running?
     → appcmd list apppools | findstr "MySitePool"
     → If "Stopped" → check Event Viewer for crash reason
  
  2. Rapid-Fail Protection tripped?
     → AppPool crashed 5+ times in 5 minutes
     → IIS disabled it to prevent crash loops
     → Fix the underlying crash, then restart AppPool
  
  3. Identity password expired?
     → If using a domain service account, password rotation
       breaks the AppPool
     → Fix: update password or switch to gMSA
  
  4. Port conflict?
     → Another site is using the same binding
     → Run: netsh http show urlacl
```

---

### Q4: What's the difference between Application, Virtual Directory, and Site?

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Site: "MySite" → *:80:mysite.com                            │
  │  │                                                           │
  │  ├── Application: /              (root app)                  │
  │  │   │  AppPool: MySitePool                                  │
  │  │   │  Physical: D:\Sites\MySite                            │
  │  │   │                                                       │
  │  │   └── Virtual Dir: /shared                                │
  │  │       Physical: E:\SharedFiles                            │
  │  │       (same AppPool as parent app — just a path mapping)  │
  │  │                                                           │
  │  └── Application: /api                                       │
  │      AppPool: ApiPool  ← SEPARATE process!                   │
  │      Physical: D:\Apps\API                                   │
  │      (independent AppPool = isolated memory, identity, CLR)  │
  └──────────────────────────────────────────────────────────────┘

  Key difference:
  • Virtual Directory = just a path alias (same AppPool, same config scope)
  • Application = isolated config scope, can have its own AppPool
  • Site = top-level container with bindings
```

---

### Q5: AppPool keeps crashing and restarting — how to diagnose?

```powershell
# 1. Check Event Viewer
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='WAS','W3SVC'; Level=1,2,3} -MaxEvents 20

# 2. Enable crash dumps for w3wp.exe
# Use ProcDump from Sysinternals:
procdump -ma -e -w w3wp.exe D:\CrashDumps\

# 3. Check FREB traces (enable for 500 errors)

# 4. Common causes:
#    → Stack overflow in app code
#    → Unhandled exception in app startup
#    → Missing .NET Core Hosting Bundle
#    → Wrong .NET CLR version in AppPool
#    → 32-bit app in 64-bit AppPool (or vice versa)

# 5. Check if 32-bit mode is needed
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "enable32BitAppOnWin64" -Value $true
```

---

### Q6: ARR reverse proxy returns 502.3 — backend timeout?

```
  502.3 = Bad Gateway — ARR can't reach the backend or backend timed out

  ┌──────────────────────────────────────────────────────────────┐
  │  Checklist:                                                  │
  │                                                              │
  │  1. Is the backend running?                                  │
  │     → Test: curl http://localhost:3000/health                │
  │                                                              │
  │  2. Firewall blocking?                                       │
  │     → Windows Firewall, NSG, or network ACLs                 │
  │                                                              │
  │  3. ARR proxy timeout too low?                               │
  │     → IIS Manager → ARR → Proxy → Time-out (default: 30s)    │
  │     → Increase for slow backends                             │
  │                                                              │
  │  4. Backend returning invalid response?                      │
  │     → Check backend logs                                     │
  │     → Test direct backend connection                         │
  │                                                              │
  │  5. ARR not enabled?                                         │
  │     → Server Node → ARR → Enable proxy = TRUE                │
  └──────────────────────────────────────────────────────────────┘
```

```powershell
# Increase ARR proxy timeout
appcmd set config -section:system.webServer/proxy /timeout:"00:03:00" /commit:apphost
```

---

### Q7: How do I do zero-downtime deployments on IIS?

```
  Method 1: Overlapped Recycling (built-in)
  ┌──────────────────────────────────────────────────────────────┐
  │  When an AppPool recycles:                                   │
  │  1. New w3wp.exe starts with new code                        │
  │  2. Old w3wp.exe continues serving existing requests         │
  │  3. New requests go to new w3wp.exe                          │
  │  4. Old w3wp.exe shuts down after requests complete          │
  │  = Zero dropped connections ✅                               │
  │                                                              │
  │  Trigger: appcmd recycle apppool "MySitePool"                │
  └──────────────────────────────────────────────────────────────┘

  Method 2: App Initialization (pre-warm)
  ┌──────────────────────────────────────────────────────────────┐
  │  Problem: After recycle, first request is SLOW (cold start)  │
  │  Solution: App Initialization module                         │
  │                                                              │
  │  1. Set AppPool startMode = AlwaysRunning                    │
  │  2. Set site preloadEnabled = true                           │
  │  3. Configure initialization page                            │
  │                                                              │
  │  IIS pre-warms the app BEFORE sending real traffic ✅        │
  └──────────────────────────────────────────────────────────────┘
```

```xml
<!-- web.config — App Initialization -->
<system.webServer>
    <applicationInitialization remapManagedRequestsTo="loading.html" skipManagedModules="false">
        <add initializationPage="/" />
        <add initializationPage="/api/health" />
    </applicationInitialization>
</system.webServer>
```

```powershell
# Enable AlwaysRunning + Preload
Set-ItemProperty "IIS:\AppPools\MySitePool" -Name "startMode" -Value "AlwaysRunning"
Set-ItemProperty "IIS:\Sites\MySite" -Name "applicationDefaults.preloadEnabled" -Value "True"

# Install Application Initialization feature
Install-WindowsFeature Web-AppInit
```

---

### Q8: `maxAllowedContentLength` vs `maxRequestLength` — which limits uploads?

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Two DIFFERENT settings control upload limits:               │
  │                                                              │
  │  1. maxAllowedContentLength (IIS level — Request Filtering)  │
  │     Default: 30,000,000 bytes (~28.6 MB)                     │
  │     Unit: BYTES                                              │
  │     Error: 404.13                                            │
  │                                                              │
  │  2. maxRequestLength (ASP.NET level)                         │
  │     Default: 4,096 KB (4 MB!)                                │
  │     Unit: KILOBYTES                                          │
  │     Error: HttpException                                     │
  │                                                              │
  │  ⚠️ BOTH must be set — the LOWER one wins!                   │
  │  Most people only set one and can't figure out why           │
  │  uploads fail.                                               │
  └──────────────────────────────────────────────────────────────┘
```

```xml
<!-- Set both for 100MB uploads -->
<system.webServer>
    <security>
        <requestFiltering>
            <!-- IIS level: 100MB in bytes -->
            <requestLimits maxAllowedContentLength="104857600" />
        </requestFiltering>
    </security>
</system.webServer>

<system.web>
    <!-- ASP.NET level: 100MB in kilobytes + execution timeout -->
    <httpRuntime maxRequestLength="102400" executionTimeout="3600" />
</system.web>
```

---

### Q9: How to run IIS in a Windows Docker container?

```dockerfile
# Windows Server Core with IIS
FROM mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

# Remove default site content
RUN powershell -Command Remove-Item -Recurse C:\inetpub\wwwroot\*

# Copy application
COPY ./publish/ C:/inetpub/wwwroot/

# Install ASP.NET Core Hosting Bundle (if needed)
RUN powershell -Command \
    Invoke-WebRequest -Uri 'https://download.visualstudio.microsoft.com/...' -OutFile dotnet-hosting.exe ; \
    Start-Process dotnet-hosting.exe -ArgumentList '/install','/quiet' -Wait ; \
    Remove-Item dotnet-hosting.exe

# Configure IIS
RUN powershell -Command \
    Import-Module WebAdministration ; \
    Set-ItemProperty 'IIS:\AppPools\DefaultAppPool' -Name managedRuntimeVersion -Value ''

EXPOSE 80

# IIS runs as a service — use ServiceMonitor to keep container alive
ENTRYPOINT ["C:\\ServiceMonitor.exe", "w3svc"]
```

**Gotchas:**

1. **ServiceMonitor.exe** — IIS runs as a Windows service, not a foreground process. `ServiceMonitor.exe` watches the W3SVC service and keeps the container alive. Without it, the container exits immediately.

2. **Windows container size:** ~5 GB for Server Core. Use Nano Server for smaller images, but Nano Server doesn't support full IIS.

3. **Licensing:** Windows containers require a Windows Server license on the host.

4. **No Linux containers:** IIS only runs in Windows containers on Windows hosts. Can't mix with Linux containers on the same Docker engine without WSL2/Hyper-V switching.

---

### Q10: My URL Rewrite rules silently fail — no errors, no redirect?

```
  Debugging URL Rewrite:
  ┌──────────────────────────────────────────────────────────────┐
  │  1. Enable Failed Request Tracing for status 200             │
  │     → FREB shows exactly which rules were evaluated          │
  │     → And WHY they didn't match                              │
  │                                                              │
  │  2. Check rule order — first match with stopProcessing=true  │
  │     wins and stops further evaluation                        │
  │                                                              │
  │  3. Server Variables not allowed?                            │
  │     → URL Rewrite → View Server Variables → whitelist vars   │
  │     → Rewrite rules setting HTTP_X_FORWARDED_HOST etc.       │
  │       need the variable to be explicitly allowed             │
  │                                                              │
  │  4. ARR proxy not enabled?                                   │
  │     → Rewrite rules of type "Rewrite" to external URLs       │
  │       require ARR proxy to be enabled at server level        │
  │                                                              │
  │  5. Regex escaping                                           │
  │     → Dots must be escaped: \. not .                         │
  │     → Backslash in URLs: use / not \\                        │
  │     → Test regex at: https://regex101.com                    │
  └──────────────────────────────────────────────────────────────┘
```

---

### Q11: How to handle Windows Authentication through a reverse proxy?

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Problem: Kerberos authentication doesn't work through       │
  │  a reverse proxy (Kerberos tickets are bound to the SPN      │
  │  of the ORIGINAL server, not the proxy).                     │
  │                                                              │
  │  Solutions:                                                  │
  │                                                              │
  │  1. Kerberos Constrained Delegation (KCD)                    │
  │     → Configure the proxy server's computer account          │
  │       in AD to delegate to the backend SPN                   │
  │     → Proxy authenticates the user, then requests a          │
  │       new Kerberos ticket for the backend on the             │
  │       user's behalf                                          │
  │                                                              │
  │  2. Protocol Transition (S4U)                                │
  │     → Proxy receives non-Kerberos auth (Basic, Forms)        │
  │     → Proxy uses S4U2Self to get a Kerberos ticket           │
  │       for the user, then S4U2Proxy to access backend         │
  │                                                              │
  │  3. NTLM Fallback                                            │
  │     → NTLM works through proxies (challenge-response)        │
  │     → But NTLM is slower and less secure than Kerberos       │
  │     → May break with some proxy configurations               │
  │                                                              │
  │  4. Claims-Based Auth (convert to tokens)                    │
  │     → Authenticate at proxy, issue JWT/SAML token            │
  │     → Backend validates token instead of Windows auth        │
  │     → Most scalable and modern approach                      │
  └──────────────────────────────────────────────────────────────┘
```

> **This is the #1 reason Windows Auth + reverse proxy projects fail.** Always plan the auth delegation strategy before designing the proxy architecture.

---

### Q12: IIS vs Kestrel — when to use which for ASP.NET Core?

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Kestrel (stand-alone)      │  IIS + Kestrel (recommended)   │
  │                             │                                │
  │  dotnet run                 │  IIS → ANCM → Kestrel          │
  │  ├── Fast, cross-platform   │  ├── Windows Auth ✅           │
  │  ├── No Windows Auth ❌     │  ├── Port sharing ✅           │
  │  ├── No port 80 sharing     │  ├── SSL cert management ✅    │
  │  ├── No process management  │  ├── Process management ✅     │
  │  ├── Limited SSL mgmt       │  ├── Static file caching ✅    │
  │  │                          │  ├── Request filtering ✅      │
  │  │  Use for:                │  │  Use for:                   │
  │  │  • Linux deployment      │  │  • Windows Server prod      │
  │  │  • Docker containers     │  │  • Intranet apps w/ Win Auth│
  │  │  • Behind Nginx/Apache   │  │  • Enterprise environments  │
  │  └──────────────────────────┘  └─────────────────────────────┘
  │                                                              │
  │  In-Process (IIS + ANCM InProcess):                          │
  │  → Kestrel is NOT used — app runs directly in w3wp.exe       │
  │  → Fastest option on Windows                                 │
  │  → Uses HTTP.sys kernel-mode features                        │
  └──────────────────────────────────────────────────────────────┘
```

---

### Q13: How to migrate from IIS to Nginx (or use both)?

```
  Common pattern: Nginx (edge) → IIS (application)
  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │  Internet → Nginx (Linux)    → IIS (Windows)                 │
  │             ├── SSL termination  ├── ASP.NET app             │
  │             ├── Rate limiting    ├── Windows Auth            │
  │             ├── Static files     ├── .NET backend            │
  │             ├── Compression      └── Active Directory        │
  │             └── Load balancing                               │
  │                                                              │
  │  This is the best of both worlds:                            │
  │  • Nginx handles edge concerns (cheap, efficient)            │
  │  • IIS handles Windows-native concerns                       │
  │  • Backend is not directly exposed to the internet           │
  └──────────────────────────────────────────────────────────────┘
```

```nginx
# Nginx config proxying to IIS
upstream iis_backend {
    server 10.0.0.50:80;
    server 10.0.0.51:80;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/ssl/certs/app.crt;
    ssl_certificate_key /etc/ssl/private/app.key;

    location / {
        proxy_pass http://iis_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

### Q14: When should I choose IIS over Nginx/Apache?

```
  Choose IIS when:
  ┌──────────────────────────────────────────────────────────────┐
  │ ✅ Running ASP.NET Framework (4.x) — IIS is the only host    │
  │ ✅ Need native Windows Authentication (Kerberos/NTLM/SSO)    │
  │ ✅ Active Directory integration is critical                  │
  │ ✅ Your ops team knows Windows but not Linux                 │
  │ ✅ Compliance requires Windows (some govt/enterprise reqs)   │
  │ ✅ ASP.NET Core in-process hosting (best .NET perf)          │
  │ ✅ Need IIS Manager GUI for less technical staff             │
  │ ✅ Running legacy COM/ISAPI apps                             │
  └──────────────────────────────────────────────────────────────┘

  Choose Nginx/Apache when:
  ┌──────────────────────────────────────────────────────────────┐
  │ ✅ Non-Windows workloads (PHP, Python, Node, Go, Java)       │
  │ ✅ Containers & Kubernetes (Linux containers are standard)   │
  │ ✅ Budget-conscious (no Windows Server license cost)         │
  │ ✅ Maximum concurrency / minimum memory footprint            │
  │ ✅ Cloud-native / multi-cloud portability                    │
  │ ✅ Edge proxy / CDN / API gateway use case                   │
  └──────────────────────────────────────────────────────────────┘
```

---

### Q15: What are the IIS sub-status codes and why do they matter?

| Status | Sub-Status | Meaning |
|--------|-----------|---------|
| 400 | 400.0 | Bad request |
| 401 | 401.1 | Logon failed |
| 401 | 401.2 | Logon failed (server config) |
| 401 | 401.3 | ACL on resource denied |
| 403 | 403.14 | Directory listing denied |
| 403 | 403.18 | URL sequence denied (request filtering) |
| 404 | 404.0 | File not found |
| 404 | 404.3 | **MIME type restriction** (missing MIME map!) |
| 404 | 404.13 | Content length too large |
| 404 | 404.15 | Query string too long |
| 500 | 500.0 | Internal server error (app crash) |
| 500 | 500.19 | **Config error** (bad web.config) |
| 500 | 500.21 | Handler module not found |
| 500 | 500.24 | ASP.NET in Classic mode but needs Integrated |
| 502 | 502.3 | **ARR proxy timeout** |
| 503 | 503.0 | AppPool is stopped |
| 503 | 503.2 | Concurrent request limit exceeded |

> **These sub-status codes are IIS-specific** — browsers show generic "500 Internal Server Error." Check IIS logs for the sub-status code, which pinpoints the exact problem. This is one area where IIS diagnostics are actually **superior** to Nginx/Apache.

---

> **End of IIS Guide** | [🏠 Home](../README.md) · [Servers](README.md) · [← Prev: Apache](apache.md)
