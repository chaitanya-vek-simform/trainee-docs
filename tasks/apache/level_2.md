# Apache — Level 2 (🟡 Medium)

## 1. Load Balancing Fundamentals

### 1.1 What is Load Balancing?

Load balancing distributes incoming requests across multiple backend servers. This brings several key advantages:

- **Improve Performance** — Spread the workload so no single server becomes a bottleneck.
- **Increase Reliability** — If one server goes down, traffic is routed to the remaining healthy servers.
- **Enable Scaling** — Add or remove backend servers to match demand without downtime.

---

### 1.2 Load Balancing Methods Explained

Apache supports several load balancing algorithms via different modules:

| Method | Module | Directive | How It Works |
|--------|--------|-----------|-------------|
| **Round Robin** | `lbmethod_byrequests` | `byrequests` | Distributes requests equally across all backends, one by one in rotation. Simplest and most common method. |
| **Weighted** | `lbmethod_byrequests` | `byrequests` + `loadfactor` | Distributes requests proportionally based on a weight (`loadfactor`) assigned to each backend. Higher weight = more traffic. |
| **By Traffic** | `lbmethod_bytraffic` | `bytraffic` | Distributes based on the amount of data (bytes) transferred. Backends that have transferred less data receive more requests. |
| **By Busyness** | `lbmethod_bybusyness` | `bybusyness` | Sends requests to the backend with the fewest active connections. Best for backends with uneven processing times. |

---

### 1.3 How to Set Up Load Balancing

Here is a step-by-step guide to create a basic load-balanced configuration.

#### Step 1 — Enable required modules

```bash
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests headers
sudo systemctl restart apache2
```

#### Step 2 — Create the load balancer configuration

```bash
sudo tee /etc/apache2/sites-available/loadbalancer.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    <Proxy "balancer://mycluster">
        BalancerMember http://127.0.0.1:9001
        BalancerMember http://127.0.0.1:9002
        BalancerMember http://127.0.0.1:9003
        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>
EOF
```

#### Line-by-line explanation

| Line | Meaning |
|------|---------|
| `<Proxy "balancer://mycluster">` | Defines a load balancer cluster named `mycluster`. The `balancer://` scheme tells Apache this is a balancer group. |
| `BalancerMember http://127.0.0.1:9001` | Adds the backend server on port 9001 as a member of the cluster. |
| `BalancerMember http://127.0.0.1:9002` | Adds the backend server on port 9002 as a member of the cluster. |
| `BalancerMember http://127.0.0.1:9003` | Adds the backend server on port 9003 as a member of the cluster. |
| `ProxySet lbmethod=byrequests` | Sets the load balancing algorithm to Round Robin (by request count). |
| `ProxyPass / balancer://mycluster/` | Forwards all incoming requests to the `mycluster` balancer group. |
| `ProxyPassReverse / balancer://mycluster/` | Rewrites response headers so clients see the proxy URL, not the backend URL. |

#### Step 3 — Enable and reload

```bash
sudo a2ensite loadbalancer.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

### 1.4 Health Checks & Failover

#### Retry Parameter

If a backend server fails, Apache will mark it as unavailable and **retry** after a set interval:

```apache
BalancerMember http://127.0.0.1:9001 retry=10
```

This tells Apache to wait **10 seconds** before retrying a failed backend.

#### Hot-Standby

A **hot-standby** server only receives traffic when all primary backends are unavailable:

```apache
<Proxy "balancer://mycluster">
    BalancerMember http://127.0.0.1:9001
    BalancerMember http://127.0.0.1:9002
    BalancerMember http://127.0.0.1:9003 status=+H
    ProxySet lbmethod=byrequests
</Proxy>
```

In this configuration, backend on port 9003 is a **hot-standby** — it sits idle until 9001 and 9002 are both down.

#### Status Flags

| Flag | Name | Description |
|------|------|-------------|
| `+H` | Hot Standby | Only receives traffic when all other members are unavailable. |
| `+D` | Disabled | Completely disabled; receives no traffic. Useful for maintenance. |
| `+S` | Stopped | Administratively stopped; will not receive new requests. |
| `+E` | In Error | Marked as being in an error state; Apache will attempt retries. |

#### Balancer Manager UI

Apache provides a built-in web interface to monitor and manage the load balancer in real time:

```apache
<VirtualHost *:80>
    ServerName localhost

    <Proxy "balancer://mycluster">
        BalancerMember http://127.0.0.1:9001
        BalancerMember http://127.0.0.1:9002
        BalancerMember http://127.0.0.1:9003 status=+H
        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass /balancer-manager !
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/

    <Location /balancer-manager>
        SetHandler balancer-manager
        Require ip 127.0.0.1
    </Location>
