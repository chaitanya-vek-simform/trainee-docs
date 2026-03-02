# Apache — Level 3 (🔴 Hard)

## 1. Practice Scenario 6 — Failover / Hot-Standby

**Goal:** Configure 2 primary backend servers and 1 hot-standby. If both primaries go down, the standby automatically takes over. When primaries recover, traffic returns to them.

### Step 1 — Start 3 backend servers

**Terminal 1 — Primary Backend 1 (port 9001):**

```bash
mkdir -p /tmp/primary1 && echo "Response from PRIMARY 1 (port 9001)" > /tmp/primary1/index.html
cd /tmp/primary1 && python3 -m http.server 9001
```

**Terminal 2 — Primary Backend 2 (port 9002):**

```bash
mkdir -p /tmp/primary2 && echo "Response from PRIMARY 2 (port 9002)" > /tmp/primary2/index.html
cd /tmp/primary2 && python3 -m http.server 9002
```

**Terminal 3 — Hot-Standby Backend (port 9003):**

```bash
mkdir -p /tmp/standby && echo "Response from HOT-STANDBY (port 9003)" > /tmp/standby/index.html
cd /tmp/standby && python3 -m http.server 9003
```

### Step 2 — Enable required modules

```bash
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests headers
sudo systemctl restart apache2
```

### Step 3 — Create the failover configuration

```bash
sudo tee /etc/apache2/sites-available/failover-lb.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    <Proxy "balancer://mycluster">
        BalancerMember http://127.0.0.1:9001 retry=10
        BalancerMember http://127.0.0.1:9002 retry=10
        BalancerMember http://127.0.0.1:9003 status=+H retry=10
        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/

    ErrorLog ${APACHE_LOG_DIR}/failover-error.log
    CustomLog ${APACHE_LOG_DIR}/failover-access.log combined
</VirtualHost>
EOF
```

> **Key points:**
> - `status=+H` marks backend on port 9003 as a **hot-standby** — it only receives traffic when all other members are down.
> - `retry=10` means Apache waits 10 seconds before retrying a failed backend.

### Step 4 — Enable and activate

```bash
sudo a2dissite 000-default.conf
sudo a2ensite failover-lb.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Step 5 — Test normal operation

```bash
for i in $(seq 1 6); do
    echo "Request $i: $(curl -s http://localhost)"
done
```

**Expected output:** Requests alternate between PRIMARY 1 and PRIMARY 2 only. The hot-standby does **not** receive any traffic.

```
Request 1: Response from PRIMARY 1 (port 9001)
Request 2: Response from PRIMARY 2 (port 9002)
Request 3: Response from PRIMARY 1 (port 9001)
Request 4: Response from PRIMARY 2 (port 9002)
Request 5: Response from PRIMARY 1 (port 9001)
Request 6: Response from PRIMARY 2 (port 9002)
```

### Step 6 — Simulate failure by stopping both primaries

Go to **Terminal 1** and **Terminal 2** and press `Ctrl+C` to stop both primary backends.

Now test again:

```bash
for i in $(seq 1 3); do
    echo "Request $i: $(curl -s http://localhost)"
done
```

**Expected output:** Traffic is routed to the hot-standby.

```
Request 1: Response from HOT-STANDBY (port 9003)
Request 2: Response from HOT-STANDBY (port 9003)
Request 3: Response from HOT-STANDBY (port 9003)
```

### Step 7 — Restore primaries and verify automatic recovery

Restart the primary backends:

**Terminal 1:**

```bash
cd /tmp/primary1 && python3 -m http.server 9001
```

**Terminal 2:**

```bash
cd /tmp/primary2 && python3 -m http.server 9002
```

Wait at least **10 seconds** (the `retry` interval), then test:

```bash
sleep 12
for i in $(seq 1 6); do
    echo "Request $i: $(curl -s http://localhost)"
done
```

**Expected output:** Traffic returns to the primary backends. The hot-standby goes back to idle.

```
Request 1: Response from PRIMARY 1 (port 9001)
Request 2: Response from PRIMARY 2 (port 9002)
Request 3: Response from PRIMARY 1 (port 9001)
Request 4: Response from PRIMARY 2 (port 9002)
Request 5: Response from PRIMARY 1 (port 9001)
Request 6: Response from PRIMARY 2 (port 9002)
```

### Step 8 — Cleanup

```bash
# Stop all Python backends (Ctrl+C in their terminals)

