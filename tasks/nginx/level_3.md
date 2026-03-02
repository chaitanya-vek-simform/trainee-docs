# Nginx — Level 3 (🔴 Hard)

## 1. Dynamic Per-Client Rate Limiting

### How Rate Limiting Works

Rate limiting controls how many requests a client can make in a given time period. Nginx uses a "leaky bucket" algorithm:

```
Incoming Requests (variable rate)
         │ │ │││ │ │ │ │││││ │ │
         ▼ ▼ ▼▼▼▼ ▼ ▼ ▼ ▼▼▼▼▼ ▼ ▼
    ┌────────────────────────────────┐
    │         Leaky Bucket           │
    │  ┌─────────────────────────┐   │
    │  │ burst=5 (queue capacity)│   │
    │  │  [R] [R] [R] [R] [R]   │   │  ← Excess requests queued (up to burst)
    │  └──────────┬──────────────┘   │
    │             │                  │
    │    rate=10r/s (leak rate)      │  ← Processed at steady rate
    │             │                  │
    │             ▼                  │
    │     ┌──────────────┐           │
    │     │   Backend    │           │
    │     └──────────────┘           │
    └────────────────────────────────┘
              │
    Overflow (beyond burst) → 429 Too Many Requests
```

### Part A — Basic Rate Limiting

```nginx
# In http { } context — define the rate limiting zone
# $binary_remote_addr = client IP (compact binary form)
# zone=basic:10m = 10MB shared memory zone named "basic" (~160,000 IPs)
# rate=10r/s = 10 requests per second per IP
limit_req_zone $binary_remote_addr zone=basic:10m rate=10r/s;

server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;

    location / {
        # Apply rate limit
        # burst=20 — allow up to 20 excess requests to queue
        # nodelay  — process burst requests immediately (don't queue them)
        limit_req zone=basic burst=20 nodelay;

        # Custom status code for rate-limited requests (default is 503)
        limit_req_status 429;

        try_files $uri $uri/ =404;
    }
}
```

### Part B — Dynamic Rate Limiting with `geo` Tiers

Create a rate limit tiers configuration file:

```bash
sudo nano /etc/nginx/conf.d/rate-limit-tiers.conf
```

```nginx
# ─── Rate Limit Tiers based on Client IP ───
# Assign each IP/range to a tier

geo $rate_limit_tier {
    default         standard;

    # Premium clients (higher limits)
    10.10.1.0/24    premium;
    192.168.50.0/24 premium;

    # VIP clients (highest limits)
    10.10.100.0/24  vip;
    192.168.100.10  vip;

    # Blocked clients (zero requests allowed)
    203.0.113.0/24  blocked;
    198.51.100.42   blocked;
}
```

Define rate limiting zones for each tier in your `http { }` context:

```nginx
# ─── Per-Tier Rate Limit Zones ───
# Standard: 10 requests/second
limit_req_zone $binary_remote_addr zone=standard:10m rate=10r/s;

# Premium: 50 requests/second
limit_req_zone $binary_remote_addr zone=premium:10m  rate=50r/s;

# VIP: 200 requests/second
limit_req_zone $binary_remote_addr zone=vip:10m      rate=200r/s;

# Blocked: 0 (we'll deny these in the server block)
```

Apply in the server block:

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;

    # Block the blocked tier entirely
    if ($rate_limit_tier = "blocked") {
        return 403;
    }

    location / {
        # Apply rate limits based on tier
        limit_req zone=standard burst=20  nodelay;
        limit_req zone=premium  burst=50  nodelay;
        limit_req zone=vip      burst=100 nodelay;
        limit_req_status 429;

        try_files $uri $uri/ =404;
    }
}
```

### Part C — Advanced Dynamic Rate Limiting with `map`

Use `map` for truly dynamic per-client zone selection. This selects which zone key to use — an empty string means no limit is applied for that zone:

```nginx
# ─── Map tier to zone keys ───
# Only populate the key for the matching tier; empty string = skip that zone

map $rate_limit_tier $standard_key {
    standard    $binary_remote_addr;
    default     "";
}

map $rate_limit_tier $premium_key {
    premium     $binary_remote_addr;
    default     "";
}

map $rate_limit_tier $vip_key {
    vip         $binary_remote_addr;
    default     "";
}

# ─── Define zones using mapped keys ───
limit_req_zone $standard_key zone=standard:10m rate=10r/s;
limit_req_zone $premium_key  zone=premium:10m  rate=50r/s;
limit_req_zone $vip_key      zone=vip:10m      rate=200r/s;
```

Server block:

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;

    if ($rate_limit_tier = "blocked") {
        return 403;
    }

    location / {
        # All three zones are applied, but only the matching one has a non-empty key
        limit_req zone=standard burst=20  nodelay;
        limit_req zone=premium  burst=50  nodelay;
        limit_req zone=vip      burst=100 nodelay;
        limit_req_status 429;

        try_files $uri $uri/ =404;
    }
}
```

### Change Rates Dynamically

To change a client's rate limits:

```bash
# Edit the tiers file
sudo nano /etc/nginx/conf.d/rate-limit-tiers.conf

# Change a client from standard to premium:
#   192.168.1.50  premium;

# Reload (graceful — no downtime)
sudo nginx -t && sudo systemctl reload nginx
```

### Verify with Apache Bench

```bash
# Install Apache Bench
sudo apt install apache2-utils -y

# Send 100 requests with 10 concurrent connections
ab -n 100 -c 10 http://mysite.local/

# Check for 429 responses in the output
# Look for "Non-2xx responses" — these are rate-limited requests
```

```bash
# Quick test with curl loop
for i in $(seq 1 20); do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://mysite.local/)
    echo "Request $i: HTTP $STATUS"
done
```