</VirtualHost>
```

> **Note:** `ProxyPass /balancer-manager !` ensures that requests to `/balancer-manager` are **not** proxied and instead handled locally by Apache.

Access the Balancer Manager at `http://localhost/balancer-manager` to view backend status, toggle members, and adjust weights.

---

---

## 2. Practice Scenario 3 — Round-Robin Load Balancing

**Goal:** Distribute traffic equally across 3 backends using the Round Robin method.

### Step 1 — Start 3 backend servers

Open **three separate terminals**:

**Terminal 1 — Backend on port 9001:**

```bash
mkdir -p /tmp/backend1 && echo "Response from Backend 1 (port 9001)" > /tmp/backend1/index.html
cd /tmp/backend1 && python3 -m http.server 9001
```

**Terminal 2 — Backend on port 9002:**

```bash
mkdir -p /tmp/backend2 && echo "Response from Backend 2 (port 9002)" > /tmp/backend2/index.html
cd /tmp/backend2 && python3 -m http.server 9002
```

**Terminal 3 — Backend on port 9003:**

```bash
mkdir -p /tmp/backend3 && echo "Response from Backend 3 (port 9003)" > /tmp/backend3/index.html
cd /tmp/backend3 && python3 -m http.server 9003
```

### Step 2 — Enable required modules

```bash
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests headers
sudo systemctl restart apache2
```

### Step 3 — Create the load balancer configuration

```bash
sudo tee /etc/apache2/sites-available/round-robin-lb.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    <Proxy "balancer://mycluster">
        BalancerMember http://127.0.0.1:9001
        BalancerMember http://127.0.0.1:9002
        BalancerMember http://127.0.0.1:9003
        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/

    ErrorLog ${APACHE_LOG_DIR}/round-robin-error.log
    CustomLog ${APACHE_LOG_DIR}/round-robin-access.log combined
</VirtualHost>
EOF
```

### Step 4 — Enable and activate

```bash
sudo a2dissite 000-default.conf
sudo a2ensite round-robin-lb.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Step 5 — Test with 9 requests to observe rotation

```bash
for i in $(seq 1 9); do
    echo "Request $i: $(curl -s http://localhost)"
done
```

**Expected output (Round Robin rotation):**

```
Request 1: Response from Backend 1 (port 9001)
Request 2: Response from Backend 2 (port 9002)
Request 3: Response from Backend 3 (port 9003)
Request 4: Response from Backend 1 (port 9001)
Request 5: Response from Backend 2 (port 9002)
Request 6: Response from Backend 3 (port 9003)
Request 7: Response from Backend 1 (port 9001)
Request 8: Response from Backend 2 (port 9002)
Request 9: Response from Backend 3 (port 9003)
```

Each backend receives exactly **3 out of 9 requests** — a perfect round-robin distribution.

### Step 6 — Cleanup

```bash
# Stop all Python backends (Ctrl+C in their terminals)

sudo a2dissite round-robin-lb.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2
sudo rm /etc/apache2/sites-available/round-robin-lb.conf
rm -rf /tmp/backend1 /tmp/backend2 /tmp/backend3
```

---

---

## 3. Practice Scenario 4 — Weighted Load Balancing

**Goal:** Distribute traffic proportionally across 3 backends with weights 60%, 30%, and 10%.

### Step 1 — Start 3 backend servers

Open **three separate terminals** (same as Scenario 3):

**Terminal 1:**

```bash
mkdir -p /tmp/backend1 && echo "Response from Backend 1 (port 9001)" > /tmp/backend1/index.html
cd /tmp/backend1 && python3 -m http.server 9001
```

**Terminal 2:**

```bash
mkdir -p /tmp/backend2 && echo "Response from Backend 2 (port 9002)" > /tmp/backend2/index.html
cd /tmp/backend2 && python3 -m http.server 9002
```

**Terminal 3:**

```bash
mkdir -p /tmp/backend3 && echo "Response from Backend 3 (port 9003)" > /tmp/backend3/index.html
cd /tmp/backend3 && python3 -m http.server 9003
```

### Step 2 — Create the weighted load balancer configuration

```bash
sudo tee /etc/apache2/sites-available/weighted-lb.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    <Proxy "balancer://mycluster">
        BalancerMember http://127.0.0.1:9001 loadfactor=60
        BalancerMember http://127.0.0.1:9002 loadfactor=30
        BalancerMember http://127.0.0.1:9003 loadfactor=10
        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/

    ErrorLog ${APACHE_LOG_DIR}/weighted-lb-error.log
    CustomLog ${APACHE_LOG_DIR}/weighted-lb-access.log combined
</VirtualHost>
EOF
```

> **How `loadfactor` works:** The `loadfactor` value is a relative weight. With values of 60, 30, and 10 (total = 100), Backend 1 gets ~60% of requests, Backend 2 gets ~30%, and Backend 3 gets ~10%.

### Step 3 — Enable and activate

```bash
sudo a2dissite 000-default.conf
sudo a2ensite weighted-lb.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Step 4 — Test with 20 requests and observe distribution