sudo a2dissite failover-lb.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2
sudo rm /etc/apache2/sites-available/failover-lb.conf
rm -rf /tmp/primary1 /tmp/primary2 /tmp/standby
```

---

---

## 2. Practice Scenario 7 — HTTPS Termination + Reverse Proxy

**Goal:** Configure Apache to handle HTTPS with a self-signed certificate, automatically redirect HTTP to HTTPS, and proxy requests to a backend on port 8080.

### Step 1 — Generate a self-signed SSL certificate

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/apache-selfsigned.key \
    -out /etc/ssl/certs/apache-selfsigned.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"
```

This creates:
- **Private key:** `/etc/ssl/private/apache-selfsigned.key`
- **Certificate:** `/etc/ssl/certs/apache-selfsigned.crt`

### Step 2 — Enable the SSL module

```bash
sudo a2enmod ssl proxy proxy_http headers rewrite
sudo systemctl restart apache2
```

### Step 3 — Start a backend server

```bash
mkdir -p /tmp/https-backend && echo "Hello from HTTPS-terminated Backend (port 8080)" > /tmp/https-backend/index.html
cd /tmp/https-backend && python3 -m http.server 8080
```

### Step 4 — Create the HTTPS virtual host configuration

```bash
sudo tee /etc/apache2/sites-available/https-proxy.conf > /dev/null <<'EOF'
# HTTP — Redirect all traffic to HTTPS
<VirtualHost *:80>
    ServerName localhost
    Redirect permanent / https://localhost/
</VirtualHost>

# HTTPS — SSL termination + reverse proxy
<VirtualHost *:443>
    ServerName localhost

    # SSL Configuration
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

    # Proxy Configuration
    ProxyPreserveHost On

    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"

    ProxyPass / http://127.0.0.1:8080/
    ProxyPassReverse / http://127.0.0.1:8080/

    ErrorLog ${APACHE_LOG_DIR}/https-proxy-error.log
    CustomLog ${APACHE_LOG_DIR}/https-proxy-access.log combined
</VirtualHost>
EOF
```

#### Configuration breakdown

| Directive | Purpose |
|-----------|---------|
| `Redirect permanent / https://localhost/` | Sends a 301 redirect from HTTP to HTTPS, ensuring all traffic uses encryption. |
| `SSLEngine on` | Enables the SSL/TLS engine for this virtual host. |
| `SSLCertificateFile` | Path to the public SSL certificate. |
| `SSLCertificateKeyFile` | Path to the private SSL key. |
| `RequestHeader set X-Forwarded-Proto "https"` | Informs the backend that the original request came over HTTPS. |
| `ProxyPass / http://127.0.0.1:8080/` | Proxies decrypted traffic to the backend over plain HTTP. |

### Step 5 — Enable and activate

```bash
sudo a2dissite 000-default.conf
sudo a2ensite https-proxy.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Step 6 — Test HTTPS with curl

```bash
# Test HTTPS (use -k to accept self-signed certificate)
curl -k https://localhost
```

**Expected output:**

```
Hello from HTTPS-terminated Backend (port 8080)
```

### Step 7 — Test HTTP → HTTPS redirect

```bash
curl -I http://localhost
```

**Expected output:**

```
HTTP/1.1 301 Moved Permanently
Location: https://localhost/
```

The `301 Moved Permanently` confirms that HTTP traffic is being redirected to HTTPS.

### Step 8 — Cleanup

```bash
# Stop the Python backend (Ctrl+C in its terminal)

sudo a2dissite https-proxy.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2
sudo rm /etc/apache2/sites-available/https-proxy.conf
sudo rm /etc/ssl/private/apache-selfsigned.key
sudo rm /etc/ssl/certs/apache-selfsigned.crt
rm -rf /tmp/https-backend
```

---

---

## 3. Practice Scenario 8 — WebSocket Proxying

**Goal:** Configure Apache to proxy WebSocket connections to a Python WebSocket server running on port 9090.

### Step 1 — Install the websockets library

```bash
pip3 install websockets
```

### Step 2 — Create a simple WebSocket server

```bash
cat > /tmp/ws_server.py << 'PYEOF'
import asyncio
import websockets

