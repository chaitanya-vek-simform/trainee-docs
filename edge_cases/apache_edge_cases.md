[🏠 Home](../README.md) · [Edge Cases](.)

# 🪶 Apache HTTP Server — Edge Cases (110)

> **Audience:** DevOps engineers, sysadmins, and SREs operating Apache httpd as a web server, reverse proxy, or application gateway in production.  
> **Coverage:** `.htaccess` traps, `mod_rewrite` nightmares, virtual host conflicts, SSL/TLS pitfalls, module loading, MPM tuning, CGI/WSGI failures, authentication edge cases, log management, and container deployments.

---

## Table of Contents

1. [.htaccess & Per-Directory Config (EC-001–015)](#htaccess--per-directory-config)
2. [mod_rewrite & URL Manipulation (EC-016–030)](#mod_rewrite--url-manipulation)
3. [Virtual Hosts & Server Configuration (EC-031–042)](#virtual-hosts--server-configuration)
4. [SSL / TLS & Certificates (EC-043–055)](#ssl--tls--certificates)
5. [Modules & Extensions (EC-056–065)](#modules--extensions)
6. [MPM, Performance & Tuning (EC-066–078)](#mpm-performance--tuning)
7. [Reverse Proxy & mod_proxy (EC-079–090)](#reverse-proxy--mod_proxy)
8. [Authentication & Access Control (EC-091–098)](#authentication--access-control)
9. [Logging, Monitoring & Troubleshooting (EC-099–105)](#logging-monitoring--troubleshooting)
10. [Containers, CI/CD & Cloud (EC-106–110)](#containers-cicd--cloud)
11. [Production FAQ](#production-faq)

---

## .htaccess & Per-Directory Config

### EC-001 — `.htaccess` Silently Ignored — `AllowOverride None` in Parent Config
**Category:** .htaccess | **Severity:** High | **Env:** Both

**Scenario:** Developer adds rewrite rules in `.htaccess`. Nothing happens — no error, no rewrite. The main `httpd.conf` or virtual host has `AllowOverride None`, which disables all `.htaccess` processing.

```
  AllowOverride chain:
  
  httpd.conf:
  <Directory "/var/www/html">
      AllowOverride None       ← disables ALL .htaccess directives
  </Directory>
  
  /var/www/html/.htaccess:
  RewriteEngine On             ← completely ignored!
  RewriteRule ^old$ /new [R=301,L]
  
  ┌──────────────────────────────────────────────────────────┐
  │ AllowOverride None  → .htaccess fully ignored            │
  │ AllowOverride All   → all .htaccess directives active    │
  │ AllowOverride FileInfo AuthConfig → selective enable     │
  └──────────────────────────────────────────────────────────┘
```

**Detection:**
```bash
# Check effective AllowOverride for a path:
apachectl -S   # show virtual host configuration
grep -rn "AllowOverride" /etc/apache2/ /etc/httpd/
# Enable rewrite log for debugging:
LogLevel alert rewrite:trace3
```

**Fix:**
```apache
<Directory "/var/www/html">
    AllowOverride All
    Require all granted
</Directory>
```

> ⚠️ **DevOps Gotcha:** `AllowOverride All` enables .htaccess but costs performance — Apache reads .htaccess from EVERY directory in the path on EVERY request. For production, put rules in the virtual host config instead of .htaccess whenever possible. `.htaccess` is a developer convenience, not a production best practice.

---

### EC-002 — `.htaccess` in Parent Directory Overridden by Child `.htaccess`
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** `.htaccess` in `/var/www/html/` sets `RewriteEngine On` and redirect rules. `.htaccess` in `/var/www/html/subdir/` exists but doesn't include `RewriteEngine On`. Rewrites stop working in the subdirectory.

```
  /var/www/html/.htaccess:
  RewriteEngine On
  RewriteRule ^old$ /new [R=301]      ← works here
  
  /var/www/html/subdir/.htaccess:
  # RewriteEngine not declared here
  Options -Indexes                     ← rewrites BROKEN in this dir!
  
  Each .htaccess is processed independently — child doesn't inherit
  RewriteEngine state from parent.
```

> ⚠️ **DevOps Gotcha:** `.htaccess` in a child directory creates a new scope. If the child file exists at all, it doesn't automatically inherit `RewriteEngine On` from the parent. You must redeclare it.

---

### EC-003 — `.htaccess` Parse Error Returns 500 for Entire Site
**Category:** .htaccess | **Severity:** Critical | **Env:** Prod

**Scenario:** Typo in `.htaccess` (e.g., `RewrteEngine On`). Apache can't parse it. EVERY request to the directory (and all subdirectories) returns HTTP 500 Internal Server Error.

**Detection:**
```bash
# Check error log:
tail -f /var/log/apache2/error.log
# Look for: ".htaccess: Invalid command 'RewrteEngine'"
```

**Fix:**
```bash
# Fix the typo, or temporarily disable .htaccess:
# In virtual host config:
AllowOverride None   # disables .htaccess processing entirely
apachectl graceful
```

> ⚠️ **DevOps Gotcha:** A single broken `.htaccess` file takes down the entire directory tree beneath it with 500 errors. In shared hosting, one user's broken `.htaccess` can't affect others (different `<Directory>` roots). But in a monolithic deployment, this is devastating. Always test `.htaccess` changes in staging first.

---

### EC-004 — `.htaccess` Performance Penalty — Stat Syscall Per Directory Level
**Category:** .htaccess | **Severity:** Medium | **Env:** Prod

**Scenario:** Request for `/var/www/html/app/assets/img/logo.png`. Apache checks for `.htaccess` in: `/var/www/html/`, `/var/www/html/app/`, `/var/www/html/app/assets/`, `/var/www/html/app/assets/img/`. That's 4 `stat()` syscalls even if no `.htaccess` exists.

```
  Request: /app/assets/img/logo.png
  
  Apache checks (with AllowOverride All):
  1. stat("/var/www/html/.htaccess")           ← file I/O
  2. stat("/var/www/html/app/.htaccess")       ← file I/O
  3. stat("/var/www/html/app/assets/.htaccess") ← file I/O
  4. stat("/var/www/html/app/assets/img/.htaccess") ← file I/O
  
  × 1000 requests/second = 4000 unnecessary syscalls/second
```

**Fix:**
```apache
# Move rules into vhost config and disable .htaccess:
<Directory "/var/www/html">
    AllowOverride None
    # Put your rewrite rules HERE instead of .htaccess
</Directory>
```

---

### EC-005 — `.htaccess` Can't Use `<VirtualHost>` or Server-Level Directives
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** Developer puts `<VirtualHost>`, `Listen`, `ServerName`, or `LoadModule` in `.htaccess`. Apache returns 500 — these directives are only valid in server config context, not directory context.

**Detection:**
```bash
# Error log shows:
# ".htaccess: <VirtualHost> not allowed here"
```

---

### EC-006 — `Options` Directive in `.htaccess` Requires `AllowOverride Options`
**Category:** .htaccess | **Severity:** Low | **Env:** Both

**Scenario:** `.htaccess` uses `Options +Indexes` or `Options -Indexes`. But `AllowOverride FileInfo` doesn't cover `Options` — needs `AllowOverride Options` or `AllowOverride All`.

---

### EC-007 — `.htpasswd` File Readable via Web — Credentials Exposed
**Category:** .htaccess | **Severity:** Critical | **Env:** Prod

**Scenario:** `.htpasswd` placed in web root. Browser request to `https://example.com/.htpasswd` downloads the file with hashed credentials. Weak hashes crackable offline.

**Fix:**
```apache
# Block access to dot-files:
<FilesMatch "^\.ht">
    Require all denied
</FilesMatch>
# Or store .htpasswd outside the web root:
AuthUserFile /etc/apache2/.htpasswd
```

> ⚠️ **DevOps Gotcha:** Apache blocks `.htaccess` and `.htpasswd` by default in modern configs, but custom or legacy configurations may not. Always verify with `curl https://example.com/.htpasswd` — if you get a download, it's exposed.

---

### EC-008 — `ErrorDocument` in `.htaccess` Causes Redirect Loop
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** `ErrorDocument 404 /error.php`. But `error.php` doesn't exist or itself triggers a 404. Apache enters a redirect loop and eventually returns a plain-text "Internal Server Error."

---

### EC-009 — PHP Directives in `.htaccess` Fail When Using PHP-FPM
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** `.htaccess` has `php_value upload_max_filesize 50M`. Works with mod_php. Fails with PHP-FPM (proxy/FastCGI) — `php_value` is a mod_php-only directive. Apache returns 500.

**Fix:**
```ini
# For PHP-FPM, use .user.ini instead of .htaccess:
# /var/www/html/.user.ini:
upload_max_filesize = 50M
post_max_size = 55M
# Or configure in php-fpm pool: /etc/php/8.2/fpm/pool.d/www.conf
```

---

### EC-010 — `RewriteBase` Missing in Subdirectory — Rewrites Go to Wrong Path
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** Application deployed in subdirectory `/app/`. `.htaccess` rewrite rules assume root deployment. Without `RewriteBase /app/`, rewrites resolve to wrong paths.

**Fix:**
```apache
# In /var/www/html/app/.htaccess:
RewriteEngine On
RewriteBase /app/
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /app/index.php [L]
```

---

### EC-011 — `.htaccess` `Require` Syntax Changed Between Apache 2.2 and 2.4
**Category:** .htaccess | **Severity:** High | **Env:** Both

**Scenario:** Migrating from Apache 2.2 to 2.4. Old `.htaccess` uses `Order deny,allow` and `Allow from`. Apache 2.4 ignores these (mod_access_compat may not be loaded). Access controls silently fail.

```
  Apache 2.2 syntax:              Apache 2.4 syntax:
  Order deny,allow                Require all granted
  Allow from all                  
  
  Order allow,deny                Require all denied
  Deny from all                   
  
  Allow from 10.0.0.0/8           Require ip 10.0.0.0/8
```

> ⚠️ **DevOps Gotcha:** Apache 2.4 can load `mod_access_compat` for backward compatibility, but mixing old and new syntax in the same `<Directory>` block causes unpredictable behavior. Migrate fully to 2.4 syntax.

---

### EC-012 — `Header` Directive in `.htaccess` Requires `mod_headers`
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** `.htaccess` sets `Header set X-Frame-Options "DENY"`. Module `mod_headers` not loaded. Apache returns 500 Internal Server Error.

**Fix:**
```bash
a2enmod headers
systemctl restart apache2
```

---

### EC-013 — `.htaccess` With BOM (Byte Order Mark) Causes Parse Error
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** `.htaccess` edited in Windows Notepad with UTF-8 BOM. BOM bytes (`EF BB BF`) prepended to file. Apache tries to parse BOM as a directive name → 500 error.

**Detection:**
```bash
file .htaccess         # check encoding
hexdump -C .htaccess | head -1   # look for EF BB BF at start
```

**Fix:**
```bash
sed -i '1s/^\xEF\xBB\xBF//' .htaccess
```

---

### EC-014 — Nested `.htaccess` Merging Causes Unexpected Directive Stacking
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** Parent `.htaccess` sets `Header append X-Frame-Options "DENY"`. Child `.htaccess` also sets `Header append X-Frame-Options "SAMEORIGIN"`. Both directives execute — response has TWO X-Frame-Options headers (browser behavior undefined).

**Fix:**
```apache
# In child .htaccess, use "set" to overwrite instead of "append":
Header set X-Frame-Options "SAMEORIGIN"
```

---

### EC-015 — `FallbackResource` Conflicts With `mod_rewrite` in `.htaccess`
**Category:** .htaccess | **Severity:** Medium | **Env:** Both

**Scenario:** Using `FallbackResource /index.php` (cleaner than rewrite rules) AND `RewriteRule` in the same `.htaccess`. Both process — rewrite may override FallbackResource or vice versa, causing unexpected routing.

> ⚠️ **DevOps Gotcha:** Use ONE approach: either `FallbackResource` OR `mod_rewrite`. Never both. `FallbackResource` is simpler and more efficient for single-entrypoint apps (Laravel, Symfony, WordPress).

---

## mod_rewrite & URL Manipulation

### EC-016 — RewriteRule Regex Matches More Than Expected
**Category:** mod_rewrite | **Severity:** High | **Env:** Both

**Scenario:** `RewriteRule ^page$ /new-page [R=301,L]` intended to match only `/page`. But without anchoring the full request, it may also match query strings or encoded variants depending on context.

```
  RewriteRule pattern matching scope:
  
  In .htaccess:    pattern matches RELATIVE path (no leading /)
  In vhost config: pattern matches FULL URI path (with leading /)
  
  .htaccess:   RewriteRule ^page$ ...    ← matches "page" (no slash)
  vhost:       RewriteRule ^/page$ ...   ← matches "/page" (with slash)
  
  Common mistake: copying rules between .htaccess and vhost without
  adjusting the leading slash.
```

---

### EC-017 — Infinite Rewrite Loop — Missing `[L]` Flag or Exit Condition
**Category:** mod_rewrite | **Severity:** Critical | **Env:** Both

**Scenario:** `RewriteRule ^(.*)$ /index.php?path=$1`. Every request (including `/index.php`) matches. Rewrites `/index.php` to `/index.php?path=index.php`, which matches again → infinite loop → 500 error.

```
  Loop detection:
  Request: /about
  1. RewriteRule → /index.php?path=about
  2. Internal redirect → /index.php matches again!
  3. RewriteRule → /index.php?path=index.php
  4. Internal redirect → loop continues...
  5. Apache detects loop after ~10 iterations → 500
```

**Fix:**
```apache
# Add condition to skip existing files:
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ /index.php?path=$1 [L,QSA]
# [L] = last rule (stop processing), [QSA] = append query string
```

> ⚠️ **DevOps Gotcha:** The `[L]` flag stops rule processing for the CURRENT pass. But the rewritten URL goes through the entire rule set AGAIN (internal redirect). Use `RewriteCond` to prevent matching on the rewritten URL.

---

### EC-018 — `[R]` Without Status Code Defaults to 302 — Bad for SEO
**Category:** mod_rewrite | **Severity:** Medium | **Env:** Prod

**Scenario:** `RewriteRule ^old-page$ /new-page [R,L]` — `[R]` alone defaults to 302 (temporary redirect). Search engines don't transfer page rank. Old URL stays in index.

**Fix:**
```apache
RewriteRule ^old-page$ /new-page [R=301,L]   # permanent redirect
```

---

### EC-019 — Query String Not Cleared After Rewrite — Old Params Leak Through
**Category:** mod_rewrite | **Severity:** Medium | **Env:** Both

**Scenario:** `RewriteRule ^search$ /find?engine=google [L]`. Original request: `/search?q=test`. Result: `/find?engine=google&q=test` — old query string APPENDED by default.

**Fix:**
```apache
# Add ? at end to clear query string:
RewriteRule ^search$ /find?engine=google? [L]
# Or use [QSD] flag (Apache 2.4+):
RewriteRule ^search$ /find?engine=google [L,QSD]
```

---

### EC-020 — `RewriteCond %{HTTPS} on` Doesn't Work Behind Load Balancer
**Category:** mod_rewrite | **Severity:** High | **Env:** Prod

**Scenario:** TLS terminated at load balancer. Apache receives plain HTTP. `%{HTTPS}` is always "off" even though the client uses HTTPS. HTTP→HTTPS redirect loop.

```
  Client → HTTPS → [ALB] → HTTP → Apache
  
  Apache sees: %{HTTPS} = "off"  (always!)
  Apache sends: 301 → https://...
  Client follows → HTTPS → ALB → HTTP → Apache
  Apache sees: %{HTTPS} = "off"  → 301 again → LOOP!
```

**Fix:**
```apache
# Check X-Forwarded-Proto instead:
RewriteCond %{HTTP:X-Forwarded-Proto} !https
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
```

---

### EC-021 — `mod_rewrite` `[P]` Flag Silently Requires `mod_proxy`
**Category:** mod_rewrite | **Severity:** High | **Env:** Both

**Scenario:** `RewriteRule ^/api/(.*)$ http://backend:8080/$1 [P,L]` — the `[P]` (proxy) flag requires `mod_proxy` and `mod_proxy_http` to be loaded. If they're not, Apache returns 500 with a cryptic log message.

**Fix:**
```bash
a2enmod proxy proxy_http
systemctl restart apache2
```

---

### EC-022 — `RewriteMap` in `.htaccess` Not Allowed — Server Config Only
**Category:** mod_rewrite | **Severity:** Medium | **Env:** Both

**Scenario:** Developer tries to use `RewriteMap` in `.htaccess` for URL lookups. `RewriteMap` is only valid in server/vhost context, not per-directory. Apache returns 500.

---

### EC-023 — URL Encoding in RewriteRule — `%20` vs Space Confusion
**Category:** mod_rewrite | **Severity:** Medium | **Env:** Both

**Scenario:** URL `/my%20page` — mod_rewrite receives the DECODED path (`my page` with a space). Rules matching `%20` literally never match. Rules must match the decoded form.

```
  URL:              /my%20page
  mod_rewrite sees: my page   (decoded!)
  
  RewriteRule ^my%20page$ ...  ← WON'T match
  RewriteRule "^my page$" ...  ← WILL match (quoted for space)
  
  Use [B] flag to re-encode backreferences:
  RewriteRule ^(.*)$ /handler?path=$1 [B,L]
```

---

### EC-024 — `mod_rewrite` Runs Twice — Once in Server Context, Once in Directory
**Category:** mod_rewrite | **Severity:** Medium | **Env:** Both

**Scenario:** RewriteRules in both virtual host config and `.htaccess` for the same path. Rules execute TWICE — first the virtual host pass, then the directory (`.htaccess`) pass. Unexpected double rewrites.

---

### EC-025 — `RewriteRule` Backreference `$1` Empty When Pattern Has No Capture Group
**Category:** mod_rewrite | **Severity:** Low | **Env:** Both

**Scenario:** `RewriteRule ^old-path /new-path/$1 [L]` — no capture group `()` in pattern. `$1` is empty. URL becomes `/new-path/` with trailing slash but no captured content.

---

### EC-026 — `mod_alias` `Redirect` and `mod_rewrite` Conflict
**Category:** mod_rewrite | **Severity:** Medium | **Env:** Both

**Scenario:** Both `Redirect 301 /old /new` (mod_alias) and `RewriteRule ^old$ /new [R=301,L]` (mod_rewrite) defined. mod_rewrite runs first (despite appearing later in config). Both may fire depending on processing phase, causing double redirects.

> ⚠️ **DevOps Gotcha:** Don't mix `mod_alias` (`Redirect`, `RedirectMatch`, `Alias`, `ScriptAlias`) with `mod_rewrite` for the same URL paths. Pick one module and use it consistently. `mod_rewrite` is more powerful but complex; `mod_alias` is simpler for basic redirects.

---

### EC-027 — HTTPS Redirect Strips POST Body — Data Loss
**Category:** mod_rewrite | **Severity:** High | **Env:** Prod

**Scenario:** User submits form over HTTP. Rewrite rule redirects to HTTPS with `[R=301]`. Browser follows redirect with GET (not POST) — form data lost. Standard says 301 redirect changes POST to GET.

**Fix:**
```apache
# Use 307 (temporary) or 308 (permanent) to preserve method:
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=308,L]
# 308 = permanent redirect that preserves HTTP method (POST stays POST)
```

---

### EC-028 — `ProxyPassMatch` With `RewriteRule [P]` — Double Proxy Processing
**Category:** mod_rewrite | **Severity:** Medium | **Env:** Both

**Scenario:** Both `ProxyPassMatch` and `RewriteRule [P]` configured for overlapping paths. Request gets proxied twice or conflicts arise. Confusing debug behavior.

---

### EC-029 — `RewriteRule` `[END]` Flag (Apache 2.4) Prevents Further Processing
**Category:** mod_rewrite | **Severity:** Low | **Env:** Both

**Scenario:** `[END]` flag introduced in Apache 2.4 — stops ALL rewrite processing (unlike `[L]` which only stops the current pass). Developers unaware of `[END]` use complex `RewriteCond` chains to prevent re-processing when `[END]` would suffice.

```
  [L] flag:  stops current rule-set pass, URL goes through another pass
  [END] flag: stops ALL rewrite processing permanently (Apache 2.4+)
```

---

### EC-030 — `mod_rewrite` Log Level Changed in Apache 2.4 — Old Debug Method Broken
**Category:** mod_rewrite | **Severity:** Low | **Env:** Both

**Scenario:** Apache 2.2 used `RewriteLog` and `RewriteLogLevel`. Apache 2.4 removed both. Developers can't debug rewrite rules.

**Fix:**
```apache
# Apache 2.4+ rewrite debugging:
LogLevel alert rewrite:trace8
# Logs go to the regular error log
# trace1-trace8: increasing verbosity (trace8 = maximum)
```

---

## Virtual Hosts & Server Configuration

### EC-031 — First Virtual Host Becomes Default Catch-All
**Category:** VHost | **Severity:** High | **Env:** Prod

**Scenario:** No explicit `_default_` virtual host. Request with unknown `Host` header hits the FIRST virtual host in config load order. Potentially exposes an internal application.

```
  Apache virtual host selection:
  
  1. Match by IP:port
  2. Within matching IP:port, match ServerName/ServerAlias
  3. If no match → use FIRST virtual host for that IP:port
  
  Without explicit default:
  Request Host: "evil.com" → first vhost in config → internal app!
```

**Fix:**
```apache
# Create explicit catch-all as the FIRST vhost:
<VirtualHost *:80>
    ServerName _default_
    DocumentRoot /var/www/default
    <Location />
        Require all denied
    </Location>
</VirtualHost>
```

> ⚠️ **DevOps Gotcha:** Always define a catch-all default virtual host that returns 403 or 444 equivalent. This prevents IP-based or spoofed Host header access to unintended services. Same principle as Nginx `default_server`.

---

### EC-032 — Name-Based Virtual Hosts on HTTPS — SNI Required
**Category:** VHost | **Severity:** Medium | **Env:** Prod

**Scenario:** Multiple name-based virtual hosts on port 443 with different certificates. Old clients without SNI support get the first (default) virtual host's certificate. Certificate mismatch for other domains.

---

### EC-033 — `ServerAlias` Wildcard `*.example.com` Conflicts With Specific Subdomain VHost
**Category:** VHost | **Severity:** Medium | **Env:** Prod

**Scenario:** VHost A: `ServerAlias *.example.com`. VHost B: `ServerName api.example.com`. Apache should match the most specific (`api.example.com`) first, but config load order can cause VHost A to match instead.

**Fix:**
```apache
# Ensure specific vhost loads BEFORE wildcard:
# Name the files: 001-api.conf, 002-wildcard.conf
# Or use IncludeOptional with explicit ordering
```

---

### EC-034 — `DocumentRoot` Points to Non-Existent Directory — 403 Not 404
**Category:** VHost | **Severity:** Medium | **Env:** Both

**Scenario:** `DocumentRoot /var/www/myapp` but directory doesn't exist. Apache returns 403 Forbidden (not 404). Confusing error message — looks like a permissions problem.

---

### EC-035 — `ServerName` Not Set — Apache Uses OS Hostname
**Category:** VHost | **Severity:** Low | **Env:** Both

**Scenario:** `ServerName` omitted in main config. Apache logs: "Could not reliably determine the server's fully qualified domain name." Uses OS hostname which may be wrong. Self-referential URLs use wrong hostname.

**Fix:**
```apache
# In main httpd.conf:
ServerName www.example.com:80
```

---

### EC-036 — Port-Based Virtual Host Ignores `ServerName`
**Category:** VHost | **Severity:** Medium | **Env:** Both

**Scenario:** Two virtual hosts: `<VirtualHost *:8080>` and `<VirtualHost *:8081>`. Both have `ServerName` set. Request to port 8080 with wrong Host header still routes to the 8080 vhost — port matching takes precedence.

---

### EC-037 — `Listen` Directive Missing for New Port — Apache Won't Bind
**Category:** VHost | **Severity:** High | **Env:** Both

**Scenario:** New virtual host added on port 8443. `Listen 8443` not declared. Apache can't bind to the port — virtual host silently doesn't work (no error in some cases, startup error in others depending on config).

---

### EC-038 — Include Order of Config Files Changes Virtual Host Priority
**Category:** VHost | **Severity:** Medium | **Env:** Prod

**Scenario:** `IncludeOptional sites-enabled/*.conf` loads alphabetically. Renaming `api.conf` to `z-api.conf` changes its load order. If it was the first vhost (default), it's no longer default.

---

### EC-039 — `<Directory>` Block Doesn't Match Symlinked Paths
**Category:** VHost | **Severity:** Medium | **Env:** Both

**Scenario:** DocumentRoot is `/var/www/current` (symlink to `/var/www/releases/v2.0`). `<Directory /var/www/current>` block works. But `<Directory /var/www/releases/v2.0>` doesn't apply to requests through the symlink unless `Options +FollowSymLinks` is set.

---

### EC-040 — `<Location>` vs `<Directory>` vs `<Files>` — Processing Order Confusion
**Category:** VHost | **Severity:** Medium | **Env:** Both

**Scenario:** Access control set in `<Directory>` block. Different rule in `<Location>` block for same path. `<Location>` always wins because it processes LAST.

```
  Apache configuration section processing order:
  
  1. <Directory> (and .htaccess) — filesystem path based
  2. <DirectoryMatch>
  3. <Files> and <FilesMatch>
  4. <Location> and <LocationMatch>  ← ALWAYS LAST (wins conflicts!)
  
  ┌──────────────────────────────────────────────────────────┐
  │ <Directory> restricts by file system path                │
  │ <Location> restricts by URL path                         │
  │ <Files> restricts by filename pattern                    │
  │                                                          │
  │ <Location> overrides <Directory> if both match!          │
  └──────────────────────────────────────────────────────────┘
```

---

### EC-041 — `Alias` Path Doesn't Work With `<Directory>` Block
**Category:** VHost | **Severity:** Medium | **Env:** Both

**Scenario:** `Alias /static /opt/assets`. `<Directory /var/www/html>` block has `Require all granted`. But `/opt/assets` is NOT under `/var/www/html` — access denied because no `<Directory /opt/assets>` rule.

**Fix:**
```apache
Alias /static /opt/assets
<Directory /opt/assets>
    Require all granted
    Options -Indexes
</Directory>
```

---

### EC-042 — `UseCanonicalName On` Breaks Behind Reverse Proxy
**Category:** VHost | **Severity:** Medium | **Env:** Prod

**Scenario:** `UseCanonicalName On` — Apache uses `ServerName` directive for self-referential URLs. Behind a reverse proxy with a different external hostname, redirects and generated URLs point to the wrong host.

---

## SSL / TLS & Certificates

### EC-043 — SSL Certificate and Key Mismatch — Apache Won't Start
**Category:** TLS | **Severity:** Critical | **Env:** Prod

**Scenario:** `SSLCertificateFile` and `SSLCertificateKeyFile` don't match (renewed cert with old key, or files from different domains). Apache refuses to start: "certificate and private key do not match."

**Detection:**
```bash
# Verify cert and key match:
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa -noout -modulus -in key.pem | openssl md5
# Both MD5 hashes MUST match
```

---

### EC-044 — Chain Certificate Missing — `SSLCertificateChainFile` Deprecated
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Apache 2.4.8+ deprecated `SSLCertificateChainFile`. Intermediate certs must be concatenated into `SSLCertificateFile`. Old config using deprecated directive — intermediates not loaded, API clients fail.

**Fix:**
```bash
# Concatenate cert + chain:
cat server.crt intermediate.crt > fullchain.crt
```
```apache
SSLCertificateFile /etc/ssl/fullchain.crt
SSLCertificateKeyFile /etc/ssl/server.key
# DO NOT use SSLCertificateChainFile (deprecated in 2.4.8+)
```

---

### EC-045 — SSL Passphrase on Key — Apache Hangs on Startup Waiting for Input
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Private key encrypted with passphrase. Apache prompts for passphrase on startup. Automated startup (systemd) hangs indefinitely waiting for interactive input.

**Fix:**
```bash
# Remove passphrase from key:
openssl rsa -in encrypted-key.pem -out decrypted-key.pem
chmod 600 decrypted-key.pem
# Or use SSLPassPhraseDialog directive with a script:
SSLPassPhraseDialog exec:/usr/local/bin/ssl-passphrase.sh
```

> ⚠️ **DevOps Gotcha:** In production, NEVER use passphrase-protected keys. Automated restarts, systemd, and container orchestration can't provide interactive input. Use unencrypted keys with strict file permissions (600, owned by root).

---

### EC-046 — `SSLProtocol` All Minus Old Versions — Syntax Trap
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1` — the `all` keyword includes ALL protocols compiled in. On older OpenSSL, `all` might include SSLv2. On newer builds, `all` might not include TLSv1.3.

**Fix:**
```apache
# Explicitly state what you want:
SSLProtocol TLSv1.2 TLSv1.3
# Don't rely on "all" minus exclusions
```

---

### EC-047 — HSTS Header Not Set on Error Pages — Bypass via Error Response
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `Header always set Strict-Transport-Security "max-age=31536000"` — the `always` keyword is REQUIRED. Without it, the header is only set on 2xx responses. Error pages (404, 500) don't include HSTS, allowing a downgrade attack window.

```apache
# Without "always": header only on success responses
Header set Strict-Transport-Security "max-age=31536000"   # ← only 2xx!

# With "always": header on ALL responses including errors
Header always set Strict-Transport-Security "max-age=31536000"   # ← all responses ✅
```

---

### EC-048 — Client Certificate Verification With `SSLVerifyClient optional` Leaks Info
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `SSLVerifyClient optional` — Apache requests client cert but doesn't require it. If the client cert is self-signed or invalid, Apache still accepts the connection. Must check `SSL_CLIENT_VERIFY` in application code.

---

### EC-049 — OCSP Stapling Configuration Requires `SSLStaplingCache`
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `SSLUseStapling on` configured but `SSLStaplingCache` not set. Apache logs error and OCSP stapling doesn't work. Must be set in global context, not inside virtual host.

**Fix:**
```apache
# In global config (OUTSIDE virtual host):
SSLStaplingCache shmcb:/var/run/ocsp(128000)
# Inside virtual host:
SSLUseStapling on
```

---

### EC-050 — SSL Renegotiation in `<Location>` Block Breaks Large POST Requests
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** `SSLVerifyClient require` set inside `<Location /secure>` (not globally). Apache triggers TLS renegotiation when the URL matches. Large POST requests already in flight fail because renegotiation resets the connection.

```
  POST /secure (50MB upload):
  1. TLS handshake (no client cert required globally)
  2. Apache receives request, matches <Location /secure>
  3. SSLVerifyClient require → triggers TLS RENEGOTIATION
  4. Connection reset → 50MB upload lost!
```

> ⚠️ **DevOps Gotcha:** If you need client certificate auth for specific paths, set `SSLVerifyClient optional` globally and check `SSL_CLIENT_VERIFY` variable per-location. Avoids renegotiation.

---

### EC-051 — Multiple SSL Certificates on Same IP:Port Without SNI
**Category:** TLS | **Severity:** Low | **Env:** Prod

**Scenario:** Two virtual hosts on `*:443` with different certs. Without SNI, Apache uses the first virtual host's certificate for all connections. Modern browsers support SNI, but some legacy tools don't.

---

### EC-052 — `SSLCipherSuite` Too Restrictive — Older Clients Can't Connect
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Ultra-strict cipher suite (only TLS 1.3 ciphers). Legacy internal systems (Java 8, older curl) can't connect. Balance security with compatibility.

---

### EC-053 — HTTP/2 (`mod_http2`) Requires TLS — Fails on Plain HTTP
**Category:** TLS | **Severity:** Low | **Env:** Both

**Scenario:** `Protocols h2 http/1.1` set on port 80. HTTP/2 over plain HTTP (h2c) is not supported by browsers. Only works with TLS (h2). No error, just falls back to HTTP/1.1.

---

### EC-054 — Let's Encrypt Wildcard Cert Requires DNS-01 Challenge
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `certbot --apache -d *.example.com` — wildcard requires DNS-01 challenge. HTTP-01 (which `--apache` plugin uses) doesn't support wildcards. Certbot fails with unclear error.

**Fix:**
```bash
certbot certonly --manual --preferred-challenges dns -d "*.example.com" -d "example.com"
# Or use DNS plugin:
certbot certonly --dns-cloudflare -d "*.example.com" -d "example.com"
```

---

### EC-055 — `SSLSessionCache` Not Configured — Expensive Handshakes Repeat
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** No session cache configured. Each connection does a full TLS handshake. High latency and CPU usage under load.

**Fix:**
```apache
# Global config:
SSLSessionCache shmcb:/var/run/ssl_scache(512000)
SSLSessionCacheTimeout 300
```

---

## Modules & Extensions

### EC-056 — Module Loaded Twice — Apache Won't Start
**Category:** Modules | **Severity:** High | **Env:** Both

**Scenario:** `LoadModule rewrite_module modules/mod_rewrite.so` appears in both `httpd.conf` AND `conf.modules.d/00-base.conf`. Apache fails: "module rewrite_module is already loaded."

**Detection:**
```bash
grep -rn "LoadModule rewrite" /etc/httpd/ /etc/apache2/
apachectl -M | sort   # list loaded modules
```

---

### EC-057 — `mod_security` Blocks Legitimate Requests — False Positives
**Category:** Modules | **Severity:** High | **Env:** Prod

**Scenario:** ModSecurity with OWASP Core Rule Set (CRS) enabled. Blocks legitimate POST requests containing SQL-like keywords in body (e.g., form field explaining a database). Returns 403.

**Fix:**
```apache
# Whitelist specific rule for specific URL:
<LocationMatch "^/api/submit">
    SecRuleRemoveById 942100   # SQL injection detection rule
</LocationMatch>
# Or set anomaly threshold higher:
SecAction "id:900110,phase:1,nolog,pass,t:none,setvar:tx.inbound_anomaly_score_threshold=10"
```

---

### EC-058 — `mod_php` and `mod_fcgid` Conflict — Segfaults
**Category:** Modules | **Severity:** Critical | **Env:** Both

**Scenario:** Both `mod_php` and `mod_fcgid` (or `mod_proxy_fcgi`) loaded. Two different PHP execution methods competing. Apache segfaults on PHP requests or returns wrong handler.

**Fix:**
```bash
# Use ONLY one PHP handler:
a2dismod php8.2   # disable mod_php
# Keep mod_proxy_fcgi for PHP-FPM (recommended)
```

> ⚠️ **DevOps Gotcha:** `mod_php` embeds PHP in Apache worker processes (high memory, not thread-safe with worker/event MPM). PHP-FPM via `mod_proxy_fcgi` is the modern approach — separate process pool, works with all MPMs, better resource management.

---

### EC-059 — `mod_status` Exposed Publicly — Server Info Leaked
**Category:** Modules | **Severity:** Medium | **Env:** Prod

**Scenario:** `ExtendedStatus On` with `/server-status` accessible from any IP. Exposes: active connections, request details, client IPs, backend URLs, worker state. Reconnaissance goldmine.

**Fix:**
```apache
<Location /server-status>
    SetHandler server-status
    Require ip 10.0.0.0/8
    Require ip 127.0.0.1
</Location>
```

---

### EC-060 — `mod_deflate` Compresses Already-Compressed Content
**Category:** Modules | **Severity:** Low | **Env:** Prod

**Scenario:** `mod_deflate` configured to compress all responses. JPEG, PNG, GIF already compressed — re-compression wastes CPU with no size benefit (may even increase size).

**Fix:**
```apache
# Exclude binary/compressed formats:
SetOutputFilter DEFLATE
SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png|webp|mp4|zip|gz|bz2)$ no-gzip
# Or explicitly list types to compress:
AddOutputFilterByType DEFLATE text/html text/plain text/css application/json application/javascript
```

---

### EC-061 — `mod_expires` Headers Not Sent for Dynamic Content
**Category:** Modules | **Severity:** Medium | **Env:** Prod

**Scenario:** `mod_expires` configured for static assets. Dynamic PHP pages don't get `Expires` headers because PHP sets its own `Cache-Control: no-cache`. `mod_expires` respects existing headers.

---

### EC-062 — `mod_evasive` Rate Limiting Blocks Monitoring Probes
**Category:** Modules | **Severity:** Medium | **Env:** Prod

**Scenario:** `mod_evasive` configured with aggressive thresholds. Health check probe (hitting `/health` every 5 seconds) from load balancer exceeds page threshold. Load balancer IP banned → health check fails → all traffic removed.

**Fix:**
```apache
DOSWhitelist 10.0.0.*   # whitelist load balancer IP range
```

---

### EC-063 — `mod_wsgi` Daemon Mode vs Embedded Mode
**Category:** Modules | **Severity:** High | **Env:** Prod

**Scenario:** Python WSGI app running in embedded mode (default). Each Apache worker loads the full Python interpreter. Memory usage: 200MB × 150 workers = 30GB. Workers can't be recycled independently.

**Fix:**
```apache
# Use daemon mode:
WSGIDaemonProcess myapp processes=4 threads=16 python-home=/opt/venv
WSGIProcessGroup myapp
WSGIScriptAlias / /var/www/myapp/wsgi.py
# Separate Python processes — independent from Apache workers
```

---

### EC-064 — `mod_pagespeed` Rewrites Break JavaScript Frameworks
**Category:** Modules | **Severity:** Medium | **Env:** Prod

**Scenario:** `mod_pagespeed` automatically combines/minifies JavaScript and CSS. Breaks Angular, React, or Vue.js apps that depend on specific script loading order or dynamic imports.

---

### EC-065 — `mod_remoteip` Not Configured — Logs Show Load Balancer IP
**Category:** Modules | **Severity:** Medium | **Env:** Prod

**Scenario:** Apache behind load balancer. All access logs show the LB's IP address instead of the real client IP. Rate limiting and geo-blocking based on IP are useless.

**Fix:**
```apache
# Enable mod_remoteip:
RemoteIPHeader X-Forwarded-For
RemoteIPTrustedProxy 10.0.0.0/8   # trust LB IP range
# Now %a in LogFormat shows real client IP
```

---

## MPM, Performance & Tuning

### EC-066 — Prefork MPM With mod_php — Memory Explosion Under Load
**Category:** MPM | **Severity:** Critical | **Env:** Prod

**Scenario:** Prefork MPM (required by mod_php). Each child process handles ONE connection and loads full PHP interpreter. 150 max clients × 200MB per process = 30GB RAM. Traffic spike → OOM killer → Apache dies.

```
  MPM Comparison:
  
  ┌──────────────┬───────────┬───────────┬──────────────────────┐
  │ MPM          │ Processes │ Threads   │ Connections          │
  ├──────────────┼───────────┼───────────┼──────────────────────┤
  │ prefork      │ Many      │ 1/proc    │ 1 per process        │
  │ worker       │ Few       │ Many      │ 1 per thread         │
  │ event        │ Few       │ Many      │ keepalive offloaded   │
  └──────────────┴───────────┴───────────┴──────────────────────┘
  
  prefork + mod_php:  200MB × 150 = 30GB RAM ← dangerous!
  event + PHP-FPM:    10MB × 150 = 1.5GB + PHP-FPM pool separately
```

**Fix:**
```bash
# Switch to event MPM + PHP-FPM:
a2dismod php8.2 mpm_prefork
a2enmod mpm_event proxy_fcgi
a2enconf php8.2-fpm
systemctl restart apache2
```

> ⚠️ **DevOps Gotcha:** If you're running mod_php, you MUST use prefork MPM (mod_php is not thread-safe). The modern approach: event MPM + PHP-FPM via mod_proxy_fcgi. Better performance, lower memory, independent PHP process management.

---

### EC-067 — `MaxRequestWorkers` Too Low — Connection Refused Under Load
**Category:** MPM | **Severity:** Critical | **Env:** Prod

**Scenario:** Default `MaxRequestWorkers 150`. Each active connection holds one worker. With slow clients and keep-alive, workers exhausted. New connections queued or refused. Site appears down to new visitors while existing connections work fine.

**Detection:**
```bash
# Check if limit is hit:
apachectl status | grep "requests currently being processed"
# Or check server-status page:
curl http://localhost/server-status?auto
```

**Fix:**
```apache
# Event MPM tuning:
<IfModule mpm_event_module>
    ServerLimit          16
    StartServers          4
    MinSpareThreads      75
    MaxSpareThreads     250
    ThreadsPerChild      64
    MaxRequestWorkers  1024    # 16 × 64 = 1024 concurrent connections
    MaxConnectionsPerChild 10000
</IfModule>
```

---

### EC-068 — `KeepAliveTimeout` Too Long — Workers Held by Idle Connections
**Category:** MPM | **Severity:** High | **Env:** Prod

**Scenario:** `KeepAliveTimeout 60` (60 seconds). Each idle keep-alive connection holds a worker thread. Users open 5 tabs, each idle → 5 workers held per user. Workers exhausted by idle connections, not active requests.

**Fix:**
```apache
KeepAlive On
KeepAliveTimeout 5          # 5 seconds is usually sufficient
MaxKeepAliveRequests 100    # limit requests per connection
```

---

### EC-069 — `MaxConnectionsPerChild 0` — Memory Leaks Never Cleaned Up
**Category:** MPM | **Severity:** Medium | **Env:** Prod

**Scenario:** `MaxConnectionsPerChild 0` (unlimited). Worker processes/threads never recycled. Applications with memory leaks (PHP, mod_wsgi) accumulate memory indefinitely. Eventually OOM.

**Fix:**
```apache
MaxConnectionsPerChild 10000   # recycle after 10000 requests
```

---

### EC-070 — `Timeout` Directive Applies to Entire Request Lifecycle
**Category:** MPM | **Severity:** Medium | **Env:** Prod

**Scenario:** `Timeout 300` (5 minutes). This is the time Apache waits for: client to send request, DNS lookup, connection to backend, backend response, client to acknowledge response. One value for all phases — too coarse.

> ⚠️ **DevOps Gotcha:** Unlike Nginx's granular timeout settings, Apache's `Timeout` is a single value for the entire I/O lifecycle. For fine-grained control with proxy, use `ProxyTimeout` (separate from `Timeout`).

---

### EC-071 — `ServerLimit` Can't Be Changed Without Full Restart
**Category:** MPM | **Severity:** Medium | **Env:** Prod

**Scenario:** Increase `MaxRequestWorkers` beyond `ServerLimit × ThreadsPerChild`. Graceful restart doesn't increase `ServerLimit` — requires full stop/start. In production, this means brief downtime.

```
  MaxRequestWorkers = ServerLimit × ThreadsPerChild
  
  If ServerLimit = 16, ThreadsPerChild = 25:
  MaxRequestWorkers max = 16 × 25 = 400
  
  To increase beyond 400 → must increase ServerLimit → full restart required
```

---

### EC-072 — `EnableSendfile` Causes Stale Content With NFS/CIFS
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** `EnableSendfile On` (default). Files served from NFS/CIFS mount. Sendfile on network filesystems can serve stale data from kernel cache. File updated on NFS server but Apache serves old version.

**Fix:**
```apache
# Disable sendfile for network filesystems:
EnableSendfile Off
# Or only in specific directories:
<Directory /mnt/nfs/www>
    EnableSendfile Off
</Directory>
```

---

### EC-073 — `HostnameLookups On` — Massive Latency Per Request
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** `HostnameLookups On` triggers reverse DNS lookup for every client connection. DNS lookup adds 50-500ms per request. Under load, DNS servers overwhelmed.

**Fix:**
```apache
HostnameLookups Off   # ALWAYS off in production
# If you need hostnames in logs, do post-processing:
# logresolve < access.log > resolved.log
```

---

### EC-074 — `AllowOverride All` in High-Traffic Static File Server
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** CDN origin serves static assets. `AllowOverride All` causes filesystem stat calls for `.htaccess` on every request across every directory level. Thousands of unnecessary I/O operations per second.

---

### EC-075 — Apache Fails to Start After `ulimit` Change Not Applied
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** Increase `MaxRequestWorkers` to 4096. Apache hits OS file descriptor limit. "Too many open files" in error log. Each connection needs ~2 file descriptors.

**Fix:**
```bash
# In /etc/security/limits.conf:
www-data soft nofile 65536
www-data hard nofile 65536
# Or in systemd unit:
# /etc/systemd/system/apache2.service.d/limits.conf:
[Service]
LimitNOFILE=65536
```

---

### EC-076 — Event MPM `AsyncRequestWorkerFactor` Misconfigured
**Category:** MPM | **Severity:** Medium | **Env:** Prod

**Scenario:** Event MPM uses `AsyncRequestWorkerFactor` to handle idle keepalive connections beyond `MaxRequestWorkers`. Default factor `2` means it can handle `MaxRequestWorkers × 2` idle connections. If not understood, connection limits are confusing.

```
  Event MPM connection math:
  MaxRequestWorkers = 400
  AsyncRequestWorkerFactor = 2
  
  Active request slots: 400
  Total connections (active + idle keepalive): up to 400 × 2 = 800
```

---

### EC-077 — CGI Script Timeout — `mod_cgi` Default Is the Global `Timeout`
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** CGI script takes 120 seconds. Global `Timeout 60`. Apache kills CGI process after 60 seconds. No separate CGI timeout directive in base Apache (unlike `ProxyTimeout` for mod_proxy).

---

### EC-078 — Apache Graceful Restart Leaves Old Workers Running for Hours
**Category:** MPM | **Severity:** Medium | **Env:** Prod

**Scenario:** `apachectl graceful` — old workers finish current requests before exiting. Long-running downloads or WebSocket connections keep old workers alive indefinitely. Old configuration persists in those workers.

**Fix:**
```apache
GracefulShutdownTimeout 60   # force-kill old workers after 60 seconds
```

---

## Reverse Proxy & mod_proxy

### EC-079 — `ProxyPass` Order Matters — Longest Path First
**Category:** Proxy | **Severity:** High | **Env:** Both

**Scenario:** `ProxyPass / http://default-backend/` declared before `ProxyPass /api http://api-backend/`. The `/` rule matches everything first — `/api` requests go to default-backend instead of api-backend.

```
  WRONG order (first match wins):
  ProxyPass /     http://default-backend/    ← matches /api too!
  ProxyPass /api  http://api-backend/        ← never reached for /api
  
  CORRECT order (most specific first):
  ProxyPass /api  http://api-backend/        ← matches /api first
  ProxyPass /     http://default-backend/    ← catches everything else
```

> ⚠️ **DevOps Gotcha:** `ProxyPass` rules are matched in CONFIG ORDER (first match wins). Always put the most specific (longest) paths first, catch-all (`/`) last. This is the opposite of Nginx's longest-prefix-match behavior.

---

### EC-080 — `ProxyPass` and `ProxyPassReverse` Path Mismatch
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** `ProxyPass /app http://backend:8080/`. `ProxyPassReverse` not set or set incorrectly. Backend sends `Location: http://backend:8080/login` redirect. Client receives internal backend URL — can't access it.

**Fix:**
```apache
ProxyPass /app http://backend:8080/
ProxyPassReverse /app http://backend:8080/
# ProxyPassReverse rewrites Location headers from backend to external-facing URLs
```

---

### EC-081 — Backend Returns 503 — Apache Shows Default Error Page
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** Backend returns 503 with a custom JSON error body. `ProxyErrorOverride On` configured — Apache replaces the backend's 503 response with its own default HTML error page. API clients expect JSON, get HTML.

**Fix:**
```apache
# Don't override error responses if backend sends proper content:
ProxyErrorOverride Off
# Or selectively override only specific status codes:
ProxyErrorOverride On
ErrorDocument 503 /custom-503.json
```

---

### EC-082 — `ProxyTimeout` Not Set — Uses Global `Timeout` for Backend
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** Global `Timeout 60`. Slow backend API takes 120 seconds. `ProxyTimeout` not explicitly set. Apache uses `Timeout` value for proxy connections — backend response cut off at 60 seconds.

**Fix:**
```apache
ProxyTimeout 300   # 5 minutes for slow backend operations
# Or per-location:
<Location /api/reports>
    ProxyPass http://backend:8080/reports
    ProxyTimeout 600
</Location>
```

---

### EC-083 — `ProxyPreserveHost On` Not Set — Backend Receives Wrong Host Header
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** Request to `www.example.com`. Apache proxies to `http://backend:8080`. Backend receives `Host: backend:8080` instead of `Host: www.example.com`. Backend's virtual host routing breaks.

**Fix:**
```apache
ProxyPreserveHost On
ProxyPass / http://backend:8080/
ProxyPassReverse / http://backend:8080/
```

---

### EC-084 — WebSocket Proxy Fails — Missing `mod_proxy_wstunnel`
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** WebSocket requests through Apache reverse proxy fail. `mod_proxy_http` can't handle WebSocket upgrade. Need `mod_proxy_wstunnel`.

**Fix:**
```bash
a2enmod proxy_wstunnel
```
```apache
# WebSocket proxy:
RewriteEngine On
RewriteCond %{HTTP:Upgrade} websocket [NC]
RewriteCond %{HTTP:Connection} upgrade [NC]
RewriteRule ^/ws/(.*) ws://backend:8080/ws/$1 [P,L]
# Or direct proxy:
ProxyPass /ws/ ws://backend:8080/ws/
ProxyPassReverse /ws/ ws://backend:8080/ws/
```

---

### EC-085 — `ProxyPass` With `retry=0` — Retries Indefinitely After Backend Failure
**Category:** Proxy | **Severity:** High | **Env:** Prod

**Scenario:** Backend goes down. By default, Apache marks it as in error state and retries after `retry` seconds (default 60). During those 60 seconds, ALL requests to that backend return 503 immediately even if the backend has recovered.

**Fix:**
```apache
ProxyPass / http://backend:8080/ retry=5
# retry=5: check backend again after 5 seconds
# retry=0: check on every request (no error state delay)
```

---

### EC-086 — Load Balancer `byrequests` Doesn't Account for Response Time
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** `mod_proxy_balancer` with `lbmethod=byrequests` (round-robin). One backend is slow. Gets equal traffic but serves slowly → user experience degraded.

**Fix:**
```apache
<Proxy "balancer://cluster">
    BalancerMember http://backend1:8080
    BalancerMember http://backend2:8080
    ProxySet lbmethod=bytraffic    # or bybusyness for active request count
</Proxy>
```

---

### EC-087 — Sticky Sessions (Route) Lost After Backend Restart
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** `mod_proxy_balancer` with sticky sessions (route cookie). Backend restarts — session affinity cookie still points to it. But backend has lost session state (in-memory sessions). User effectively logged out.

> ⚠️ **DevOps Gotcha:** Sticky sessions are NOT a replacement for shared session storage (Redis, database). After backend restart, sessions are lost regardless of stickiness. Use shared session stores for resilience.

---

### EC-088 — `ProxyPass` to HTTPS Backend Without Certificate Verification
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** `ProxyPass / https://backend:8443/` — Apache connects to backend over TLS but doesn't verify the backend's certificate by default. MITM between Apache and backend goes undetected.

**Fix:**
```apache
SSLProxyEngine On
SSLProxyVerify require
SSLProxyCACertificateFile /etc/ssl/certs/backend-ca.pem
ProxyPass / https://backend:8443/
```

---

### EC-089 — Backend Health Check Not Available in Open Source Apache
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** Unlike commercial Apache Traffic Server, standard Apache `mod_proxy` has no active health checks. Relies on passive failure detection. Dead backends discovered only when real user requests fail.

**Fix:**
```apache
# Use mod_proxy_hcheck (Apache 2.4.21+):
<Proxy "balancer://cluster">
    BalancerMember http://backend1:8080 hcmethod=TCP hcinterval=5 hcpasses=2 hcfails=3
    BalancerMember http://backend2:8080 hcmethod=TCP hcinterval=5 hcpasses=2 hcfails=3
</Proxy>
# hcmethod: TCP, OPTIONS, HEAD, GET
# hcinterval: seconds between checks
```

---

### EC-090 — `mod_proxy_fcgi` Path Info Handling Breaks PHP Routing
**Category:** Proxy | **Severity:** Medium | **Env:** Both

**Scenario:** PHP framework expects `PATH_INFO` from URL. `mod_proxy_fcgi` doesn't set `PATH_INFO` correctly with `SetHandler` directive. Framework can't route requests.

**Fix:**
```apache
# Use ProxyPassMatch for proper PATH_INFO handling:
<FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost"
</FilesMatch>
# Or:
ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://127.0.0.1:9000/var/www/html/$1
```

---

## Authentication & Access Control

### EC-091 — `Require valid-user` Without Auth Module — Everyone Denied
**Category:** Auth | **Severity:** High | **Env:** Both

**Scenario:** `Require valid-user` set but no `AuthType`/`AuthBasicProvider` configured. Apache can't authenticate anyone → all requests denied (500 or 403 depending on configuration).

**Fix:**
```apache
<Location /admin>
    AuthType Basic
    AuthName "Admin Area"
    AuthBasicProvider file
    AuthUserFile /etc/apache2/.htpasswd
    Require valid-user
</Location>
```

---

### EC-092 — `<RequireAll>` vs `<RequireAny>` — Wrong Boolean Logic
**Category:** Auth | **Severity:** High | **Env:** Prod

**Scenario:** Intent: allow only authenticated users FROM trusted IP range. Using `<RequireAny>` instead of `<RequireAll>` — either condition alone grants access.

```
  <RequireAny>:   IP match OR valid-user → access granted
  <RequireAll>:   IP match AND valid-user → access granted
  
  ┌──────────────────────────────────────────────────┐
  │ <RequireAll>                                     │
  │   Require ip 10.0.0.0/8                          │
  │   Require valid-user                             │
  │ </RequireAll>                                    │
  │ → Both conditions must be true                   │
  │                                                  │
  │ <RequireAny>                                     │
  │   Require ip 10.0.0.0/8                          │
  │   Require valid-user                             │
  │ </RequireAny>                                    │
  │ → Either condition is sufficient                 │
  └──────────────────────────────────────────────────┘
```

---

### EC-093 — LDAP Authentication Fails Silently — `AuthLDAPURL` Syntax Wrong
**Category:** Auth | **Severity:** High | **Env:** Prod

**Scenario:** `AuthLDAPURL "ldap://ldap.company.com/ou=People,dc=company,dc=com"` — missing the search attribute. Apache can't look up users. Returns 500 with cryptic LDAP error in logs.

**Fix:**
```apache
# Include search attribute (uid) and scope:
AuthLDAPURL "ldap://ldap.company.com/ou=People,dc=company,dc=com?uid?sub?(objectClass=*)"
```

---

### EC-094 — `mod_auth_openidc` Token Refresh Race Condition
**Category:** Auth | **Severity:** Medium | **Env:** Prod

**Scenario:** OpenID Connect authentication. Token expires. Multiple simultaneous requests trigger concurrent token refresh. Race condition — some requests get 401 while token is being refreshed.

---

### EC-095 — Digest Auth Incompatible With Proxied Password Backends
**Category:** Auth | **Severity:** Medium | **Env:** Both

**Scenario:** `AuthType Digest` requires storing passwords in a specific format (MD5-hashed with realm). Can't use the same password file as Basic auth. Migrating between auth types requires regenerating all passwords.

---

### EC-096 — `Require` Directive Outside Auth Block Grants Unintended Access
**Category:** Auth | **Severity:** Critical | **Env:** Prod

**Scenario:** `Require all granted` in a `<Directory>` block overrides authentication requirements in a `<Location>` block. Because `<Location>` is processed after `<Directory>`, it should win — but `<Directory>` with `.htaccess` containing `Require all granted` can override.

---

### EC-097 — IP-Based Access Control Bypassed via `X-Forwarded-For` Spoofing
**Category:** Auth | **Severity:** High | **Env:** Prod

**Scenario:** `Require ip 10.0.0.0/8` for internal admin page. Attacker adds `X-Forwarded-For: 10.0.0.1` header. If `mod_remoteip` is configured to trust all `X-Forwarded-For`, spoofed IP bypasses restriction.

**Fix:**
```apache
# Only trust known proxy IPs:
RemoteIPHeader X-Forwarded-For
RemoteIPTrustedProxy 10.0.0.1   # only trust your LB
# Don't use RemoteIPInternalProxy with broad ranges
```

> ⚠️ **DevOps Gotcha:** NEVER trust `X-Forwarded-For` from untrusted sources for access control. Always validate the immediate source. `mod_remoteip` should only trust your own infrastructure's proxy IPs.

---

### EC-098 — Form-Based Auth With `mod_session` — Session Cookie Not Secure
**Category:** Auth | **Severity:** High | **Env:** Prod

**Scenario:** `mod_session_cookie` stores session data in a cookie. Cookie not marked `Secure` or `HttpOnly`. Session hijacking possible over HTTP or via JavaScript.

**Fix:**
```apache
Session On
SessionCookieName session path=/;httponly;secure;SameSite=Strict
SessionCryptoPassphrase secretpassword
```

---

## Logging, Monitoring & Troubleshooting

### EC-099 — `CustomLog` Uses `%h` — Logs Load Balancer IP Instead of Client
**Category:** Logging | **Severity:** High | **Env:** Prod

**Scenario:** Behind a load balancer. `%h` in `LogFormat` logs the remote hostname/IP — which is the LB's IP, not the client's. All log entries show the same IP.

**Fix:**
```apache
# With mod_remoteip configured:
LogFormat "%a %l %u %t \"%r\" %>s %b" combined
# %a = client IP (from mod_remoteip), %h = immediate peer (LB)
```

---

### EC-100 — Error Log Level Set to `debug` in Production — Disk Fills Fast
**Category:** Logging | **Severity:** High | **Env:** Prod

**Scenario:** Developer sets `LogLevel debug` for troubleshooting. Forgets to change back. Gigabytes of debug logs per hour. Disk fills. Apache can't write → cascading failures.

**Fix:**
```apache
# Use per-module log levels:
LogLevel warn rewrite:trace3 ssl:warn
# Targeted debugging without flooding all modules
```

---

### EC-101 — `rotatelogs` Pipe Breaks — Apache Stops Logging
**Category:** Logging | **Severity:** High | **Env:** Prod

**Scenario:** `CustomLog "|/usr/bin/rotatelogs ..." combined` — piped logging. If `rotatelogs` process crashes, pipe breaks. Apache either stops logging (silent) or crashes depending on configuration.

**Fix:**
```apache
# Use reliable piped logging — restart on failure:
CustomLog "||/usr/bin/rotatelogs /var/log/apache2/access.%Y%m%d.log 86400" combined
# Double pipe (||) means: restart the program if it fails
```

---

### EC-102 — `mod_status` `ExtendedStatus` Performance Impact
**Category:** Monitoring | **Severity:** Low | **Env:** Prod

**Scenario:** `ExtendedStatus On` enables detailed per-request tracking in server-status. Adds a small overhead per request (mutex operations for shared memory updates). On extremely high-throughput servers (100k+ req/s), this overhead is measurable.

---

### EC-103 — Forensic Log (`mod_log_forensic`) Not Flushed Before Crash
**Category:** Logging | **Severity:** Medium | **Env:** Prod

**Scenario:** `mod_log_forensic` logs request BEFORE processing (for crash forensics). But if Apache process crashes during request handling, the response log entry (with status code) is missing. Only the request entry exists.

---

### EC-104 — `ErrorLog` Per-VHost Not Working — All Errors Go to Main Log
**Category:** Logging | **Severity:** Medium | **Env:** Prod

**Scenario:** `ErrorLog /var/log/apache2/myapp-error.log` set inside `<VirtualHost>`. But some errors (startup errors, module errors) still go to the main `ErrorLog`. If main error log isn't monitored, critical errors are missed.

---

### EC-105 — Apache Status Shows "W" (Sending Reply) Workers Stuck
**Category:** Monitoring | **Severity:** High | **Env:** Prod

**Scenario:** `server-status` shows many workers in "W" state (sending reply to client). But clients aren't receiving data. Indicates: slow clients, proxy to dead backend, or blocked I/O. Workers held indefinitely.

**Detection:**
```bash
curl http://localhost/server-status?auto | grep -E "^BusyWorkers|^Scoreboard"
# "W" in scoreboard = sending reply, "R" = reading request
# Many "W" = potential backend or client issue
```

---

## Containers, CI/CD & Cloud

### EC-106 — Apache Docker Container Runs as Root by Default
**Category:** Container | **Severity:** High | **Env:** Prod

**Scenario:** Official `httpd` Docker image runs as root. Kubernetes Pod Security Standards reject it. Or a vulnerability in Apache gives attacker root access inside the container.

**Fix:**
```dockerfile
FROM httpd:2.4-alpine
# Create non-root user:
RUN sed -i 's/Listen 80/Listen 8080/' /usr/local/apache2/conf/httpd.conf
RUN chown -R www-data:www-data /usr/local/apache2/logs
USER www-data
EXPOSE 8080
```

---

### EC-107 — Apache PID File Lost in Container Restart — Graceful Reload Fails
**Category:** Container | **Severity:** Medium | **Env:** Prod

**Scenario:** `apachectl graceful` needs PID file. Container restarts lose the PID file (ephemeral filesystem). Graceful reload fails because Apache can't find the master process PID.

**Fix:**
```apache
# Use foreground mode in containers:
# CMD ["httpd-foreground"]   ← official image already does this
# For signals, use: docker kill --signal=USR1 <container>
```

---

### EC-108 — Config Test in CI Pipeline Uses Wrong MPM — Tests Pass, Prod Fails
**Category:** CI/CD | **Severity:** Medium | **Env:** Both

**Scenario:** CI runs `apachectl configtest`. CI image has prefork MPM compiled in, but production uses event MPM. Config using event-specific directives passes in CI (ignored for wrong MPM) but fails in prod.

**Fix:**
```bash
# In CI, use the SAME Apache build as production:
# Use identical Docker image for config testing:
docker run --rm -v $PWD/conf:/usr/local/apache2/conf httpd:2.4 httpd -t
```

---

### EC-109 — Healthcheck Returns 200 but Application Not Ready — Premature Traffic
**Category:** Cloud | **Severity:** High | **Env:** Prod

**Scenario:** Load balancer health checks Apache on `/`. Apache returns 200 (it's running). But PHP-FPM or backend application hasn't finished startup. Users get 503 from the application.

**Fix:**
```apache
# Proxy health check to backend application:
<Location /health>
    ProxyPass http://127.0.0.1:9000/health
    ProxyPassReverse http://127.0.0.1:9000/health
</Location>
# LB health check targets /health → only passes when app is truly ready
```

---

### EC-110 — Apache `mod_md` (Managed Domains) ACME Challenge Conflicts With Proxy Rules
**Category:** Cloud | **Severity:** Medium | **Env:** Prod

**Scenario:** `mod_md` for automatic Let's Encrypt certificates needs to serve `/.well-known/acme-challenge/` files. But `ProxyPass / http://backend/` proxies ALL requests to backend. ACME challenge fails — cert renewal broken.

**Fix:**
```apache
# Exclude ACME challenge from proxy:
ProxyPass /.well-known/acme-challenge/ !
ProxyPass / http://backend:8080/
# "!" means: do NOT proxy this path — let Apache handle it locally
```

> ⚠️ **DevOps Gotcha:** When using catch-all proxy rules (`ProxyPass /`), always carve out exceptions for health checks, ACME challenges, and server-status BEFORE the catch-all rule. `ProxyPass` is first-match, so exceptions must come first.

---

## Production FAQ

**Q1: Should I use Apache or Nginx? When does Apache make more sense?**  
A: Apache excels when: (1) You need `.htaccess` for per-directory configuration (shared hosting). (2) You need rich module ecosystem (mod_security, mod_auth_*, mod_wsgi). (3) You have existing infrastructure built on Apache. Nginx excels at: pure reverse proxy, high concurrency, lower memory. Both are production-grade. Use event MPM + PHP-FPM to match Nginx's performance characteristics.

**Q2: Apache shows "AH00558: Could not reliably determine the server's fully qualified domain name." How to fix?**  
A: Add `ServerName localhost` to the global `httpd.conf` (outside any virtual host). This is a warning, not an error — Apache still works. But it may cause issues with self-referential URLs and redirects.

**Q3: How do I migrate from Apache 2.2 to 2.4?**  
A: Key changes: (1) Access control: `Order/Allow/Deny` → `Require`. (2) `SSLCertificateChainFile` → concatenate into `SSLCertificateFile`. (3) Module loading: verify all modules available in 2.4. (4) `.htaccess`: check `AllowOverride` still works. Run `apachectl configtest` after each change.

**Q4: Apache is running out of workers during traffic spikes. Quick fix?**  
A: (1) Switch to event MPM if using prefork. (2) Increase `MaxRequestWorkers`. (3) Reduce `KeepAliveTimeout` to 3-5 seconds. (4) Increase `ServerLimit` (requires full restart). (5) Add caching (mod_cache) to reduce backend load. (6) Put a CDN in front for static assets.

**Q5: How do I debug mod_rewrite rules?**  
A: Set `LogLevel alert rewrite:trace3` (or up to trace8 for maximum detail) in the virtual host. Check the error log. Each rule evaluation is logged: match, condition check, substitution result. Remember to remove after debugging.

**Q6: Our `.htaccess` changes aren't taking effect. Checklist?**  
A: (1) `AllowOverride All` set for the directory? (2) `mod_rewrite` loaded? (`a2enmod rewrite`) (3) `.htaccess` in the correct directory? (4) No syntax errors? (check error log) (5) No BOM in file? (6) `RewriteEngine On` declared in THIS `.htaccess`? (not inherited from parent).

**Q7: Apache keeps getting OOM-killed. How to diagnose?**  
A: Check which MPM: `apachectl -V | grep MPM`. If prefork + mod_php: switch to event + PHP-FPM. Check `MaxRequestWorkers` × per-process memory (use `ps aux | grep httpd`). Set `MaxConnectionsPerChild` to recycle workers. Monitor with `server-status`.

**Q8: How to do zero-downtime config reload?**  
A: `apachectl graceful` — sends SIGUSR1 to master process. New workers start with new config, old workers finish current requests then exit. For `ServerLimit` changes, you need a full restart (brief downtime). Blue-green deployment is safest for major config changes.

**Q9: How do I set up Apache as a reverse proxy with health checks?**  
A: Use `mod_proxy` + `mod_proxy_balancer` + `mod_proxy_hcheck` (Apache 2.4.21+). Configure `BalancerMember` with `hcmethod=GET hcuri=/health hcinterval=10`. Set `ProxyPass` with `retry=5` for quick recovery after failure.

**Q10: Apache logs show "MaxRequestWorkers reached" — what happens to new connections?**  
A: New connections are queued in the OS TCP backlog (`ListenBacklog` directive, default 511). If the backlog fills, the OS drops new connections (users see connection timeout). Increase `MaxRequestWorkers`, optimize backend response time, or add more Apache instances behind a load balancer.

---

> **Next:** [IIS Edge Cases](iis_edge_cases.md) · [Windows Server Edge Cases](windows_server_edge_cases.md)  
> **Back:** [Nginx Edge Cases](nginx_edge_cases.md) · [Batch State](batch_state.md) · [🏠 Home](../README.md)