```bash
for i in $(seq 1 20); do
    curl -s http://localhost
done | sort | uniq -c | sort -rn
```

**Expected output (approximate distribution):**

```
     12 Response from Backend 1 (port 9001)
      6 Response from Backend 2 (port 9002)
      2 Response from Backend 3 (port 9003)
```

Backend 1 (~60%) receives the most traffic, Backend 2 (~30%) receives moderate traffic, and Backend 3 (~10%) receives the least.

### Step 5 — Cleanup

```bash
# Stop all Python backends (Ctrl+C in their terminals)

sudo a2dissite weighted-lb.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2
sudo rm /etc/apache2/sites-available/weighted-lb.conf
rm -rf /tmp/backend1 /tmp/backend2 /tmp/backend3
```

---

---

## 4. Practice Scenario 5 — Sticky Sessions

**Goal:** Ensure a client always reaches the same backend server using cookie-based session stickiness.

### Step 1 — Start 2 backend servers

**Terminal 1 — Backend on port 9001:**

```bash
mkdir -p /tmp/sticky1 && echo "Response from Backend 1 (port 9001)" > /tmp/sticky1/index.html
cd /tmp/sticky1 && python3 -m http.server 9001
```

**Terminal 2 — Backend on port 9002:**

```bash
mkdir -p /tmp/sticky2 && echo "Response from Backend 2 (port 9002)" > /tmp/sticky2/index.html
cd /tmp/sticky2 && python3 -m http.server 9002
```

### Step 2 — Enable required modules

```bash
sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests headers
sudo systemctl restart apache2
```

### Step 3 — Create the sticky session configuration

```bash
sudo tee /etc/apache2/sites-available/sticky-lb.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName localhost

    Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED

    <Proxy "balancer://mycluster">
        BalancerMember http://127.0.0.1:9001 route=server1
        BalancerMember http://127.0.0.1:9002 route=server2
        ProxySet lbmethod=byrequests
        ProxySet stickysession=ROUTEID
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/

    ErrorLog ${APACHE_LOG_DIR}/sticky-lb-error.log
    CustomLog ${APACHE_LOG_DIR}/sticky-lb-access.log combined
</VirtualHost>
EOF
```

### Step 4 — Enable and activate

```bash
sudo a2dissite 000-default.conf
sudo a2ensite sticky-lb.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Step 5 — Test sticky sessions with curl

**First request (no cookie — Apache assigns a route):**

```bash
curl -v http://localhost 2>&1 | grep -i "set-cookie\|Response"
```

You should see a `Set-Cookie` header like `ROUTEID=.server1` or `ROUTEID=.server2`.

**Subsequent requests with the cookie (should always reach the same backend):**

```bash
# Sticky to server1
for i in $(seq 1 5); do
    echo "Request $i: $(curl -s -b "ROUTEID=.server1" http://localhost)"
done
```

**Expected output:**

```
Request 1: Response from Backend 1 (port 9001)
Request 2: Response from Backend 1 (port 9001)
Request 3: Response from Backend 1 (port 9001)
Request 4: Response from Backend 1 (port 9001)
Request 5: Response from Backend 1 (port 9001)
```

**Test with server2 route:**

```bash
for i in $(seq 1 5); do
    echo "Request $i: $(curl -s -b "ROUTEID=.server2" http://localhost)"
done
```

**Expected output:**

```
Request 1: Response from Backend 2 (port 9002)
Request 2: Response from Backend 2 (port 9002)
Request 3: Response from Backend 2 (port 9002)
Request 4: Response from Backend 2 (port 9002)
Request 5: Response from Backend 2 (port 9002)
```

### Configuration Explanation

| Directive | Purpose |
|-----------|---------|
| `route=server1` / `route=server2` | Assigns a unique route identifier to each backend. This is appended to the sticky cookie value. |
| `ROUTEID` cookie | A cookie set on the client that stores the route (e.g., `.server1`). On subsequent requests, Apache reads this cookie to route to the same backend. |
| `Header add Set-Cookie "ROUTEID=..."` | Instructs Apache to send a `Set-Cookie` header to the client on the first request (or when the route changes). |
| `env=BALANCER_ROUTE_CHANGED` | Only sets the cookie when the route actually changes (e.g., first visit or failover). |
| `stickysession=ROUTEID` | Tells the balancer which cookie name to look for when determining session affinity. |

### Step 6 — Cleanup

```bash
# Stop both Python backends (Ctrl+C in their terminals)

sudo a2dissite sticky-lb.conf
sudo a2ensite 000-default.conf
sudo systemctl reload apache2
sudo rm /etc/apache2/sites-available/sticky-lb.conf
rm -rf /tmp/sticky1 /tmp/sticky2
```