---

---

## 2. PDF Download Throttling at 200Kbps

### How Throttling Works

Nginx can limit the speed at which it sends response data to the client:

```
200 Kbps (kilobits per second) = 200 / 8 = 25 KB/s (kilobytes per second)

Download without throttling:
┌────────┐   10 MB PDF at 100 MB/s  ┌────────┐
│ Nginx  │══════════════════════════▶│ Client │  Time: ~0.1 seconds
└────────┘                           └────────┘

Download with 200 Kbps throttling:
┌────────┐   10 MB PDF at 25 KB/s   ┌────────┐
│ Nginx  │──·──·──·──·──·──·──·──·─▶│ Client │  Time: ~400 seconds
└────────┘   (drip feed)             └────────┘
```

### Configuration

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;
    index index.html;

    # ─── Throttle PDF downloads to 200 Kbps (25 KB/s) ───
    location ~* \.pdf$ {
        # Allow the first 100KB at full speed, then throttle
        limit_rate_after 100k;

        # Throttle to 25 KB/s (200 Kbps / 8 = 25 KB/s)
        limit_rate 25k;

        # Custom header to indicate throttling is active
        add_header X-Throttled "PDF-200Kbps";

        try_files $uri =404;
    }

    # ─── All other files served at full speed ───
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Directives Explanation

| Directive | Value | Purpose |
|-----------|-------|---------|
| `limit_rate` | `25k` | Maximum transfer rate per connection (25 KB/s = 200 Kbps) |
| `limit_rate_after` | `100k` | Serve the first 100 KB at full speed before throttling kicks in. This allows the download to start quickly. |
| `add_header X-Throttled` | `"PDF-200Kbps"` | Custom header for debugging — confirms throttling is active |

### Test and Verify

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Create a test PDF:

```bash
# Create a 5 MB test file
sudo dd if=/dev/urandom of=/var/www/mysite/files/large-test.pdf bs=1M count=5
sudo chown www-data:www-data /var/www/mysite/files/large-test.pdf
```

Test download speed:

```bash
# Download PDF — observe the speed (should be ~25 KB/s after first 100KB)
curl -o /dev/null -w "Speed: %{speed_download} bytes/sec\nTime: %{time_total}s\nSize: %{size_download} bytes\n" http://mysite.local/files/large-test.pdf
```

Expected output:

```
Speed: 25600.000 bytes/sec     ← ~25 KB/s
Time: 200.123s                 ← Slow due to throttling
Size: 5242880 bytes            ← 5 MB
```

Compare with a non-PDF file (should be full speed):

```bash
# Rename to .txt to test without throttling
sudo cp /var/www/mysite/files/large-test.pdf /var/www/mysite/files/large-test.bin
curl -o /dev/null -w "Speed: %{speed_download} bytes/sec\nTime: %{time_total}s\n" http://mysite.local/files/large-test.bin
```

Check the custom header:

```bash
curl -I http://mysite.local/files/large-test.pdf
# Look for: X-Throttled: PDF-200Kbps
```

---

---

## 3. PCI-DSS Compliant Cipher Suites

### PCI-DSS TLS Requirements