async def handler(websocket, path):
    async for message in websocket:
        response = f"Echo: {message}"
        await websocket.send(response)
        print(f"Received: {message} | Sent: {response}")

async def main():
    server = await websockets.serve(handler, "0.0.0.0", 9090)
    print("WebSocket server running on ws://0.0.0.0:9090")
    await server.wait_closed()

asyncio.run(main())
PYEOF
```

### Step 3 — Start the WebSocket server

```bash
python3 /tmp/ws_server.py
```

Leave this terminal running.

### Step 4 — Enable required modules

```bash
sudo a2enmod proxy proxy_http proxy_wstunnel rewrite headers
sudo systemctl restart apache2
```

### Step 5 — Create the WebSocket proxy configuration

```bash
sudo tee /etc/apache2/sites-available/websocket-proxy.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    # WebSocket upgrade handling
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule ^/?(.*) ws://127.0.0.1:9090/$1 [P,L]

    # Regular HTTP proxy (fallback)
    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:9090/
    ProxyPassReverse / http://127.0.0.1:9090/

    ErrorLog ${APACHE_LOG_DIR}/websocket-proxy-error.log
    CustomLog ${APACHE_LOG_DIR}/websocket-proxy-access.log combined
</VirtualHost>
EOF
```

#### Configuration breakdown

| Directive | Purpose |
|-----------|---------|
| `RewriteEngine On` | Enables the Apache URL rewriting engine. |
| `RewriteCond %{HTTP:Upgrade} websocket [NC]` | Checks if the `Upgrade` header contains `websocket` (case-insensitive). |
| `RewriteCond %{HTTP:Connection} upgrade [NC]` | Checks if the `Connection` header contains `upgrade`. |
| `RewriteRule ^/?(.*) ws://127.0.0.1:9090/$1 [P,L]` | If both conditions match, proxy the request to the WebSocket backend using the `ws://` protocol. `[P]` = proxy, `[L]` = last rule. |
| `ProxyPass / http://127.0.0.1:9090/` | Handles regular HTTP requests as a fallback. |

### Step 6 — Enable and activate

```bash
sudo a2dissite 000-default.conf
sudo a2ensite websocket-proxy.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Step 7 — Test with wscat

First, install `wscat` (a WebSocket client):

```bash
sudo npm install -g wscat
```

Then connect to the WebSocket through Apache:

```bash
wscat -c ws://localhost
```

Once connected, type a message:

```
> Hello, WebSocket!
< Echo: Hello, WebSocket!

> Testing Apache proxy
< Echo: Testing Apache proxy
```

Press `Ctrl+C` to disconnect.

### Step 8 — Cleanup

```bash
# Stop the WebSocket server (Ctrl+C in its terminal)

sudo a2dissite websocket-proxy.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2
sudo rm /etc/apache2/sites-available/websocket-proxy.conf
rm /tmp/ws_server.py
```

---

---

## 4. Troubleshooting Cheat Sheet

### Common Issues & Solutions

| Problem | Possible Cause | Solution |
|---------|---------------|----------|
| **503 Service Unavailable** | All backend servers are down or unreachable. | Verify backends are running: `curl http://127.0.0.1:<port>`. Check balancer member status. |
| **403 Forbidden** | Apache denying access due to directory permissions or `Require` directives. | Check `<Directory>` or `<Location>` blocks. Ensure `Require all granted` is set where needed. |
| **Connection Refused** | Backend is not listening on the expected port. | Confirm the backend is running: `ss -tlnp | grep <port>`. Check firewall rules: `sudo ufw status`. |
| **502 Bad Gateway** | Apache cannot communicate with the backend (network issue, wrong address, or backend crashed). | Verify backend address and port. Check error logs: `sudo tail -f /var/log/apache2/error.log`. |
| **ERR_TOO_MANY_REDIRECTS** | Redirect loop between HTTP and HTTPS configurations. | Ensure the HTTP `<VirtualHost>` only redirects, and the HTTPS `<VirtualHost>` does not redirect again. Check `ProxyPass` targets use `http://` not `https://`. |
| **Sticky sessions not working** | Cookie not being set or wrong `stickysession` name. | Verify the `Set-Cookie` header is present in responses. Ensure `stickysession=ROUTEID` matches the cookie name. Check `route=` is set on each `BalancerMember`. |
| **SSL errors** | Certificate issues, wrong paths, or expired certificates. | Verify certificate files exist and are readable: `sudo ls -la /etc/ssl/certs/`. Test with `openssl s_client -connect localhost:443`. |
| **Modules not loaded** | Required modules not enabled. | List loaded modules: `sudo apachectl -M`. Enable missing ones: `sudo a2enmod <module>`. Restart Apache. |

