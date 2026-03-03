[🏠 Home](../README.md) · [Servers](README.md)

# 🪶 Apache HTTP Server — Advanced Configuration & Production Guide

> **Audience:** DevOps / Cloud Engineers & Trainees | **Focus:** Architecture, Production Config, Gotchas  
> **Covers:** Reverse Proxy, SSL/TLS, Static Hosting, Dynamic Scripting (mod_php, CGI, WSGI), Virtual Hosts, .htaccess, Hardening  
> **Tip:** Sections build on each other — read sequentially first, then use TOC for reference.

---

## Table of Contents

1. [Architecture & Mental Model](#1-architecture--mental-model)
2. [Installation & File Layout](#2-installation--file-layout)
3. [Core Configuration Anatomy](#3-core-configuration-anatomy)
4. [Virtual Hosts (vhosts)](#4-virtual-hosts-vhosts)
5. [Static Site Hosting](#5-static-site-hosting)
6. [Reverse Proxy (mod_proxy)](#6-reverse-proxy-mod_proxy)
7. [HTTPS & SSL/TLS Certificate Installation](#7-https--ssltls-certificate-installation)
8. [Server-Side Scripting — PHP, Python, CGI](#8-server-side-scripting--php-python-cgi)
9. [.htaccess — The Love-Hate File](#9-htaccess--the-love-hate-file)
10. [Load Balancing (mod_proxy_balancer)](#10-load-balancing-mod_proxy_balancer)
11. [Caching & Performance Tuning](#11-caching--performance-tuning)
12. [Security Hardening](#12-security-hardening)
13. [Logging, Debugging & Monitoring](#13-logging-debugging--monitoring)
14. [Production Gotchas & Tricky Q&A](#14-production-gotchas--tricky-qa)

---

## 1. Architecture & Mental Model

### 1.1 What Is Apache HTTP Server?

Apache (httpd) is the **most widely deployed web server in history**. It's a **modular, process/thread-based** server. Unlike Nginx's compiled-in approach, Apache can load/unload modules at runtime and supports per-directory configuration via `.htaccess` files.

### 1.2 Multi-Processing Modules (MPMs) — The Core Architecture Choice

Apache's concurrency strategy is **pluggable** via MPMs. This is the single most important architectural decision.

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │               Apache MPM Models (pick ONE at runtime)               │
  │                                                                     │
  │  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────┐  │
  │  │     prefork         │  │     worker          │  │   event     │  │
  │  │                     │  │                     │  │  (default)  │  │
  │  │  ┌───┐ ┌───┐ ┌───┐  │  │  ┌──────────────┐   │  │  ┌────────┐ │  │
  │  │  │ P │ │ P │ │ P │  │  │  │  Process 1   │   │  │  │Proc 1  │ │  │
  │  │  │ r │ │ r │ │ r │  │  │  │ ┌──┐┌──┐┌──┐ │   │  │  │ ┌─┐┌─┐ │ │  │
  │  │  │ o │ │ o │ │ o │  │  │  │ │T1││T2││T3│ │   │  │  │ │T││T│ │ │  │
  │  │  │ c │ │ c │ │ c │  │  │  │ └──┘└──┘└──┘ │   │  │  │ └─┘└─┘ │ │  │
  │  │  └───┘ └───┘ └───┘  │  │  └──────────────┘   │  │  └────────┘ │  │
  │  │                     │  │  ┌──────────────┐   │  │  Threads    │  │
  │  │  1 process =        │  │  │  Process 2   │   │  │  handle     │  │
  │  │    1 connection     │  │  │ ┌──┐┌──┐┌──┐ │   │  │  keepalive  │  │
  │  │                     │  │  │ │T1││T2││T3│ │   │  │  WITHOUT    │  │
  │  │  Heavy but safe     │  │  │ └──┘└──┘└──┘ │   │  │  blocking   │  │
  │  │  (mod_php needs     │  │  └──────────────┘   │  │  a thread   │  │
  │  │   this)             │  │                     │  │             │  │
  │  │                     │  │  Hybrid:            │  │  Best for   │  │
  │  │  Memory: HIGH       │  │  process + threads  │  │  modern     │  │
  │  │  Concurrency: LOW   │  │                     │  │  workloads  │  │
  │  │                     │  │  Memory: MEDIUM     │  │             │  │
  │  │                     │  │  Concurrency: GOOD  │  │  Memory:LOW │  │
  │  │                     │  │                     │  │  Conc: HIGH │  │
  │  └─────────────────────┘  └─────────────────────┘  └─────────────┘  │
  └─────────────────────────────────────────────────────────────────────┘
```

### 1.3 MPM Comparison Table

| Aspect | prefork | worker | event (default) |
|--------|---------|--------|-----------------|
| **Model** | 1 process per connection | Process pool, each with threads | Like worker, but async keepalive |
| **Concurrency** | Low (~256 default) | Medium (~400 threads) | High (~400+ threads, idle connections don't block) |
| **Memory** | Very high (full process per conn) | Medium | Lowest |
| **Thread-safety required?** | No — isolated processes | Yes — shared memory | Yes |
| **mod_php compatible?** | ✅ Yes (traditional) | ❌ No (not thread-safe) | ❌ No (use PHP-FPM) |
| **Best for** | Legacy PHP apps | Mixed workloads | Modern apps, reverse proxy, high traffic |

> **DevOps Relevance:** If you're deploying Apache in 2026, use **event MPM + PHP-FPM** (not mod_php + prefork). The prefork+mod_php combo is a legacy pattern that eats RAM. Event MPM is comparable to Nginx for reverse proxy workloads.

### 1.4 Apache vs Nginx — Architecture Comparison

```
  Apache (event MPM)                    Nginx
  ┌──────────────────────────┐          ┌──────────────────────────┐
  │ Process pool + threads   │          │ Worker processes +       │
  │ per connection           │          │ event loop (epoll)       │
  │                          │          │                          │
  │ + .htaccess support      │          │ + Lower memory           │
  │ + Runtime module loading │          │ + Higher concurrency     │
  │ + mod_php (legacy)       │          │ + Better static content  │
  │ + mod_rewrite power      │          │ + Simpler config syntax  │
  │                          │          │                          │
  │ - Higher memory/conn     │          │ - No .htaccess           │
  │ - Complex config syntax  │          │ - Modules compiled in    │
  │ - Slower for static      │          │ - No built-in scripting  │
  │                          │          │                          │
  │ Best: shared hosting,    │          │ Best: reverse proxy,     │
  │ .htaccess-dependent      │          │ containers, K8s ingress, │
  │ apps, mod_rewrite-heavy  │          │ static CDN, API gateway  │
  └──────────────────────────┘          └──────────────────────────┘
```

---

## 2. Installation & File Layout

### 2.1 Installation

```bash
# --- RHEL/CentOS/Rocky/Alma ---
sudo dnf install httpd mod_ssl -y
sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --reload

# --- Ubuntu/Debian ---
sudo apt update
sudo apt install apache2 -y
sudo systemctl enable --now apache2
sudo ufw allow 'Apache Full'

# --- Verify ---
httpd -v            # RHEL: shows version
apache2 -v          # Debian: shows version
curl -I http://localhost
```

> **⚠️ Gotcha:** The binary/service name differs by distro: `httpd` on RHEL vs `apache2` on Debian. Config paths differ too. This trips people up constantly.

### 2.2 File Layout — Mental Model

```
  RHEL/CentOS                                Debian/Ubuntu
  ─────────────────────────────               ─────────────────────────────
  /etc/httpd/                                 /etc/apache2/
  ├── conf/                                   ├── apache2.conf         ← Master config
  │   └── httpd.conf       ← Master config    ├── ports.conf           ← Listen directives
  ├── conf.d/              ← Drop-in configs  ├── conf-available/      ← Available config snippets
  │   ├── ssl.conf         ← SSL defaults     ├── conf-enabled/        ← Symlinks to active configs
  │   └── *.conf           ← Custom vhosts    ├── mods-available/      ← Available modules
  ├── conf.modules.d/     ← Module loads      ├── mods-enabled/        ← Symlinks to active modules
  │   └── 00-*.conf                           ├── sites-available/     ← All vhost definitions
  └── modules/            ← .so module files  ├── sites-enabled/       ← Symlinks to active vhosts
                                              └── envvars              ← Environment variables
  /var/log/httpd/                             /var/log/apache2/
  ├── access_log                              ├── access.log
  └── error_log                               └── error.log

  /var/www/html/           ← Default doc root (both distros)
  /run/httpd/              ← Runtime PID, sockets
```

### 2.3 Debian's a2en*/a2dis* Tools

Debian provides helper scripts to enable/disable modules, sites, and configs:

```bash
# Modules
sudo a2enmod rewrite ssl proxy proxy_http headers
sudo a2dismod autoindex

# Sites (vhosts)
sudo a2ensite mysite.conf
sudo a2dissite 000-default.conf

# Config snippets
sudo a2enconf security
sudo a2disconf charset

# After any change:
sudo systemctl reload apache2
```

> **RHEL equivalent:** No helper scripts — you manually edit `conf.d/*.conf` and `conf.modules.d/*.conf`. Use `LoadModule` directives directly.

### 2.4 Config Testing & Reload

```bash
# Test syntax
apachectl configtest        # or: httpd -t / apache2ctl -t

# Show loaded modules
apachectl -M                # or: httpd -M

# Dump full parsed config (vhosts)
apachectl -S                # Shows which vhost handles which domain

# Graceful reload (zero downtime)
sudo apachectl graceful     # or: systemctl reload httpd

# Hard restart
sudo systemctl restart httpd
```

> **Production Rule:** Always `apachectl configtest` then `apachectl graceful`. Never blind `restart`.

---

## 3. Core Configuration Anatomy

### 3.1 Config Structure — Directive Contexts

```
  ┌────────────────────────────────────────────────────────────────────┐
  │  httpd.conf / apache2.conf — Configuration Contexts                │
  │                                                                    │
  │  Server Config (global scope)                                      │
  │  ├── ServerRoot "/etc/httpd"                                       │
  │  ├── Listen 80                                                     │
  │  ├── LoadModule rewrite_module modules/mod_rewrite.so              │
  │  │                                                                 │
  │  ├── <VirtualHost *:80>            ← Per-domain config             │
  │  │     ├── ServerName example.com                                  │
  │  │     ├── DocumentRoot /var/www/example                           │
  │  │     │                                                           │
  │  │     ├── <Directory /var/www/example>   ← Per-directory config   │
  │  │     │     ├── Options Indexes FollowSymLinks                    │
  │  │     │     ├── AllowOverride All                                 │
  │  │     │     └── Require all granted                               │
  │  │     │   </Directory>                                            │
  │  │     │                                                           │
  │  │     ├── <Location /api>         ← Per-URL-path config           │
  │  │     │     └── ProxyPass http://localhost:3000                   │
  │  │     │   </Location>                                             │
  │  │     │                                                           │
  │  │     └── <Files ".ht*">          ← Per-filename config           │
  │  │           └── Require all denied                                │
  │  │         </Files>                                                │
  │  │   </VirtualHost>                                                │
  │  │                                                                 │
  │  └── Include conf.d/*.conf         ← Include drop-in files         │
  └────────────────────────────────────────────────────────────────────┘
```

### 3.2 Directive Context Hierarchy

| Context | Container | Scope | Override possible? |
|---------|-----------|-------|-------------------|
| **Server config** | (global / no container) | Entire server | — |
| **VirtualHost** | `<VirtualHost>` | Per domain/IP:port | — |
| **Directory** | `<Directory>`, `<DirectoryMatch>` | Filesystem path | Via `.htaccess` |
| **Location** | `<Location>`, `<LocationMatch>` | URL path | No |
| **Files** | `<Files>`, `<FilesMatch>` | Filename pattern | Via `.htaccess` |
| **.htaccess** | (per-directory file) | Directory + children | Controls via `AllowOverride` |

### 3.3 Merge Order — Which Context Wins?

When multiple contexts match, Apache merges them in this order (last wins):

```
  Merge Priority (lowest → highest):
  ┌───────────────────────────────────────────────────┐
  │  1. <Directory>  (shortest path first)            │
  │  2. <DirectoryMatch> / regex <Directory>          │
  │  3. .htaccess  (same level as matching Directory) │
  │  4. <Files> / <FilesMatch>                        │
  │  5. <Location> / <LocationMatch>                  │
  └───────────────────────────────────────────────────┘

  Example conflict:
  <Directory /var/www/html>
      Require all granted         ← Step 1
  </Directory>

  <Location /secret>
      Require all denied          ← Step 5 WINS — overrides Directory
  </Location>
```

> **⚠️ Gotcha:** `<Location>` beats `<Directory>`. If you set `Require all denied` in a `<Directory>` block but `Require all granted` in a `<Location>` block matching the same path, `<Location>` wins and the directory is exposed. This is a common security mistake.

### 3.4 The `Require` Directive (Apache 2.4+ Access Control)

```apache
# Allow everyone
Require all granted

# Deny everyone
Require all denied

# Allow specific IPs
Require ip 10.0.0.0/8 192.168.1.0/24

# Allow authenticated users
Require valid-user

# Allow specific user
Require user admin john

# Complex logic
<RequireAll>
    Require ip 10.0.0.0/8
    Require valid-user
</RequireAll>

<RequireAny>
    Require ip 127.0.0.1
    Require valid-user
</RequireAny>
```

> **⚠️ Gotcha (Apache 2.2 → 2.4 migration):** Old `Order Allow,Deny` / `Allow from` / `Deny from` directives are **deprecated in 2.4**. They still work if `mod_access_compat` is loaded, but mixing old and new syntax causes unpredictable behavior. Use `Require` exclusively.

---

## 4. Virtual Hosts (vhosts)

### 4.1 Name-Based vs IP-Based Virtual Hosts

```
  Name-Based Virtual Hosts               IP-Based Virtual Hosts
  (most common — shared IP)              (separate IP per site)

  ┌─────────────┐                        ┌──────────┐
  │ Client      │                        │ Client   │
  │ Host: a.com ├──┐                     │          │
  └─────────────┘  │  ┌───────────┐      └────┬─────┘
                   ├──►           │           │
  ┌─────────────┐  │  │  Apache   │      ┌────▼─────┐
  │ Client      │  │  │ (one IP)  │      │ Apache   │
  │ Host: b.com ├──┘  │           │      │ 10.0.0.1 │ ──► Site A
  └─────────────┘     │ Routes by │      │ 10.0.0.2 │ ──► Site B
                      │ Host hdr  │      └──────────┘
                      └───────────┘
                      Routes by              Routes by
                      Host header            IP address
```

### 4.2 Name-Based Virtual Host Configuration

**RHEL/CentOS — `/etc/httpd/conf.d/mysite.conf`**  
**Debian/Ubuntu — `/etc/apache2/sites-available/mysite.conf`**

```apache
<VirtualHost *:80>
    ServerName    mysite.com
    ServerAlias   www.mysite.com
    DocumentRoot  /var/www/mysite
    ServerAdmin   admin@mysite.com

    <Directory /var/www/mysite>
        Options -Indexes +FollowSymLinks    # Disable dir listing, allow symlinks
        AllowOverride All                    # Allow .htaccess
        Require all granted
    </Directory>

    ErrorLog  /var/log/httpd/mysite-error.log
    CustomLog /var/log/httpd/mysite-access.log combined
</VirtualHost>
```

```bash
# Debian: Enable the site
sudo a2ensite mysite.conf
sudo systemctl reload apache2

# RHEL: Just place in conf.d/ — auto-loaded
sudo systemctl reload httpd

# Verify vhost routing
apachectl -S
```

### 4.3 Default / Catch-All VirtualHost

```apache
# This catches requests that don't match any ServerName
<VirtualHost _default_:80>
    ServerName  catchall.localhost
    DocumentRoot /var/www/default

    # Option 1: Show a default page
    <Directory /var/www/default>
        Require all granted
    </Directory>

    # Option 2: Reject with 444-like behavior
    # Redirect 404 /
</VirtualHost>
```

> **⚠️ Gotcha:** The **first** `<VirtualHost>` in load order is the default for unmatched requests. On Debian, `000-default.conf` is the catch-all. On RHEL, it's whatever comes first alphabetically in `conf.d/`. If you delete the default without a catch-all, random sites may be served.

### 4.4 `apachectl -S` Output — How to Read It

```
VirtualHost configuration:
*:80     is a NameVirtualHost
   default server mysite.com (/etc/httpd/conf.d/mysite.conf:1)
   port 80 namevhost mysite.com (/etc/httpd/conf.d/mysite.conf:1)
   port 80 namevhost api.mysite.com (/etc/httpd/conf.d/api.conf:1)
*:443    is a NameVirtualHost
   ...
```

This command is your **best friend** for debugging "wrong site showing" issues.

---

## 5. Static Site Hosting

### 5.1 Basic Static Site

```apache
<VirtualHost *:80>
    ServerName  static.example.com
    DocumentRoot /var/www/static-site

    <Directory /var/www/static-site>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    # Cache static assets
    <FilesMatch "\.(css|js|jpg|jpeg|png|gif|ico|svg|woff2?)$">
        Header set Cache-Control "public, max-age=2592000, immutable"
    </FilesMatch>

    # Deny hidden files
    <FilesMatch "^\.">
        Require all denied
    </FilesMatch>
</VirtualHost>
```

### 5.2 `DocumentRoot` vs `Alias` vs `ScriptAlias`

```
  DocumentRoot:  Maps / to a filesystem directory
  ┌──────────────────────────────────────────────────────────┐
  │  DocumentRoot /var/www/site                              │
  │                                                          │
  │  Request:  /images/logo.png                              │
  │  File:     /var/www/site/images/logo.png                 │
  └──────────────────────────────────────────────────────────┘

  Alias:  Maps a URL prefix to a DIFFERENT filesystem path
  ┌──────────────────────────────────────────────────────────┐
  │  Alias /downloads /data/shared/files                     │
  │                                                          │
  │  Request:  /downloads/report.pdf                         │
  │  File:     /data/shared/files/report.pdf                 │
  │            (/data/shared/files replaces /downloads)      │
  └──────────────────────────────────────────────────────────┘

  ScriptAlias:  Like Alias but marks directory as CGI-executable
  ┌──────────────────────────────────────────────────────────┐
  │  ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/                 │
  │                                                          │
  │  Request:  /cgi-bin/script.py                            │
  │  Executes: /usr/lib/cgi-bin/script.py  (runs as CGI)     │
  └──────────────────────────────────────────────────────────┘
```

> **⚠️ Gotcha:** `Alias` path does NOT include the matched prefix. `Alias /img /data/images` + request `/img/cat.jpg` → `/data/images/cat.jpg`. But if you forget the trailing slash consistency, you get broken paths. Match slashes on both sides.

### 5.3 SPA Hosting (React / Vue / Angular)

Apache equivalent of Nginx's `try_files`:

```apache
<VirtualHost *:80>
    ServerName app.example.com
    DocumentRoot /var/www/spa/dist

    <Directory /var/www/spa/dist>
        Options -Indexes
        AllowOverride None
        Require all granted

        # SPA fallback — if file doesn't exist, serve index.html
        RewriteEngine On
        RewriteBase /
        RewriteRule ^index\.html$ - [L]
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule . /index.html [L]
    </Directory>
</VirtualHost>
```

> **Requires:** `sudo a2enmod rewrite` (Debian) or ensure `mod_rewrite` is loaded (RHEL).

### 5.4 Directory Listing

```apache
<Directory /var/www/shared>
    Options +Indexes
    IndexOptions FancyIndexing FoldersFirst NameWidth=* DescriptionWidth=*
    IndexIgnore .ht* *.log
    HeaderName /autoindex-header.html     # Custom header HTML
    ReadmeName /autoindex-footer.html     # Custom footer HTML
</Directory>
```

### 5.5 Custom Error Pages

```apache
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html
ErrorDocument 403 "Access Denied — contact admin@example.com"

# If using external URL (causes redirect — not recommended):
# ErrorDocument 404 http://example.com/not-found
```

> **⚠️ Gotcha:** String error documents (in quotes) are sent inline. File paths (starting with `/`) are served from `DocumentRoot`. External URLs cause a **302 redirect**, which changes the URL in the browser and returns a 302 status (not 404) — confusing for SEO and monitoring.

---

## 6. Reverse Proxy (mod_proxy)

### 6.1 Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │               Apache Reverse Proxy Stack                     │
  │                                                              │
  │  mod_proxy          ← Core proxy framework                   │
  │  ├── mod_proxy_http ← HTTP/HTTPS backend proxying            │
  │  ├── mod_proxy_fcgi ← FastCGI proxying (PHP-FPM)             │
  │  ├── mod_proxy_wstunnel ← WebSocket proxying                 │
  │  ├── mod_proxy_ajp  ← AJP protocol (Tomcat)                  │
  │  ├── mod_proxy_balancer ← Load balancing                     │
  │  └── mod_proxy_hcheck ← Health checks                        │
  │                                                              │
  │  Required modules to enable:                                 │
  │  proxy proxy_http proxy_wstunnel proxy_balancer              │
  │  lbmethod_byrequests headers ssl rewrite                     │
  └──────────────────────────────────────────────────────────────┘
```

```bash
# Enable required modules (Debian)
sudo a2enmod proxy proxy_http proxy_wstunnel proxy_balancer lbmethod_byrequests headers ssl rewrite

# RHEL: These are typically loaded by default; verify with:
httpd -M | grep proxy
```

### 6.2 Basic Reverse Proxy

```apache
<VirtualHost *:80>
    ServerName api.example.com

    # Prevent Apache from being used as a forward proxy
    ProxyRequests Off

    # Preserve the original Host header
    ProxyPreserveHost On

    # Proxy rules
    ProxyPass        / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    # Forward client information
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}e"
    RequestHeader set X-Forwarded-Proto "%{REQUEST_SCHEME}e"

    # Timeouts
    ProxyTimeout 60
</VirtualHost>
```

### 6.3 `ProxyPass` vs `ProxyPassReverse` — Why Both?

```
  ProxyPass:         Routes REQUESTS from client → backend
  ProxyPassReverse:  Rewrites RESPONSE headers (Location, Content-Location, URI)
                     so redirects point to the proxy, not the internal backend

  Without ProxyPassReverse:
  ┌────────┐   GET /app   ┌───────┐   GET /app    ┌─────────┐
  │ Client ├─────────────►│ Apache├──────────────►│ Backend │
  │        │              │       │               │ :3000   │
  │        │◄─────────────┤       │◄──────────────┤         │
  │        │  302 Location│       │  302 Location:│         │
  │        │  http://127. │       │  http://127.  │         │
  │        │  0.0.1:3000/ │       │  0.0.1:3000/  │         │
  │        │  login       │       │  login        │         │
  └────────┘  ← BROKEN!   └───────┘               └─────────┘
              Client tries
              to reach 127.0.0.1 directly

  With ProxyPassReverse:
  ┌────────┐   GET /app   ┌───────┐   GET /app    ┌─────────┐
  │ Client ├─────────────►│ Apache├──────────────►│ Backend │
  │        │              │       │               │ :3000   │
  │        │◄─────────────┤       │◄──────────────┤         │
  │        │  302 Location│       │  302 Location:│         │
  │        │  http://api. │  REWRITE  http://127. │         │
  │        │  example.com/│       │  0.0.1:3000/  │         │
  │        │  login       │       │  login        │         │
  └────────┘  ✅ CORRECT  └───────┘               └─────────┘
```

> **Rule:** Every `ProxyPass` should have a matching `ProxyPassReverse`. Always.

### 6.4 ProxyPass Ordering — The #1 Apache Proxy Gotcha

```apache
# ❌ WRONG ORDER — /app catches everything, /app/api never matches
ProxyPass /app   http://frontend:8080/
ProxyPass /app/api http://backend:3000/

# ✅ CORRECT — most specific FIRST
ProxyPass /app/api http://backend:3000/
ProxyPass /app     http://frontend:8080/

# OR use <Location> blocks (order doesn't matter — longest match wins)
<Location /app/api>
    ProxyPass http://backend:3000/
    ProxyPassReverse http://backend:3000/
</Location>
<Location /app>
    ProxyPass http://frontend:8080/
    ProxyPassReverse http://frontend:8080/
</Location>
```

> **⚠️ Critical Gotcha:** `ProxyPass` directives are matched **in order of appearance** (first match wins). Unlike `<Location>` blocks which use longest-match. This ordering difference causes 90% of Apache proxy routing bugs. Use `<Location>` blocks for sanity.

### 6.5 WebSocket Proxying

```apache
# Enable module
# Debian: sudo a2enmod proxy_wstunnel

<VirtualHost *:80>
    ServerName ws.example.com

    # WebSocket endpoints
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule ^/?(.*) ws://127.0.0.1:8080/$1 [P,L]

    # Regular HTTP fallback
    ProxyPass        / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/
</VirtualHost>

# Alternative (simpler, if WS is on a dedicated path):
<Location /ws/>
    ProxyPass        ws://127.0.0.1:8080/ws/
    ProxyPassReverse ws://127.0.0.1:8080/ws/
</Location>
```

### 6.6 Multi-Service Reverse Proxy (Microservices Gateway)

```
  ┌──────────────────────────────────────────────────────────┐
  │               Apache (Gateway)                           │
  │               :443 (SSL termination)                     │
  │                                                          │
  │  /api/users/*  ──────►  user-service:3001                │
  │  /api/orders/* ──────►  order-service:3002               │
  │  /api/auth/*   ──────►  auth-service:3003                │
  │  /             ──────►  frontend:8080  (SPA)             │
  └──────────────────────────────────────────────────────────┘
```

```apache
<VirtualHost *:443>
    ServerName app.example.com

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/app.crt
    SSLCertificateKeyFile /etc/ssl/private/app.key

    ProxyRequests Off
    ProxyPreserveHost On

    # Most specific paths FIRST
    <Location /api/users>
        ProxyPass        http://user-service:3001
        ProxyPassReverse http://user-service:3001
    </Location>

    <Location /api/orders>
        ProxyPass        http://order-service:3002
        ProxyPassReverse http://order-service:3002
    </Location>

    <Location /api/auth>
        ProxyPass        http://auth-service:3003
        ProxyPassReverse http://auth-service:3003
    </Location>

    # Frontend catch-all (LAST)
    ProxyPass        / http://frontend:8080/
    ProxyPassReverse / http://frontend:8080/
</VirtualHost>
```

---

## 7. HTTPS & SSL/TLS Certificate Installation

### 7.1 Let's Encrypt with Certbot

```bash
# Install Certbot
# RHEL: sudo dnf install certbot python3-certbot-apache -y
# Debian: sudo apt install certbot python3-certbot-apache -y

# Obtain + auto-configure
sudo certbot --apache -d example.com -d www.example.com

# Certbot will:
# 1. Verify domain (HTTP-01 challenge)
# 2. Install cert
# 3. Modify vhost to add SSL directives
# 4. Add HTTP → HTTPS redirect

# Auto-renewal
sudo certbot renew --dry-run
sudo systemctl enable --now certbot-renew.timer
```

### 7.2 Manual SSL Configuration

```apache
# Ensure mod_ssl is loaded
# RHEL: installed with httpd-mod_ssl or mod_ssl package
# Debian: sudo a2enmod ssl

<VirtualHost *:443>
    ServerName example.com

    # --- Certificate Files ---
    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile   /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/intermediate.crt    # RHEL-specific
    # On Debian, include the full chain in SSLCertificateFile

    # --- Protocol & Cipher Hardening ---
    SSLProtocol             all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
    SSLHonorCipherOrder     on
    SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    SSLCompression          off    # Prevent CRIME attack

    # --- OCSP Stapling ---
    SSLUseStapling          on
    SSLStaplingResponderTimeout 5
    SSLStaplingReturnResponderErrors off

    # --- HSTS ---
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"

    # --- Your site config ---
    DocumentRoot /var/www/example.com
    <Directory /var/www/example.com>
        Require all granted
    </Directory>
</VirtualHost>

# OCSP stapling cache (must be OUTSIDE VirtualHost, in server config)
SSLStaplingCache shmcb:/run/httpd/ssl_stapling(32768)

# --- HTTP → HTTPS Redirect ---
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    Redirect permanent / https://example.com/
</VirtualHost>
```

### 7.3 Certificate File Differences: Apache vs Nginx

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    Apache                 Nginx                  │
  │                                                                  │
  │  Leaf cert:   SSLCertificateFile         ssl_certificate         │
  │  Private key: SSLCertificateKeyFile      ssl_certificate_key     │
  │  Chain:       SSLCertificateChainFile    (included in            │
  │               (separate file)             ssl_certificate)       │
  │                                                                  │
  │  Apache 2.4.8+: Can use full chain in SSLCertificateFile         │
  │  (like Nginx), making SSLCertificateChainFile optional.          │
  └──────────────────────────────────────────────────────────────────┘
```

### 7.4 Redirect HTTP to HTTPS — Three Methods

```apache
# Method 1: Redirect directive (simplest, recommended)
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>

# Method 2: mod_rewrite (more control)
<VirtualHost *:80>
    ServerName example.com
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>

# Method 3: .htaccess (if no vhost access)
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
```

> **Best Practice:** Method 1 is fastest (no regex engine). Use Method 2 only when you need conditional logic. Method 3 is for shared hosting.

---

## 8. Server-Side Scripting — PHP, Python, CGI

### 8.1 How Apache Runs Dynamic Content

```
  ┌──────────────────────────────────────────────────────────────────┐
  │            Apache Dynamic Content Methods                        │
  │                                                                  │
  │  ┌───────────┐                                                   │
  │  │  Apache   │                                                   │
  │  │           ├── mod_php (in-process) ──► PHP runs inside Apache │
  │  │           │   ⚠️ Requires prefork MPM  (legacy, avoid)        │
  │  │           │                                                   │
  │  │           ├── mod_proxy_fcgi ────────► PHP-FPM (external)     │
  │  │           │   ✅ Works with event MPM  (modern, recommended)  │
  │  │           │                                                   │
  │  │           ├── mod_cgi / mod_cgid ───► CGI scripts (Perl, Py)  │
  │  │           │   ⚠️ Forks a new process per request (slow)       │
  │  │           │                                                   │
  │  │           ├── mod_proxy_http ────────► Gunicorn / Node.js     │
  │  │           │   ✅ HTTP proxy to any backend                    │
  │  │           │                                                   │
  │  │           └── mod_wsgi ─────────────► Python WSGI in-process  │
  │  │               ✅ Efficient for Django/Flask                   │
  │  └───────────┘                                                   │
  └──────────────────────────────────────────────────────────────────┘
```

### 8.2 PHP with PHP-FPM (Modern — Recommended)

```bash
# Install PHP-FPM
# RHEL: sudo dnf install php-fpm php-mysqlnd php-json php-mbstring -y
# Debian: sudo apt install php-fpm php-mysql php-json php-mbstring -y

# Enable required Apache modules
# Debian: sudo a2enmod proxy_fcgi setenvif
# RHEL: already loaded usually

# Start PHP-FPM
sudo systemctl enable --now php-fpm    # or php8.2-fpm
```

```apache
<VirtualHost *:80>
    ServerName phpapp.example.com
    DocumentRoot /var/www/phpapp

    <Directory /var/www/phpapp>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Route PHP to FPM via unix socket
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php-fpm/www.sock|fcgi://localhost"
        # Debian: proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost
    </FilesMatch>

    # OR route via TCP
    # <FilesMatch \.php$>
    #     SetHandler "proxy:fcgi://127.0.0.1:9000"
    # </FilesMatch>

    DirectoryIndex index.php index.html
</VirtualHost>
```

> **⚠️ Gotcha:** On RHEL, PHP-FPM defaults to socket at `/run/php-fpm/www.sock` with `apache:apache` ownership. On Debian, it's `/run/php/php8.2-fpm.sock` owned by `www-data:www-data`. Mismatch = `503 Service Unavailable`.

### 8.3 PHP with mod_php (Legacy — Avoid in New Projects)

```apache
# RHEL: sudo dnf install php php-mysqlnd -y
# This automatically installs mod_php and switches MPM to prefork

<VirtualHost *:80>
    ServerName legacy.example.com
    DocumentRoot /var/www/legacy

    <Directory /var/www/legacy>
        Options -Indexes
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler application/x-httpd-php
    </FilesMatch>

    DirectoryIndex index.php
</VirtualHost>
```

> **Why avoid mod_php?** It forces prefork MPM (1 process per connection), is NOT thread-safe, and bloats every Apache process with the PHP interpreter even for static file requests. PHP-FPM is superior in every way.

### 8.4 Python with mod_wsgi (Django / Flask)

```bash
# Install mod_wsgi
# RHEL: sudo dnf install python3-mod_wsgi -y
# Debian: sudo apt install libapache2-mod-wsgi-py3 -y
# Debian: sudo a2enmod wsgi
```

```apache
<VirtualHost *:80>
    ServerName pyapp.example.com

    # WSGI application
    WSGIDaemonProcess pyapp python-home=/var/www/pyapp/venv python-path=/var/www/pyapp
    WSGIProcessGroup pyapp
    WSGIScriptAlias / /var/www/pyapp/myproject/wsgi.py

    <Directory /var/www/pyapp/myproject>
        <Files wsgi.py>
            Require all granted
        </Files>
    </Directory>

    # Static files served directly by Apache (bypass WSGI)
    Alias /static /var/www/pyapp/staticfiles
    <Directory /var/www/pyapp/staticfiles>
        Require all granted
    </Directory>

    Alias /media /var/www/pyapp/media
    <Directory /var/www/pyapp/media>
        Require all granted
    </Directory>
</VirtualHost>
```

**Alternatively, proxy to Gunicorn (simpler):**

```apache
<VirtualHost *:80>
    ServerName pyapp.example.com

    ProxyPass        /static !              # Don't proxy static files
    ProxyPass        /media  !              # Don't proxy media files
    ProxyPass        / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/

    Alias /static /var/www/pyapp/staticfiles
    Alias /media  /var/www/pyapp/media
</VirtualHost>
```

> **ProxyPass exclusion trick:** `ProxyPass /static !` tells Apache "do NOT proxy requests starting with `/static`". This lets Apache serve static files directly, much faster than routing through Gunicorn.

### 8.5 Node.js

Same as Nginx — proxy to Node process:

```apache
<VirtualHost *:80>
    ServerName nodeapp.example.com

    ProxyRequests Off
    ProxyPreserveHost On

    ProxyPass        / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    # Serve static assets directly
    Alias /public /var/www/nodeapp/public
    ProxyPass /public !

    <Directory /var/www/nodeapp/public>
        Require all granted
        <FilesMatch "\.(css|js|jpg|png|svg|woff2?)$">
            Header set Cache-Control "public, max-age=2592000"
        </FilesMatch>
    </Directory>
</VirtualHost>
```

### 8.6 CGI Scripts (Legacy — Perl, Python, Shell)

```apache
ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/

<Directory /usr/lib/cgi-bin>
    Options +ExecCGI
    AddHandler cgi-script .py .pl .sh
    Require all granted
</Directory>
```

Example CGI script (`/usr/lib/cgi-bin/hello.py`):

```python
#!/usr/bin/env python3
print("Content-Type: text/html\n")
print("<h1>Hello from CGI!</h1>")
```

```bash
chmod +x /usr/lib/cgi-bin/hello.py
curl http://localhost/cgi-bin/hello.py
```

> **CGI is the original dynamic web — and it's horribly slow.** Every request forks a new process. Use only for legacy compatibility or quick admin scripts. Modern equivalent: FastCGI, WSGI, or HTTP proxy.

---

## 9. .htaccess — The Love-Hate File

### 9.1 What Is .htaccess?

`.htaccess` is a **per-directory configuration file** that Apache reads on every request. It lets users override server config without editing vhosts (critical for shared hosting).

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    .htaccess Flow                            │
  │                                                              │
  │  Request: /blog/2025/post.html                               │
  │                                                              │
  │  Apache checks:                                              │
  │  1. /.htaccess                 (if AllowOverride ≠ None)     │
  │  2. /blog/.htaccess            (if AllowOverride ≠ None)     │
  │  3. /blog/2025/.htaccess       (if AllowOverride ≠ None)     │
  │                                                              │
  │  ALL matching .htaccess files are merged (deepest wins)      │
  │                                                              │
  │  ⚠️ This happens on EVERY request — even for static files    │
  │  ⚠️ The filesystem traversal is a performance penalty        │
  └──────────────────────────────────────────────────────────────┘
```

### 9.2 `AllowOverride` Controls What .htaccess Can Do

```apache
# In vhost or <Directory> block:

AllowOverride None       # .htaccess completely ignored (fastest)
AllowOverride All        # .htaccess can do anything
AllowOverride AuthConfig # Only authentication directives
AllowOverride FileInfo   # Document type, error docs, rewrite rules
AllowOverride Indexes    # Directory indexing
AllowOverride Limit      # Access control (Require, etc.)
AllowOverride Options    # Options directive
```

> **Production Best Practice:** Set `AllowOverride None` globally, then `AllowOverride All` only in DocumentRoots that need it (e.g., WordPress). This eliminates the performance penalty of scanning for `.htaccess` in every directory.

### 9.3 Common .htaccess Recipes

```apache
# --- URL Rewrite: Pretty URLs ---
RewriteEngine On
RewriteBase /
RewriteRule ^index\.html$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.html [L]

# --- Force HTTPS ---
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]

# --- Block IPs ---
Require all granted
Require not ip 10.0.0.100
Require not ip 192.168.1.0/24

# --- Password Protect ---
AuthType Basic
AuthName "Restricted"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user

# --- Custom Error Pages ---
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html

# --- Prevent Hotlinking ---
RewriteEngine On
RewriteCond %{HTTP_REFERER} !^$
RewriteCond %{HTTP_REFERER} !^https?://(www\.)?example\.com [NC]
RewriteRule \.(jpg|jpeg|png|gif|svg)$ - [F,NC,L]

# --- Disable PHP Execution in Uploads ---
<FilesMatch "\.php$">
    Require all denied
</FilesMatch>

# --- Set Security Headers ---
Header set X-Frame-Options "SAMEORIGIN"
Header set X-Content-Type-Options "nosniff"
Header set X-XSS-Protection "1; mode=block"
```

### 9.4 .htaccess vs VHost Config — Performance Impact

```
  .htaccess enabled:
  ┌──────────────────────────────────────────────────┐
  │  Every request:                                  │
  │  1. Stat /.htaccess         (filesystem call)    │
  │  2. Stat /path/.htaccess    (filesystem call)    │
  │  3. Stat /path/sub/.htaccess (filesystem call)   │
  │  4. Parse each found file                        │
  │  5. Merge with server config                     │
  │                                                  │
  │  For 100 req/sec = 300 extra stat() calls/sec    │
  └──────────────────────────────────────────────────┘

  AllowOverride None (vhost config only):
  ┌──────────────────────────────────────────────────┐
  │  Config is parsed ONCE at startup                │
  │  Zero filesystem overhead per request            │
  │  ~15-20% faster for static file serving          │
  └──────────────────────────────────────────────────┘
```

> **Nginx doesn't have .htaccess — and that's by design.** Nginx processes config only at load/reload time. If you're migrating from Apache to Nginx, you must convert all `.htaccess` rules into `server {}` / `location {}` blocks.

---

## 10. Load Balancing (mod_proxy_balancer)

### 10.1 Architecture

```
                         ┌─────────────┐
                         │   Client    │
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │   Apache    │
                         │   (LB)      │
                         └──┬───┬───┬──┘
                            │   │   │
                   ┌────────┘   │   └────────┐
                   ▼            ▼            ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │ Worker 1│ │ Worker 2│ │ Worker 3│
              │ :3001   │ │ :3002   │ │ :3003   │
              └─────────┘ └─────────┘ └─────────┘
```

### 10.2 Load Balancing Methods

```apache
# --- Round Robin (byrequests — default) ---
<Proxy "balancer://backend">
    BalancerMember http://10.0.0.1:3000
    BalancerMember http://10.0.0.2:3000
    BalancerMember http://10.0.0.3:3000
</Proxy>

ProxyPass        / balancer://backend/
ProxyPassReverse / balancer://backend/

# --- Weighted ---
<Proxy "balancer://backend">
    BalancerMember http://10.0.0.1:3000 loadfactor=5
    BalancerMember http://10.0.0.2:3000 loadfactor=3
    BalancerMember http://10.0.0.3:3000 loadfactor=1
</Proxy>

# --- By Busyness (least connections equivalent) ---
<Proxy "balancer://backend">
    BalancerMember http://10.0.0.1:3000
    BalancerMember http://10.0.0.2:3000
    ProxySet lbmethod=bybusyness
</Proxy>

# --- Sticky Sessions (cookie-based) ---
<Proxy "balancer://backend">
    BalancerMember http://10.0.0.1:3000 route=node1
    BalancerMember http://10.0.0.2:3000 route=node2
    ProxySet stickysession=ROUTEID
</Proxy>

# Apache sets ROUTEID cookie to pin client to a backend

# --- Hot Standby ---
<Proxy "balancer://backend">
    BalancerMember http://10.0.0.1:3000
    BalancerMember http://10.0.0.2:3000
    BalancerMember http://10.0.0.3:3000 status=+H   # Hot standby — only if others fail
</Proxy>
```

### 10.3 Balancer Manager (Live Dashboard)

```apache
<Location "/balancer-manager">
    SetHandler balancer-manager
    Require ip 10.0.0.0/8
</Location>
```

> Access at `http://server/balancer-manager` — lets you view member status, enable/disable backends, change weights **live** without restart. Very useful for rolling deployments.

| Method | Module | Best For |
|--------|--------|----------|
| **byrequests** | `lbmethod_byrequests` | Stateless APIs, equal backends |
| **bytraffic** | `lbmethod_bytraffic` | Bandwidth-sensitive (video, downloads) |
| **bybusyness** | `lbmethod_bybusyness` | Varied response times |
| **heartbeat** | `lbmethod_heartbeat` | Cluster-aware (backends report load) |

---

## 11. Caching & Performance Tuning

### 11.1 mod_cache — Reverse Proxy Cache

```apache
# Load modules
# Debian: sudo a2enmod cache cache_disk headers

CacheEnable disk /
CacheRoot /var/cache/apache2/mod_cache_disk
CacheDefaultExpire 3600
CacheMaxExpire 86400
CacheIgnoreNoLastMod On
CacheLock On
CacheLockPath /tmp/mod_cache-lock
CacheLockMaxAge 5

# Don't cache authenticated content
CacheIgnoreHeaders Set-Cookie

# Show cache status in response
CacheHeader On    # Adds X-Cache: HIT/MISS header
```

### 11.2 Compression (mod_deflate)

```apache
# Debian: sudo a2enmod deflate

<IfModule mod_deflate.c>
    # Compress text-based content
    AddOutputFilterByType DEFLATE text/html text/plain text/css
    AddOutputFilterByType DEFLATE application/javascript application/json
    AddOutputFilterByType DEFLATE application/xml text/xml
    AddOutputFilterByType DEFLATE image/svg+xml
    AddOutputFilterByType DEFLATE application/vnd.ms-fontobject
    AddOutputFilterByType DEFLATE font/opentype font/ttf

    # Don't compress already-compressed formats
    SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png|webp|zip|gz|bz2|rar|mp[34]|avi|mov|woff2)$ no-gzip

    # Compression level (1-9, default 6)
    DeflateCompressionLevel 6

    # Add Vary header for proxies
    Header append Vary Accept-Encoding
</IfModule>
```

### 11.3 MPM Tuning (event MPM)

```apache
# /etc/httpd/conf.modules.d/00-mpm.conf (RHEL)
# /etc/apache2/mods-available/mpm_event.conf (Debian)

<IfModule mpm_event_module>
    StartServers             3        # Initial child processes
    MinSpareThreads         75        # Keep this many idle threads ready
    MaxSpareThreads        250        # Kill threads beyond this
    ThreadsPerChild         25        # Threads per child process
    MaxRequestWorkers      400        # Total concurrent connections (key limit)
    MaxConnectionsPerChild   0        # 0 = unlimited (restart child after N to prevent leaks)
    ServerLimit             16        # Max child processes (MaxRequestWorkers / ThreadsPerChild)
    AsyncRequestWorkerFactor 2        # event MPM: multiplier for async connections
</IfModule>
```

**Capacity calculation:**

```
  MaxRequestWorkers = ServerLimit × ThreadsPerChild
  Example: 16 × 25 = 400 concurrent connections

  Memory per child ≈ 50-150 MB (depends on modules loaded)
  Total RAM ≈ ServerLimit × memory_per_child
  Example: 16 × 100 MB = 1.6 GB for Apache processes

  Compare with Nginx: 4 workers × ~10 MB = 40 MB
  (Apache uses ~40x more RAM for similar concurrency)
```

### 11.4 KeepAlive Tuning

```apache
KeepAlive On
KeepAliveTimeout 5           # Close idle keepalive after 5 sec (default 5)
MaxKeepAliveRequests 100     # Max requests per keepalive connection (default 100)
```

> **⚠️ Gotcha:** High `KeepAliveTimeout` (e.g., 60s) with prefork MPM = process hoarding. 1000 idle keepalive connections = 1000 processes doing nothing. With event MPM, idle keepalive uses a listener thread instead of a worker thread, so it's much more efficient.

---

## 12. Security Hardening

### 12.1 Hide Apache Version & OS

```apache
# In httpd.conf or apache2.conf
ServerTokens Prod           # "Apache" only — no version, no OS
ServerSignature Off         # Remove server info from error pages
```

| ServerTokens Value | Example Header |
|-------------------|----------------|
| `Full` (default) | `Apache/2.4.57 (Ubuntu) OpenSSL/3.0.2` |
| `Prod` (secure) | `Apache` |
| `Major` | `Apache/2` |

### 12.2 Security Headers

```apache
<IfModule mod_headers.c>
    # Prevent clickjacking
    Header always set X-Frame-Options "SAMEORIGIN"

    # Prevent MIME sniffing
    Header always set X-Content-Type-Options "nosniff"

    # XSS protection
    Header always set X-XSS-Protection "1; mode=block"

    # Referrer policy
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    # HSTS
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"

    # Permissions Policy
    Header always set Permissions-Policy "camera=(), microphone=(), geolocation=()"

    # Remove ETag (leaks inode info)
    Header unset ETag
    FileETag None

    # Remove X-Powered-By (PHP sets this)
    Header always unset X-Powered-By
</IfModule>
```

### 12.3 Disable Unnecessary Modules

```bash
# List loaded modules
httpd -M

# Disable modules you don't need (Debian)
sudo a2dismod autoindex status info cgi

# Common modules to DISABLE in production:
# - mod_info      (exposes server config at /server-info)
# - mod_status    (exposes server status at /server-status) — or restrict to internal IPs
# - mod_autoindex (directory listing)
# - mod_cgi       (if not using CGI)
# - mod_userdir   (serves ~/public_html — security risk)
```

### 12.4 Restrict Sensitive Paths

```apache
# Block .git, .svn, .env, etc.
<DirectoryMatch "^\.|\/\.">
    Require all denied
</DirectoryMatch>

# Block wp-config.php (WordPress)
<Files "wp-config.php">
    Require all denied
</Files>

# Block access to backup files
<FilesMatch "\.(bak|old|orig|save|tmp|swp)$">
    Require all denied
</FilesMatch>

# Restrict server-status
<Location /server-status>
    SetHandler server-status
    Require ip 127.0.0.1
    Require ip 10.0.0.0/8
</Location>
```

### 12.5 mod_security (Web Application Firewall)

```bash
# Install ModSecurity
# RHEL: sudo dnf install mod_security mod_security_crs -y
# Debian: sudo apt install libapache2-mod-security2 -y
```

```apache
<IfModule security2_module>
    SecRuleEngine On
    SecRequestBodyAccess On
    SecResponseBodyAccess Off
    SecAuditEngine RelevantOnly
    SecAuditLog /var/log/httpd/modsec_audit.log

    # OWASP Core Rule Set (CRS)
    IncludeOptional /etc/httpd/modsecurity.d/owasp-crs/crs-setup.conf
    IncludeOptional /etc/httpd/modsecurity.d/owasp-crs/rules/*.conf
</IfModule>
```

> **DevOps Relevance:** mod_security + OWASP CRS provides protection against SQL injection, XSS, CSRF, path traversal — without modifying application code. It's the equivalent of AWS WAF or CloudFlare WAF, but self-hosted.

---

## 13. Logging, Debugging & Monitoring

### 13.1 Log Formats

```apache
# Default combined format
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined

# JSON format (for ELK / Loki / Datadog)
LogFormat "{ \"time\":\"%t\", \"remoteIP\":\"%a\", \"host\":\"%V\", \"request\":\"%U\", \"query\":\"%q\", \"method\":\"%m\", \"status\":\"%>s\", \"userAgent\":\"%{User-agent}i\", \"referer\":\"%{Referer}i\", \"duration\":\"%D\" }" json

CustomLog /var/log/httpd/access.log combined
# CustomLog /var/log/httpd/access-json.log json
```

### 13.2 Per-VHost Logging

```apache
<VirtualHost *:80>
    ServerName mysite.com
    ErrorLog  /var/log/httpd/mysite-error.log
    CustomLog /var/log/httpd/mysite-access.log combined
    LogLevel warn         # Per-vhost log level
</VirtualHost>
```

### 13.3 Log Levels

| Level | When to Use |
|-------|------------|
| `emerg` | System unusable |
| `alert` | Action required immediately |
| `crit` | Critical conditions |
| `error` | Error conditions (default for production) |
| `warn` | Warning conditions |
| `notice` | Normal but significant |
| `info` | Informational |
| `debug` | Debug-level (very verbose — dev only) |
| `trace1`-`trace8` | Ultra-verbose trace logging |

```apache
# Per-module debug logging
LogLevel warn rewrite:trace3 proxy:debug ssl:warn
```

### 13.4 mod_status (Monitoring Dashboard)

```apache
<IfModule mod_status.c>
    <Location /server-status>
        SetHandler server-status
        Require ip 127.0.0.1 10.0.0.0/8
    </Location>

    # Enable extended status for more metrics
    ExtendedStatus On
</IfModule>
```

Access: `http://server/server-status?auto` (machine-readable for Prometheus exporters)

Key metrics:

| Metric | Meaning |
|--------|---------|
| **Total Accesses** | Request count since start |
| **Total kBytes** | Bytes served |
| **BusyWorkers** | Active request handlers |
| **IdleWorkers** | Waiting for requests |
| **Scoreboard** | Per-slot status (`.` idle, `W` sending, `R` reading, `K` keepalive) |

### 13.5 Debugging Rewrite Rules

```apache
# Enable rewrite logging (Apache 2.4+)
LogLevel warn rewrite:trace3

# In error log you'll see:
# [rewrite:trace3] applying pattern '^(.*)$' to uri '/about'
# [rewrite:trace3] RewriteCond: input='/about' pattern='!-f' => matched
```

> This is invaluable when debugging complex `.htaccess` rewrite chains. Set it temporarily, reproduce the issue, then disable (trace logging is expensive).

---

## 14. Production Gotchas & Tricky Q&A

### Q1: Why does my Apache return 403 Forbidden on a brand new install?

**Answer:** Multiple causes — debug with this checklist:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  403 Forbidden Checklist:                                    │
  │                                                              │
  │  1. Filesystem permissions:                                  │
  │     → ls -la /var/www/html                                   │
  │     → Files need: 644 (rw-r--r--)                            │
  │     → Dirs need:  755 (rwxr-xr-x)                            │
  │     → Owner: apache:apache (RHEL) or www-data:www-data (Deb) │
  │                                                              │
  │  2. SELinux context (RHEL/CentOS):                           │
  │     → ls -Z /var/www/html                                    │
  │     → Must be: httpd_sys_content_t                           │
  │     → Fix: restorecon -Rv /var/www/html                      │
  │     → Or: chcon -R -t httpd_sys_content_t /var/www/html      │
  │                                                              │
  │  3. Directory block missing:                                 │
  │     → Need: <Directory /var/www/html>                        │
  │     →         Require all granted                            │
  │     →       </Directory>                                     │
  │                                                              │
  │  4. No index file + Options -Indexes:                        │
  │     → No index.html AND directory listing disabled           │
  │     → Fix: add index.html OR Options +Indexes                │
  │                                                              │
  │  5. .htaccess denying access:                                │
  │     → Check .htaccess in all parent directories              │
  └──────────────────────────────────────────────────────────────┘
```

> **#1 trap on RHEL/CentOS:** SELinux context. You copy files from `/tmp` or create them outside `/var/www` and they have the wrong SELinux label. `restorecon -Rv /var/www/html` fixes it.

---

### Q2: SELinux + Apache — The complete story

```
  Key SELinux booleans for Apache:
  ┌──────────────────────────────────────────────────────────────┐
  │  httpd_can_network_connect     → Proxy to backend ports      │
  │  httpd_can_network_connect_db  → Connect to DB ports         │
  │  httpd_can_sendmail            → Send emails via SMTP        │
  │  httpd_use_nfs                 → Serve files from NFS mount  │
  │  httpd_read_user_content       → Read files in /home/*/      │
  │  httpd_enable_homedirs         → Enable mod_userdir          │
  └──────────────────────────────────────────────────────────────┘

  setsebool -P httpd_can_network_connect 1    # Persist across reboot
  getsebool -a | grep httpd                   # List all httpd booleans
```

> **If your reverse proxy returns 503 on RHEL and the backend is running — it's SELinux 99% of the time.** Run `setsebool -P httpd_can_network_connect 1`.

---

### Q3: What happens when I mix `ProxyPass` outside `<Location>` with `ProxyPass` inside `<Location>`?

**Answer:** They follow **different matching rules**:

```
  Outside <Location>: First match wins (ORDER MATTERS)
  Inside <Location>:  Longest match wins (order doesn't matter)
  
  Mixing both: <Location> blocks are processed AFTER bare ProxyPass

  ⚠️ This means a bare ProxyPass / can swallow everything
     before <Location /api> is even evaluated.
```

**Fix:** Use EITHER bare `ProxyPass` directives OR `<Location>` blocks — never mix them for the same URL space.

---

### Q4: My .htaccess rewrite rules aren't working — why?

```
  Checklist:
  1. Is mod_rewrite loaded?        → apachectl -M | grep rewrite
  2. Is AllowOverride set?         → Must be All or FileInfo
  3. Is RewriteEngine On?          → Must be in the .htaccess file
  4. Is RewriteBase correct?       → Usually RewriteBase /
  5. File permissions?             → .htaccess must be readable by Apache
  6. Cached redirect?              → Clear browser cache / use curl
```

> **Debug tip:** Set `LogLevel warn rewrite:trace3` in the vhost and check `error_log`. It shows every rewrite step.

---

### Q5: Apache eats all my RAM — how to diagnose?

```bash
# Check memory per Apache process
ps aux | grep httpd | awk '{sum += $6; count++} END {print "Avg MB:", sum/count/1024, "Processes:", count}'

# Which MPM is running?
httpd -V | grep -i mpm

# If prefork + mod_php:
#   Each process = 50-200 MB
#   256 processes = 12-50 GB RAM
#
# SOLUTION: Switch to event MPM + PHP-FPM
```

```
  Memory comparison:
  ┌────────────────────────────────────────────────────────────┐
  │  prefork + mod_php:    256 procs × 100 MB = 25.6 GB ❌     │
  │  event + PHP-FPM:       16 procs × 50 MB  =  0.8 GB ✅     │
  │  (+ PHP-FPM pool:       10 procs × 30 MB  =  0.3 GB)       │
  │  Total event+FPM:                            1.1 GB        │
  └────────────────────────────────────────────────────────────┘
```

---

### Q6: `ProxyPassReverse` doesn't rewrite response body URLs — how to fix?

`ProxyPassReverse` only rewrites **HTTP headers** (`Location`, `Content-Location`, `URI`). It does NOT rewrite HTML body content.

```
  If backend returns:
  <a href="http://internal:3000/page">Link</a>

  ProxyPassReverse does NOT fix this.
```

**Fix:** Use `mod_substitute` or `mod_proxy_html`:

```apache
# mod_substitute (simple string replacement)
LoadModule substitute_module modules/mod_substitute.so

<Location />
    ProxyPass http://internal:3000/
    ProxyPassReverse http://internal:3000/
    AddOutputFilterByType SUBSTITUTE text/html
    Substitute "s|http://internal:3000|https://public.example.com|i"
</Location>
```

> **Better fix:** Have the backend use relative URLs or respect `X-Forwarded-*` headers to generate correct URLs.

---

### Q7: How to do graceful rolling restart during deployments?

```bash
# 1. Test config
apachectl configtest

# 2. Graceful restart (USR1 signal)
apachectl graceful
# - Existing connections finish with OLD config
# - New connections use NEW config
# - Zero dropped connections

# 3. For code deployments behind a load balancer:
#    a. Remove node from LB
#    b. Deploy code
#    c. apachectl graceful
#    d. Health check passes
#    e. Add node back to LB
```

---

### Q8: What's the difference between `Redirect`, `RedirectMatch`, `RewriteRule`, and `ProxyPass`?

| Directive | Module | What It Does | Client Sees Redirect? |
|-----------|--------|-------------|----------------------|
| `Redirect` | mod_alias | Simple URL → URL redirect | ✅ Yes (301/302) |
| `RedirectMatch` | mod_alias | Regex URL → URL redirect | ✅ Yes (301/302) |
| `RewriteRule` | mod_rewrite | Powerful URL manipulation | Optional (with [R] flag) |
| `ProxyPass` | mod_proxy | Transparent backend routing | ❌ No (invisible to client) |

```apache
# Redirect (simple, fast)
Redirect 301 /old-page https://example.com/new-page

# RedirectMatch (regex)
RedirectMatch 301 ^/blog/(.*)$ https://newblog.com/$1

# RewriteRule (most flexible)
RewriteRule ^/legacy/(.*)$ /modern/$1 [R=301,L]

# ProxyPass (transparent proxy — no redirect)
ProxyPass /api http://backend:3000/api
```

> **Rule of thumb:** Use `Redirect` for simple moves. Use `RewriteRule` for complex conditional logic. Use `ProxyPass` for backend routing. Don't use `RewriteRule` with `[P]` flag (proxy) when `ProxyPass` works — it's less efficient.

---

### Q9: Why does my Apache vhost show the wrong site?

```
  Debugging "wrong site served":
  ┌──────────────────────────────────────────────────────────────┐
  │  1. Run: apachectl -S                                        │
  │     → Shows exactly which vhost matches which domain         │
  │                                                              │
  │  2. Check: is the FIRST vhost the catch-all?                 │
  │     → Requests with unmatched Host header go to first vhost  │
  │                                                              │
  │  3. Check: ServerName vs ServerAlias spelling                │
  │     → Typos = unmatched = falls through to default           │
  │                                                              │
  │  4. Check: correct port                                      │
  │     → <VirtualHost *:80> won't match HTTPS on :443           │
  │     → Need separate vhosts for :80 and :443                  │
  │                                                              │
  │  5. Test: curl -H "Host: mysite.com" http://server-ip        │
  │     → Bypasses DNS to test vhost routing directly            │
  └──────────────────────────────────────────────────────────────┘
```

---

### Q10: How do I handle `MaxRequestWorkers` exhaustion?

When all workers are busy, new connections queue up, then get rejected (503).

```bash
# Monitor active workers
curl -s http://localhost/server-status?auto | grep -E "BusyWorkers|IdleWorkers"

# If BusyWorkers ≈ MaxRequestWorkers consistently:
# 1. Are there slow backends? → Increase proxy timeouts or fix backend
# 2. Stuck KeepAlive connections? → Reduce KeepAliveTimeout to 2-3s
# 3. Genuinely too much traffic? → Increase MaxRequestWorkers + add RAM
# 4. Memory leak? → Set MaxConnectionsPerChild to recycle processes

# Emergency: identify stuck connections
ss -tnp | grep httpd | wc -l
```

---

### Q11: `AH01630: client denied by server configuration` — what broke?

This is the **most confusing error in Apache 2.4** and it means access was denied.

```
  Causes:
  1. Missing <Directory> block with Require all granted
  2. Wrong Require directive (Require all denied by default in 2.4)
  3. Mixing Apache 2.2 syntax (Order/Allow/Deny) with 2.4 (Require)
  4. .htaccess with Require all denied inherited from parent dir
  
  Fix:
  <Directory /your/document/root>
      Require all granted    ← This is MANDATORY in 2.4
  </Directory>
```

> In Apache 2.2, the default was "allow all." In 2.4, the default is **"deny all"** unless you explicitly grant access. This change breaks nearly every migration.

---

### Q12: Apache behind CloudFlare / AWS ALB — how to get real client IP?

```apache
# mod_remoteip (replaces mod_rpaf)
LoadModule remoteip_module modules/mod_remoteip.so

# Trust CloudFlare IPs
RemoteIPHeader CF-Connecting-IP
RemoteIPTrustedProxy 173.245.48.0/20
RemoteIPTrustedProxy 103.21.244.0/22
RemoteIPTrustedProxy 103.22.200.0/22
RemoteIPTrustedProxy 103.31.4.0/22
RemoteIPTrustedProxy 141.101.64.0/18
RemoteIPTrustedProxy 108.162.192.0/18
RemoteIPTrustedProxy 190.93.240.0/20
RemoteIPTrustedProxy 188.114.96.0/20
RemoteIPTrustedProxy 197.234.240.0/22
RemoteIPTrustedProxy 198.41.128.0/17

# For AWS ALB:
# RemoteIPHeader X-Forwarded-For
# RemoteIPTrustedProxy 10.0.0.0/8     # VPC CIDR

# Now %h in logs = real client IP
# And REMOTE_ADDR in apps = real client IP
```

---

### Q13: How to run multiple PHP versions with Apache?

```
  ┌──────────────────────────────────────────────────────────────┐
  │  PHP-FPM pool per version:                                   │
  │                                                              │
  │  php7.4-fpm → /run/php/php7.4-fpm.sock → legacy.example.com  │
  │  php8.2-fpm → /run/php/php8.2-fpm.sock → modern.example.com  │
  └──────────────────────────────────────────────────────────────┘
```

```apache
# VHost for PHP 7.4
<VirtualHost *:80>
    ServerName legacy.example.com
    DocumentRoot /var/www/legacy
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php7.4-fpm.sock|fcgi://localhost"
    </FilesMatch>
</VirtualHost>

# VHost for PHP 8.2
<VirtualHost *:80>
    ServerName modern.example.com
    DocumentRoot /var/www/modern
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost"
    </FilesMatch>
</VirtualHost>
```

> This is why PHP-FPM is superior to mod_php — you can't run multiple PHP versions with mod_php.

---

### Q14: Apache in Docker — key gotchas?

```dockerfile
FROM httpd:2.4-alpine
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
COPY ./site/ /usr/local/apache2/htdocs/
EXPOSE 80

# Apache runs in foreground by default in official Docker image
# (uses -DFOREGROUND flag)
```

**Gotchas:**

1. **Log to stdout/stderr:**
   ```apache
   ErrorLog /proc/self/fd/2
   CustomLog /proc/self/fd/1 combined
   ```

2. **Official Docker image path:** `/usr/local/apache2/` not `/etc/httpd/` or `/etc/apache2/`!

3. **No systemctl:** Use `apachectl graceful` for reloads inside containers.

4. **ServerName warning:** Add `ServerName localhost` to suppress the `Could not reliably determine the server's fully qualified domain name` warning.

---

### Q15: When should I choose Apache over Nginx?

```
  Choose Apache when:
  ┌──────────────────────────────────────────────────────────────┐
  │ ✅ App requires .htaccess (WordPress, Laravel, Drupal)       │
  │ ✅ Need mod_rewrite with complex per-directory rules         │
  │ ✅ Running mod_php legacy apps (migration not feasible)      │
  │ ✅ Need mod_security WAF without buying a commercial WAF     │
  │ ✅ Shared hosting environment (per-user .htaccess)           │
  │ ✅ mod_wsgi for Python (tighter integration than proxy)      │
  └──────────────────────────────────────────────────────────────┘

  Choose Nginx when:
  ┌──────────────────────────────────────────────────────────────┐
  │ ✅ Reverse proxy / API gateway / load balancer               │
  │ ✅ Static file serving at scale                              │
  │ ✅ Kubernetes ingress controller                             │
  │ ✅ Memory-constrained environments (containers)              │
  │ ✅ High concurrency (10K+ connections)                       │
  │ ✅ Microservices architecture                                │
  └──────────────────────────────────────────────────────────────┘

  Use BOTH (common pattern):
  ┌──────────────────────────────────────────────────────────────┐
  │  Client → Nginx (SSL + static + LB) → Apache (mod_rewrite    │
  │                                        + .htaccess + mod_php)│
  └──────────────────────────────────────────────────────────────┘
```

---

> **End of Apache Guide** | [🏠 Home](../README.md) · [Servers](README.md) · [← Prev: Nginx](nginx.md) · [Next: IIS →](iis.md)