| Requirement | Details |
|-------------|---------|
| **No SSL 2.0 / 3.0** | Completely disable SSLv2 and SSLv3 (known vulnerabilities: POODLE, DROWN) |
| **No TLS 1.0 / 1.1** | Disable TLS 1.0 and 1.1 (deprecated since March 2020) |
| **Minimum TLS 1.2** | TLS 1.2 must be supported as the minimum version |
| **TLS 1.3 Recommended** | TLS 1.3 should be enabled (better performance and security) |
| **No Weak Ciphers** | Disable NULL, EXPORT, DES, 3DES, RC4, MD5, PSK, aDH, aECDH ciphers |
| **Forward Secrecy** | Use ECDHE or DHE key exchange (ensures past sessions can't be decrypted if private key is compromised) |
| **Strong DH Parameters** | Use 2048-bit or higher Diffie-Hellman parameters |

### Step 1 — Generate Self-Signed Certificate and DH Parameters

For testing/staging (use Let's Encrypt for production):

```bash
# Generate self-signed certificate (valid for 365 days)
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/self-signed.key \
    -out /etc/nginx/ssl/self-signed.crt \
    -subj "/C=US/ST=State/L=City/O=Company/OU=DevOps/CN=mysite.local"
```

```bash
# Generate strong Diffie-Hellman parameters (this takes a few minutes)
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

### Step 2 — Create PCI-DSS SSL Snippet

```bash
sudo nano /etc/nginx/snippets/ssl-params.conf
```

```nginx
# ═══════════════════════════════════════════════════════════════
# PCI-DSS Compliant SSL/TLS Configuration
# Last updated: 2 March 2026
# ═══════════════════════════════════════════════════════════════

# ─── Protocol Versions ───
# Only TLS 1.2 and TLS 1.3 (PCI-DSS requires TLS 1.2 minimum)
ssl_protocols TLSv1.2 TLSv1.3;

# ─── Cipher Suites ───
# TLS 1.2 ciphers: ECDHE for forward secrecy, AES-GCM preferred, CHACHA20 for mobile
# TLS 1.3 ciphers are configured automatically and are all strong
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';

# ─── Server Cipher Preference ───
# Server chooses the cipher, not the client (prevents downgrade attacks)
ssl_prefer_server_ciphers on;

# ─── Diffie-Hellman Parameters ───
ssl_dhparam /etc/nginx/ssl/dhparam.pem;

# ─── SSL Session Settings ───
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;    # Disable for forward secrecy

# ─── HSTS (HTTP Strict Transport Security) ───
# Tell browsers to ONLY use HTTPS for the next 2 years
# includeSubDomains — applies to all subdomains
# preload — allow inclusion in browser HSTS preload lists
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# ─── OCSP Stapling ───
# (Only works with certificates that have an OCSP responder — not self-signed)
# ssl_stapling on;
# ssl_stapling_verify on;
# resolver 8.8.8.8 8.8.4.4 valid=300s;
# resolver_timeout 5s;
```

### Step 3 — Apply to Server Block

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
# ─── HTTP → HTTPS Redirect ───
server {
    listen 80;
    listen [::]:80;
    server_name mysite.local;

    # Redirect all HTTP to HTTPS
    return 301 https://$host$request_uri;
}

# ─── HTTPS Server ───
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name mysite.local;

    # ─── SSL Certificate ───
    ssl_certificate     /etc/nginx/ssl/self-signed.crt;
    ssl_certificate_key /etc/nginx/ssl/self-signed.key;

    # ─── Include PCI-DSS SSL Parameters ───
    include /etc/nginx/snippets/ssl-params.conf;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Verify

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Test TLS 1.2 (should work):**

```bash
curl -k --tlsv1.2 -I https://mysite.local
# Expected: HTTP/2 200 (or HTTP/1.1 200)
```

**Test TLS 1.1 (should FAIL):**

```bash
curl -k --tlsv1.1 --tls-max 1.1 -I https://mysite.local
# Expected: SSL handshake error / connection refused
```

**Test TLS 1.0 (should FAIL):**

```bash
curl -k --tlsv1.0 --tls-max 1.0 -I https://mysite.local
# Expected: SSL handshake error / connection refused
```

**Scan with nmap for cipher enumeration:**

```bash
sudo apt install nmap -y
nmap --script ssl-enum-ciphers -p 443 mysite.local
```

Expected output should show:

```
| ssl-enum-ciphers:
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 - strong
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 - strong
|       TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256 - strong
|       TLS_DHE_RSA_WITH_AES_128_GCM_SHA256 - strong
|       TLS_DHE_RSA_WITH_AES_256_GCM_SHA384 - strong
|   TLSv1.3:
|     ciphers:
|       TLS_AKE_WITH_AES_256_GCM_SHA384 - strong
|       TLS_AKE_WITH_CHACHA20_POLY1305_SHA256 - strong
|       TLS_AKE_WITH_AES_128_GCM_SHA256 - strong
|_  least strength: strong
```

No TLS 1.0, no TLS 1.1, no weak ciphers — **PCI-DSS compliant**.

---

---

## 4. Dropping "Secret.json" at TCP Level

### Methods Overview

| Method | Layer | Behavior | Response to Client |
|--------|-------|----------|--------------------|
| **Nginx `return 444`** | Application (HTTP) | Nginx accepts connection, then drops it | Connection reset (no HTTP response) |
| **iptables string match** | Network (TCP) | Kernel drops the packet before Nginx sees it | Connection timeout (no response at all) |
| **Combined (recommended)** | Both | Defense in depth — iptables + Nginx | Timeout or reset depending on which catches it |

### Method 1 — Nginx `return 444`

The special status code `444` tells Nginx to close the connection immediately without sending any response:

```nginx
server {
    listen 80;
    server_name mysite.local;
    root /var/www/mysite;

    # ─── Drop requests for Secret.json (case-insensitive) ───
    location ~* Secret\.json$ {
        return 444;
    }

    # ─── Also catch it in query strings and nested paths ───
    if ($request_uri ~* "Secret\.json") {
        return 444;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Method 2 — iptables String Match

Drop packets at the kernel level before they reach Nginx:

```bash
# Drop any TCP packet containing "Secret.json" in the payload
sudo iptables -I INPUT -p tcp --dport 80 \
    -m string --string "Secret.json" --algo bm \
    -j DROP
```

### iptables Command Explanation

| Part | Meaning |
|------|---------|
| `iptables -I INPUT` | **Insert** a rule at the top of the INPUT chain (incoming traffic) |
| `-p tcp` | Match TCP protocol only |
| `--dport 80` | Match destination port 80 (HTTP) |
| `-m string` | Load the string matching module |
| `--string "Secret.json"` | Match packets containing this exact string |
| `--algo bm` | Use Boyer-Moore algorithm for string matching (faster than `kmp` for most patterns) |
| `-j DROP` | **Drop** the packet silently (no response sent back) |

### Method 3 — Combined (Recommended)

Use both methods for defense in depth:

```bash
# iptables rule (catches at TCP level)
sudo iptables -I INPUT -p tcp --dport 80 \
    -m string --string "Secret.json" --algo bm \
    -j DROP

# Also add for HTTPS port
sudo iptables -I INPUT -p tcp --dport 443 \
    -m string --string "Secret.json" --algo bm \
    -j DROP
```

Plus the Nginx configuration from Method 1 (as a fallback if the iptables rule misses it, e.g., with chunked requests).

### Verify

```bash
# Test Secret.json — should timeout (iptables) or get connection reset (nginx 444)
curl -v --max-time 5 http://mysite.local/Secret.json
# Expected: curl: (28) Operation timed out (iptables)
#       or: curl: (52) Empty reply from server (nginx 444)

# Test that normal files still work
curl -I http://mysite.local/index.html
# Expected: HTTP/1.1 200 OK

# Test case variations
curl -v --max-time 5 http://mysite.local/SECRET.JSON
curl -v --max-time 5 http://mysite.local/secret.json
# Expected: timeout/reset for all case variations (regex is case-insensitive)

# Test in subdirectory
curl -v --max-time 5 http://mysite.local/data/Secret.json
# Expected: timeout/reset
```

### Cleanup iptables Rules

```bash
# List rules with line numbers
sudo iptables -L INPUT --line-numbers -n

# Delete a specific rule by line number
sudo iptables -D INPUT 1

# Or delete by matching the rule exactly
sudo iptables -D INPUT -p tcp --dport 80 \
    -m string --string "Secret.json" --algo bm \
    -j DROP

# Verify rules are removed
sudo iptables -L INPUT -n
```

---

---

## 5. Complete Configuration — All Requirements Combined

This section combines ALL the hard scenarios into a single production-ready configuration.

### Prerequisites

```bash
# Install required modules
sudo apt install nginx libnginx-mod-http-headers-more-filter apache2-utils -y

# Generate self-signed certificate (use Let's Encrypt for production)
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/self-signed.key \
    -out /etc/nginx/ssl/self-signed.crt \
    -subj "/C=US/ST=State/L=City/O=Company/CN=mysite.local"

# Generate DH parameters
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048

# Deploy your static website
sudo mkdir -p /var/www/mysite/{css,js,images,files}
# (copy your site files here)
sudo chown -R www-data:www-data /var/www/mysite

# Create error pages
sudo mkdir -p /var/www/error-pages
# (create 404.html, 403.html, 502.html as shown in Medium scenarios)
sudo chown -R www-data:www-data /var/www/error-pages
```

### Rate Limit Tiers File

```bash
sudo nano /etc/nginx/conf.d/rate-limit-tiers.conf
```

```nginx
# ─── Client Rate Limit Tiers ───
geo $rate_limit_tier {
    default         standard;
    10.10.1.0/24    premium;
    192.168.50.0/24 premium;
    10.10.100.0/24  vip;
    203.0.113.0/24  blocked;
}
```

### Full Combined Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/mysite
```

```nginx
# ═══════════════════════════════════════════════════════════════
# COMPLETE NGINX CONFIGURATION
# Combines: Rate Limiting, SSL/PCI-DSS, Hidden Identity,
#           Custom Error Pages, Secret.json Drop, Caching,
#           PDF Throttling
# ═══════════════════════════════════════════════════════════════

# ─── Rate Limit Zones with Map for Dynamic Selection ───
map $rate_limit_tier $standard_key {
    standard    $binary_remote_addr;
    default     "";
}

map $rate_limit_tier $premium_key {
    premium     $binary_remote_addr;
    default     "";
}

map $rate_limit_tier $vip_key {
    vip         $binary_remote_addr;
    default     "";
}

limit_req_zone $standard_key zone=standard:10m rate=10r/s;
limit_req_zone $premium_key  zone=premium:10m  rate=50r/s;
limit_req_zone $vip_key      zone=vip:10m      rate=200r/s;

# ═══════════════════════════════════════════════════════════════
# HTTP → HTTPS Redirect
# ═══════════════════════════════════════════════════════════════
server {
    listen 80;
    listen [::]:80;
    server_name mysite.local;

    # Redirect all HTTP traffic to HTTPS
    return 301 https://$host$request_uri;
}

# ═══════════════════════════════════════════════════════════════
# Main HTTPS Server
# ═══════════════════════════════════════════════════════════════
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name mysite.local;

    root /var/www/mysite;
    index index.html;

    # ─── SSL / PCI-DSS Compliance ───
    ssl_certificate     /etc/nginx/ssl/self-signed.crt;
    ssl_certificate_key /etc/nginx/ssl/self-signed.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # ─── Hidden Server Identity ───
    server_tokens off;
    more_clear_headers 'Server';
    more_clear_headers 'X-Powered-By';
    etag off;

    # ─── HSTS (HTTP Strict Transport Security) ───
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # ─── Custom Error Pages ───
    error_page 404 /404.html;
    error_page 403 /403.html;
    error_page 502 /502.html;

    location = /404.html {
        root /var/www/error-pages;
        internal;
    }

    location = /403.html {
        root /var/www/error-pages;
        internal;
    }

    location = /502.html {
        root /var/www/error-pages;
        internal;
    }

    # ─── Drop Secret.json (connection reset) ───
    location ~* Secret\.json$ {
        return 444;
    }

    if ($request_uri ~* "Secret\.json") {
        return 444;
    }

    # ─── Block blocked tier ───
    if ($rate_limit_tier = "blocked") {
        return 403;
    }

    # ─── Cache JavaScript files for 30 days ───
    location ~* \.js$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable";
        limit_req zone=standard burst=20  nodelay;
        limit_req zone=premium  burst=50  nodelay;
        limit_req zone=vip      burst=100 nodelay;
        limit_req_status 429;
    }

    # ─── Do NOT cache images ───
    location ~* \.(jpg|jpeg|png|gif|ico|svg|webp|bmp)$ {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate, proxy-revalidate, max-age=0";
        limit_req zone=standard burst=20  nodelay;
        limit_req zone=premium  burst=50  nodelay;
        limit_req zone=vip      burst=100 nodelay;
        limit_req_status 429;
    }

    # ─── Throttle PDF downloads at 200 Kbps ───
    location ~* \.pdf$ {
        limit_rate_after 100k;
        limit_rate 25k;
        add_header X-Throttled "PDF-200Kbps";
        limit_req zone=standard burst=20  nodelay;
        limit_req zone=premium  burst=50  nodelay;
        limit_req zone=vip      burst=100 nodelay;
        limit_req_status 429;
        try_files $uri =404;
    }

    # ─── Default Location ───
    location / {
        limit_req zone=standard burst=20  nodelay;
        limit_req zone=premium  burst=50  nodelay;
        limit_req zone=vip      burst=100 nodelay;
        limit_req_status 429;

        try_files $uri $uri/ =404;
    }

    # ─── Logging ───
    access_log /var/log/nginx/mysite-access.log;
    error_log  /var/log/nginx/mysite-error.log;
}
```

### iptables Rule for Secret.json

```bash
# TCP-level drop for Secret.json
sudo iptables -I INPUT -p tcp --dport 443 \
    -m string --string "Secret.json" --algo bm \
    -j DROP
```

### Comprehensive Test Script

```bash
cat > /tmp/test-nginx.sh << 'TESTEOF'
#!/bin/bash
# ═══════════════════════════════════════════════════════════════
# Nginx Configuration Test Script
# ═══════════════════════════════════════════════════════════════

DOMAIN="mysite.local"
PASS=0
FAIL=0

test_result() {
    if [ "$1" = "PASS" ]; then
        echo -e "  ✅ PASS: $2"
        ((PASS++))
    else
        echo -e "  ❌ FAIL: $2"
        ((FAIL++))
    fi
}

echo "═══════════════════════════════════════════"
echo "  Running Nginx Configuration Tests"
echo "═══════════════════════════════════════════"
echo ""

# ─── Test 1: HTTP → HTTPS Redirect ───
echo "Test 1: HTTP → HTTPS Redirect"
STATUS=$(curl -k -s -o /dev/null -w "%{http_code}" http://$DOMAIN/)
if [ "$STATUS" = "301" ]; then
    test_result "PASS" "HTTP returns 301 redirect"
else
    test_result "FAIL" "HTTP returned $STATUS (expected 301)"
fi

# ─── Test 2: HTTPS Works ───
echo "Test 2: HTTPS Access"
STATUS=$(curl -k -s -o /dev/null -w "%{http_code}" https://$DOMAIN/)
if [ "$STATUS" = "200" ]; then
    test_result "PASS" "HTTPS returns 200 OK"
else
    test_result "FAIL" "HTTPS returned $STATUS (expected 200)"
fi

# ─── Test 3: Server Identity Hidden ───
echo "Test 3: Server Identity Hidden"
SERVER_HEADER=$(curl -k -s -I https://$DOMAIN/ | grep -i "^Server:")
if [ -z "$SERVER_HEADER" ]; then
    test_result "PASS" "Server header is hidden"
else
    test_result "FAIL" "Server header exposed: $SERVER_HEADER"
fi

# ─── Test 4: Secret.json Dropped ───
echo "Test 4: Secret.json Dropped"
STATUS=$(curl -k -s -o /dev/null -w "%{http_code}" --max-time 5 https://$DOMAIN/Secret.json 2>/dev/null)
if [ "$STATUS" = "000" ] || [ -z "$STATUS" ]; then
    test_result "PASS" "Secret.json connection dropped (timeout/reset)"
else
    test_result "FAIL" "Secret.json returned HTTP $STATUS (expected timeout)"
fi

# ─── Test 5: JS Files Have Cache Headers ───
echo "Test 5: JavaScript Caching"
CACHE=$(curl -k -s -I https://$DOMAIN/js/main.js | grep -i "Cache-Control")
if echo "$CACHE" | grep -q "max-age=2592000"; then
    test_result "PASS" "JS files have 30-day cache"
else
    test_result "FAIL" "JS cache header: $CACHE"
fi

# ─── Test 6: PDF Throttle Header ───
echo "Test 6: PDF Throttling"
THROTTLE=$(curl -k -s -I https://$DOMAIN/files/sample.pdf | grep -i "X-Throttled")
if echo "$THROTTLE" | grep -q "PDF-200Kbps"; then
    test_result "PASS" "PDF throttle header present"
else
    test_result "FAIL" "PDF throttle header: $THROTTLE"
fi

# ─── Test 7: TLS 1.1 Rejected ───
echo "Test 7: PCI-DSS — TLS 1.1 Rejected"
RESULT=$(curl -k --tlsv1.1 --tls-max 1.1 -s -o /dev/null -w "%{http_code}" https://$DOMAIN/ 2>/dev/null)
if [ "$RESULT" = "000" ] || [ -z "$RESULT" ]; then
    test_result "PASS" "TLS 1.1 correctly rejected"
else
    test_result "FAIL" "TLS 1.1 returned HTTP $RESULT (should be rejected)"
fi

echo ""
echo "═══════════════════════════════════════════"
echo "  Results: $PASS passed, $FAIL failed"
echo "═══════════════════════════════════════════"
TESTEOF

chmod +x /tmp/test-nginx.sh
bash /tmp/test-nginx.sh
```

---

---

## 6. WebSocket Proxying

### How WebSocket Works Through Nginx

WebSocket provides full-duplex communication over a single TCP connection. It starts as an HTTP request and then "upgrades" to the WebSocket protocol:

```
┌────────┐                    ┌────────┐                    ┌──────────┐
│ Client │                    │ Nginx  │                    │ Backend  │
│        │                    │ (Proxy)│                    │ (WS App) │
│        │─── HTTP GET ──────▶│        │─── HTTP GET ──────▶│          │
│        │    Upgrade:        │        │    Upgrade:        │          │
│        │    websocket       │        │    websocket       │          │
│        │                    │        │                    │          │
│        │◀── 101 Switching ──│        │◀── 101 Switching ──│          │
│        │    Protocols       │        │    Protocols       │          │
│        │                    │        │                    │          │
│        │◀═══ WebSocket ════▶│◀═══════╪═══════════════════▶│          │
│        │   Full Duplex      │  Proxy │   Full Duplex      │          │
│        │   Messages         │        │   Messages         │          │
└────────┘                    └────────┘                    └──────────┘
```

### Step 1 — Create a Node.js WebSocket Backend

```bash
mkdir -p ~/ws-app
cd ~/ws-app
npm init -y
npm install ws
```

```bash
cat > ~/ws-app/server.js << 'EOF'
const http = require('http');
const WebSocket = require('ws');

const server = http.createServer((req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
        <!DOCTYPE html>
        <html>
        <head><title>WebSocket Test</title></head>
        <body>
            <h1>WebSocket Test Page</h1>
            <div id="messages" style="border:1px solid #ccc;padding:10px;height:300px;overflow-y:scroll;"></div>
            <input type="text" id="input" placeholder="Type a message..." style="width:300px;">
            <button onclick="send()">Send</button>
            <script>
                const ws = new WebSocket('ws://' + location.host + '/ws');
                const messages = document.getElementById('messages');
                ws.onopen = () => messages.innerHTML += '<p style="color:green;">Connected!</p>';
                ws.onmessage = (e) => messages.innerHTML += '<p>Server: ' + e.data + '</p>';
                ws.onclose = () => messages.innerHTML += '<p style="color:red;">Disconnected</p>';
                function send() {
                    const input = document.getElementById('input');
                    ws.send(input.value);
                    messages.innerHTML += '<p style="color:blue;">You: ' + input.value + '</p>';
                    input.value = '';
                }
            </script>
        </body>
        </html>
    `);
});

const wss = new WebSocket.Server({ server, path: '/ws' });

wss.on('connection', (ws, req) => {
    const clientIP = req.headers['x-real-ip'] || req.connection.remoteAddress;
    console.log(`Client connected from ${clientIP}`);

    ws.send('Welcome! You are connected to the WebSocket server.');

    ws.on('message', (message) => {
        console.log(`Received: ${message}`);
        // Echo back with timestamp
        ws.send(`Echo: ${message} (${new Date().toISOString()})`);
    });

    ws.on('close', () => {
        console.log('Client disconnected');
    });
});

server.listen(3000, '127.0.0.1', () => {
    console.log('WebSocket server running on http://127.0.0.1:3000');
});
EOF
```

```bash
# Run the WebSocket server
node ~/ws-app/server.js &
```

### Step 2 — Configure Nginx for WebSocket Proxying

```nginx
# ─── Map for WebSocket Connection header ───
# When the client sends "Upgrade: websocket", we need to pass
# "Connection: upgrade" to the backend. For regular requests,
# we close the connection.
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name mysite.local;

    # ─── WebSocket endpoint ───
    location /ws {
        proxy_pass http://127.0.0.1:3000;

        # Required for WebSocket
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Pass client info
        proxy_set_header Host       $host;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Keep WebSocket connection alive for 24 hours
        # (default 60s would disconnect idle connections)
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    # ─── Regular HTTP requests ───
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host       $host;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Verify

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**Test with `wscat` (WebSocket client):**

```bash
# Install wscat
sudo npm install -g wscat

# Connect to WebSocket through Nginx
wscat -c ws://mysite.local/ws
```

You should see:

```
Connected (press CTRL+C to quit)
< Welcome! You are connected to the WebSocket server.
> hello
< Echo: hello (2026-03-02T10:30:00.000Z)
> testing websocket
< Echo: testing websocket (2026-03-02T10:30:05.000Z)
```

**Test via browser:**

Open `http://mysite.local` in a browser. You should see the WebSocket test page with a text input and send button.

---

---

## 7. Security Headers

### Create Security Headers Snippet

```bash
sudo nano /etc/nginx/snippets/security-headers.conf
```

```nginx
# ═══════════════════════════════════════════════════════════════
# Security Headers Configuration
# Last updated: 2 March 2026
# ═══════════════════════════════════════════════════════════════

# ─── X-Frame-Options ───
# Prevent clickjacking by controlling if the site can be embedded in iframes
# SAMEORIGIN = only allow embedding from the same domain
add_header X-Frame-Options "SAMEORIGIN" always;

# ─── X-Content-Type-Options ───
# Prevent MIME-type sniffing (browser won't interpret files as different MIME type)
add_header X-Content-Type-Options "nosniff" always;

# ─── X-XSS-Protection ───
# Enable browser's built-in XSS filter (legacy, but still useful for older browsers)
# mode=block — block the page rather than sanitize
add_header X-XSS-Protection "1; mode=block" always;

# ─── Referrer-Policy ───
# Control how much referrer info is sent with requests
# strict-origin-when-cross-origin = full URL for same-origin, only origin for cross-origin
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# ─── Content-Security-Policy ───
# Control which resources the browser is allowed to load
# This is a starter policy — customize based on your application needs
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' https: data:; connect-src 'self'; frame-ancestors 'self'; base-uri 'self'; form-action 'self';" always;

# ─── Permissions-Policy ───
# Control which browser features/APIs can be used
# Restrict access to sensitive features
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=(), magnetometer=(), gyroscope=(), accelerometer=()" always;

# ─── HSTS ───
# NOTE: HSTS is typically configured in the ssl-params.conf snippet.
# Uncomment below if not using a separate SSL snippet:
# add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# ─── Hide Server Identity ───
server_tokens off;
```

### Include in Server Block

```nginx
server {
    listen 443 ssl;
    server_name mysite.local;

    # SSL configuration
    ssl_certificate     /etc/nginx/ssl/self-signed.crt;
    ssl_certificate_key /etc/nginx/ssl/self-signed.key;
    include /etc/nginx/snippets/ssl-params.conf;

    # ─── Security Headers ───
    include /etc/nginx/snippets/security-headers.conf;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Headers Explanation

| Header | Value | Protection Against |
|--------|-------|--------------------|
| `X-Frame-Options` | `SAMEORIGIN` | **Clickjacking** — prevents malicious sites from embedding your site in an iframe |
| `X-Content-Type-Options` | `nosniff` | **MIME sniffing attacks** — prevents browser from interpreting files as different types |
| `X-XSS-Protection` | `1; mode=block` | **Reflected XSS** — activates browser's built-in XSS filter (legacy browsers) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | **Information leakage** — limits referrer info shared with other sites |
| `Content-Security-Policy` | (policy string) | **XSS, injection, data theft** — whitelist of allowed content sources |
| `Permissions-Policy` | `camera=(), ...` | **Feature abuse** — disables access to device features (camera, mic, location, etc.) |
| `Strict-Transport-Security` | `max-age=63072000` | **Protocol downgrade, cookie hijacking** — forces HTTPS for 2 years |
| `server_tokens off` | — | **Information disclosure** — hides Nginx version from headers and error pages |

### Verify

```bash
sudo nginx -t
sudo systemctl reload nginx
```

```bash
# Check all security headers
curl -k -I https://mysite.local
```

Expected output:

```
HTTP/1.1 200 OK
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'; ...
Permissions-Policy: camera=(), microphone=(), ...
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Type: text/html
```

```bash
# Verify with an online scanner (if publicly accessible)
# https://securityheaders.com/?q=https://yourdomain.com
```

---

---

## 8. Performance Tuning and Optimization

### Optimized `nginx.conf`

```bash
sudo nano /etc/nginx/nginx.conf
```

```nginx
# ═══════════════════════════════════════════════════════════════
# Nginx Performance-Tuned Configuration
# ═══════════════════════════════════════════════════════════════

# Run as the www-data user
user www-data;

# Auto-detect CPU cores (1 worker per core)
worker_processes auto;

# Increase open file limit per worker
worker_rlimit_nofile 65535;

pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;

# ─── Events (connection handling) ───
events {
    # Max simultaneous connections per worker
    worker_connections 4096;

    # Use epoll (efficient event notification on Linux)
    use epoll;

    # Accept as many connections as possible at once
    multi_accept on;
}

# ─── HTTP ───
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # ─── File Serving Optimization ───
    # Use kernel sendfile() for static files (avoids user-space copy)
    sendfile on;

    # Send headers and beginning of file in one packet
    tcp_nopush on;

    # Disable Nagle's algorithm for small packets (reduces latency)
    tcp_nodelay on;

    # ─── Timeouts ───
    keepalive_timeout   65;
    keepalive_requests  1000;
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout        10;

    # ─── Buffer Sizes ───
    client_body_buffer_size    16k;
    client_header_buffer_size  1k;
    client_max_body_size       8m;
    large_client_header_buffers 4 8k;

    # ─── Gzip Compression ───
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 256;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/xml+rss
        application/vnd.ms-fontobject
        font/opentype
        image/svg+xml;
    gzip_disable "msie6";

    # ─── Proxy Cache ───
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=app_cache:10m
                     max_size=1g inactive=60m use_temp_path=off;

    # ─── Rate Limiting ───
    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;

    # ─── Logging ───
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time';

    access_log /var/log/nginx/access.log main;

    # ─── Include site configs ───
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Using Proxy Cache

Add caching to your reverse proxy configuration:

```nginx
server {
    listen 80;
    server_name mysite.local;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # ─── Enable Proxy Caching ───
        proxy_cache app_cache;

        # Cache 200 responses for 10 minutes, 404s for 1 minute
        proxy_cache_valid 200 10m;
        proxy_cache_valid 404 1m;

        # Serve stale content while updating in background
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;

        # Only one request populates the cache (others wait)
        proxy_cache_lock on;

        # Add header to show cache status (HIT, MISS, BYPASS, etc.)
        add_header X-Cache-Status $upstream_cache_status;

        # Pass client info
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    }

    # Bypass cache for admin/API
    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_cache off;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    }
}
```

### Open File Cache

Cache file metadata (existence, size, modification time) to avoid repeated disk lookups:

```nginx
http {
    # Cache file descriptors, sizes, and modification times
    open_file_cache max=10000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
}
```

### Benchmarking

**Apache Bench (`ab`):**

```bash
sudo apt install apache2-utils -y

# 10,000 requests, 100 concurrent connections
ab -n 10000 -c 100 http://mysite.local/

# With keep-alive
ab -n 10000 -c 100 -k http://mysite.local/
```

Key metrics to look for:

```
Requests per second:    5432.10 [#/sec] (mean)
Time per request:       18.409 [ms] (mean)
Transfer rate:          1234.56 [Kbytes/sec] received

Percentage of the requests served within a certain time (ms)
  50%     15
  75%     20
  90%     28
  95%     35
  99%     60
 100%     120 (longest request)
```

**wrk (more advanced benchmarking):**

```bash
# Install wrk
sudo apt install wrk -y

# 30 seconds, 12 threads, 400 connections
wrk -t12 -c400 -d30s http://mysite.local/

# With custom script for POST requests
wrk -t12 -c400 -d30s -s post.lua http://mysite.local/api/data
```

---

---

## 9. Troubleshooting Cheat Sheet

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `bind() to 0.0.0.0:80 failed (98: Address already in use)` | Port 80 is already in use by another process | `sudo ss -tlnp \| grep :80` — find and stop the conflicting process, or use a different port |
| `unknown directive "xxx"` | Typo in config or missing module | Check spelling; install required module (`apt install libnginx-mod-http-xxx`) |
| `502 Bad Gateway` | Backend server is down or unreachable | Check if backend is running: `curl http://127.0.0.1:3000`; check error log |
| `504 Gateway Timeout` | Backend is too slow to respond | Increase `proxy_read_timeout` and `proxy_connect_timeout` |
| `403 Forbidden` | Permission denied on files/directories | Check `namei -l /var/www/mysite/index.html`; fix with `chown www-data:www-data` |
| `404 Not Found` | File doesn't exist or wrong `root` path | Verify `root` directive; check `ls -la /var/www/mysite/` |
| `413 Request Entity Too Large` | Upload exceeds `client_max_body_size` | Increase `client_max_body_size` in config |
| `permission denied` in error log | Nginx worker can't access files | Check file permissions, ownership, and all parent directory permissions |
| `upstream timed out` | Backend didn't respond in time | Increase proxy timeout values; check backend health |
| `SSL: error:...` | Certificate issues | Check cert path, permissions, expiry: `openssl x509 -in cert.pem -noout -dates` |

### Debugging Commands

```bash
# ─── Test configuration for syntax errors ───
sudo nginx -t

# ─── Dump full merged configuration ───
sudo nginx -T

# ─── Check error log (last 50 lines) ───
sudo tail -50 /var/log/nginx/error.log

# ─── Real-time error log monitoring ───
sudo tail -f /var/log/nginx/error.log

# ─── Check running Nginx processes ───
ps aux | grep nginx

# ─── Check what ports Nginx is listening on ───
sudo ss -tlnp | grep nginx

# ─── List loaded modules ───
nginx -V 2>&1 | tr -- - '\n' | grep module

# ─── Check file permission chain ───
namei -l /var/www/mysite/index.html

# ─── Check SELinux context (if applicable) ───
ls -laZ /var/www/mysite/

# ─── Check if SELinux is blocking ───
sudo ausearch -m avc -ts recent
```

### Debug Logging

Enable debug logging for detailed troubleshooting (temporary only — very verbose):

```nginx
# Global debug logging
error_log /var/log/nginx/error.log debug;

# Debug for specific IP only (less noise)
events {
    debug_connection 192.168.1.100;
}
```

> **Warning:** Debug logging generates massive amounts of data. Only enable temporarily and for specific IPs when possible.

```bash
# Reload and check
sudo nginx -t && sudo systemctl reload nginx
sudo tail -f /var/log/nginx/error.log
```

---

---

## 10. Quick Reference — Important Files and Commands

### File Locations

| File / Directory | Purpose |
|------------------|---------|
| `/etc/nginx/nginx.conf` | Main configuration file |
| `/etc/nginx/sites-available/` | All server block configs |
| `/etc/nginx/sites-enabled/` | Symlinks to active configs |
| `/etc/nginx/conf.d/` | Additional config files (auto-included) |
| `/etc/nginx/snippets/` | Reusable config fragments (SSL, security headers, etc.) |
| `/etc/nginx/mime.types` | File extension → MIME type mapping |
| `/etc/nginx/ssl/` | SSL certificates and keys |
| `/etc/nginx/.htpasswd` | HTTP Basic Auth password file |
| `/var/log/nginx/access.log` | Access log (all requests) |
| `/var/log/nginx/error.log` | Error log |
| `/var/www/` | Web root directories |
| `/var/cache/nginx/` | Proxy cache storage |
| `/run/nginx.pid` | PID file for master process |
| `/etc/logrotate.d/nginx` | Log rotation configuration |

### Essential Commands

| Command | Purpose |
|---------|---------|
| `sudo nginx -t` | Test configuration syntax |
| `sudo nginx -T` | Test and dump full configuration |
| `sudo systemctl start nginx` | Start Nginx |
| `sudo systemctl stop nginx` | Stop Nginx |
| `sudo systemctl restart nginx` | Full restart (brief downtime) |
| `sudo systemctl reload nginx` | Graceful reload (no downtime) |
| `sudo systemctl status nginx` | Check service status |
| `sudo systemctl enable nginx` | Enable start on boot |
| `nginx -v` | Show version |
| `nginx -V` | Show version + build details + modules |
| `ps aux \| grep nginx` | List running processes |
| `sudo ss -tlnp \| grep nginx` | Show listening ports |
| `sudo tail -f /var/log/nginx/error.log` | Real-time error log |
| `sudo tail -f /var/log/nginx/access.log` | Real-time access log |

### Common Configuration Patterns

**Static file server:**

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Reverse proxy:**

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**HTTP → HTTPS redirect:**

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

**Load balancer:**

```nginx
upstream backends {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backends;
    }
}
```

**Deny hidden files (dotfiles):**

```nginx
# Block access to .git, .env, .htaccess, etc.
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}
```

### Location Block Priority

Nginx evaluates `location` blocks in a specific order of priority (highest to lowest):

| Priority | Modifier | Type | Example | Behavior |
|----------|----------|------|---------|----------|
| 1 | `=` | **Exact match** | `location = /favicon.ico` | Matches only the exact URI. Stops searching immediately. |
| 2 | `^~` | **Prefix (priority)** | `location ^~ /static/` | Prefix match that skips regex evaluation if matched. |
| 3 | `~` | **Regex (case-sensitive)** | `location ~ \.php$` | Case-sensitive regular expression match. First match wins. |
| 4 | `~*` | **Regex (case-insensitive)** | `location ~* \.(jpg\|png)$` | Case-insensitive regular expression match. First match wins. |
| 5 | (none) | **Prefix (normal)** | `location /api/` | Standard prefix match. Longest prefix wins, but regex can override. |

**Example — How Nginx selects a location for `/static/logo.png`:**

```nginx
location = /static/logo.png { }       # ① Checked first  — exact match? YES → USE THIS
location ^~ /static/ { }              # ② Checked second — prefix match? YES, but #1 won
location ~ \.(png|jpg)$ { }           # ③ Checked third  — regex match? YES, but #1 won
location /static/ { }                 # ④ Checked fourth — prefix match? YES, but #1 won
location / { }                        # ⑤ Checked last   — catch-all
```

If `= /static/logo.png` didn't exist, `^~ /static/` would win (skips regex).
If `^~ /static/` didn't exist, `~ \.(png|jpg)$` would win (regex overrides normal prefix).