### Useful Debugging Commands

```bash
# Check Apache configuration syntax
sudo apache2ctl configtest

# View loaded modules
sudo apachectl -M

# View active virtual hosts
sudo apachectl -S

# Tail error log in real-time
sudo tail -f /var/log/apache2/error.log

# Tail access log in real-time
sudo tail -f /var/log/apache2/access.log

# Check which process is listening on a port
sudo ss -tlnp | grep :<port>

# Test backend connectivity directly
curl -v http://127.0.0.1:<port>

# Check firewall status
sudo ufw status verbose

# Restart Apache completely
sudo systemctl restart apache2

# Reload Apache (graceful — no downtime)
sudo systemctl reload apache2
```

---

---

## 5. Quick Reference — Important Files & Commands

### File Locations

| File / Directory | Purpose |
|-----------------|---------|
| `/etc/apache2/apache2.conf` | Main Apache configuration file. |
| `/etc/apache2/sites-available/` | Directory for virtual host configuration files (available but not necessarily active). |
| `/etc/apache2/sites-enabled/` | Symlinks to active virtual host configs in `sites-available/`. |
| `/etc/apache2/mods-available/` | Directory of available Apache modules. |
| `/etc/apache2/mods-enabled/` | Symlinks to active modules in `mods-available/`. |
| `/var/log/apache2/error.log` | Default Apache error log. |
| `/var/log/apache2/access.log` | Default Apache access log. |
| `/etc/ssl/certs/` | Directory for SSL certificate files. |
| `/etc/ssl/private/` | Directory for SSL private key files. |

### Essential Commands

| Command | Description |
|---------|-------------|
| `sudo a2ensite <config>.conf` | Enable a virtual host configuration. |
| `sudo a2dissite <config>.conf` | Disable a virtual host configuration. |
| `sudo a2enmod <module>` | Enable an Apache module. |
| `sudo a2dismod <module>` | Disable an Apache module. |
| `sudo apache2ctl configtest` | Test Apache configuration for syntax errors. |
| `sudo apachectl -M` | List all loaded modules. |
| `sudo apachectl -S` | Show virtual host settings and configuration file locations. |
| `sudo systemctl start apache2` | Start the Apache service. |
| `sudo systemctl stop apache2` | Stop the Apache service. |
| `sudo systemctl restart apache2` | Restart Apache (full restart — brief downtime). |
| `sudo systemctl reload apache2` | Reload Apache (graceful — no downtime). |
| `sudo systemctl status apache2` | Check the current status of Apache. |
| `sudo systemctl enable apache2` | Enable Apache to start on boot. |

### ProxyPass Directive Quick Reference

```apache
# Basic reverse proxy
ProxyPass / http://backend:port/
ProxyPassReverse / http://backend:port/

# Path-based routing
ProxyPass /app1 http://127.0.0.1:3001/
ProxyPassReverse /app1 http://127.0.0.1:3001/

# Load balancer
ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/

# Exclude a path from proxying (e.g., for balancer-manager)
ProxyPass /balancer-manager !

# With timeout settings
ProxyPass / http://backend:port/ timeout=60 retry=5

# WebSocket proxy via rewrite
RewriteEngine On
RewriteCond %{HTTP:Upgrade} websocket [NC]
RewriteRule ^/?(.*) ws://backend:port/$1 [P,L]
```
