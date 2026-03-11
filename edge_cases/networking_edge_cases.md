[🏠 Home](../README.md) · [Edge Cases](.)

# 🌐 Networking — Edge Cases (110)

> **Audience:** DevOps engineers, network admins, and SREs dealing with production networking.  
> **Coverage:** TCP/IP quirks, DNS failures, firewall pitfalls, TLS edge cases, container networking, and cloud networking surprises.

---

## Table of Contents

1. [TCP/IP & Socket Layer (EC-001–015)](#tcpip--socket-layer)
2. [DNS & Name Resolution (EC-016–030)](#dns--name-resolution)
3. [Firewall & iptables/nftables (EC-031–045)](#firewall--iptablesnftables)
4. [TLS/SSL & Certificates (EC-046–058)](#tlsssl--certificates)
5. [Load Balancers & Proxies (EC-059–070)](#load-balancers--proxies)
6. [Container & Kubernetes Networking (EC-071–085)](#container--kubernetes-networking)
7. [VPN & Tunneling (EC-086–095)](#vpn--tunneling)
8. [Network Performance & Diagnostics (EC-096–105)](#network-performance--diagnostics)
9. [Cloud Networking (EC-106–110)](#cloud-networking)
10. [Production FAQ](#production-faq)

---

## TCP/IP & Socket Layer

### EC-001 — SYN Flood Exhausts Backlog Queue
**Category:** TCP | **Severity:** Critical | **Env:** Prod

**Scenario:** Attacker sends SYN packets without completing the handshake. `tcp_syn_backlog` fills. Legitimate clients get connection refused.

```
  Normal 3-way handshake:
  Client → [SYN] → Server  (enters SYN_RCVD)
  Client ← [SYN-ACK] ← Server
  Client → [ACK] → Server  (moves to ESTABLISHED)

  SYN Flood:
  Attacker → [SYN] → Server  (never sends final ACK)
  Attacker → [SYN] → Server  ← backlog fills up
  Attacker → [SYN] → Server
  LegitClient → [SYN] → Server → REFUSED (backlog full)
```

**Detection:**
```bash
ss -tan state syn-recv | wc -l   # count half-open connections
netstat -s | grep "SYNs to LISTEN"
```

**Fix:**
```bash
# Enable SYN cookies (server generates stateless SYN-ACK):
sysctl -w net.ipv4.tcp_syncookies=1
sysctl -w net.ipv4.tcp_max_syn_backlog=8192
sysctl -w net.ipv4.tcp_synack_retries=2
```

> ⚠️ **DevOps Gotcha:** `tcp_syncookies` is enabled by default on most modern distros, but `tcp_max_syn_backlog` is often too low (1024). For high-traffic services, set it to at least 4096–8192.

---

### EC-002 — TCP RST Injected by Network Appliance Mid-Session
**Category:** TCP | **Severity:** High | **Env:** Prod

**Scenario:** Long-running SQL queries, file transfers, or SSH sessions drop suddenly. Application shows "Connection reset by peer". No server-side crash.

**What causes it:** Deep Packet Inspection (DPI) firewall or IDS/IPS injects a RST packet to tear down "suspicious" or idle connections.

**Detection:**
```bash
tcpdump -i eth0 -n 'tcp[tcpflags] & (tcp-rst) != 0'
# Look for RSTs from IPs that aren't the client or server
```

> ⚠️ **DevOps Gotcha:** AWS ELB idle timeout (default 60s), Azure Load Balancer (4 min), and on-prem firewalls all send RSTs for idle TCP connections. Always set application-level keepalives shorter than the firewall timeout.

---

### EC-003 — FIN_WAIT2 Connections Accumulate Indefinitely
**Category:** TCP | **Severity:** High | **Env:** Prod

**Scenario:** Server sends FIN to close connection. Client acknowledges (ACK) but never sends its own FIN. Server stays in `FIN_WAIT2` forever. These accumulate and consume file descriptors.

**Detection:**
```bash
ss -tan state fin-wait-2 | wc -l
sysctl net.ipv4.tcp_fin_timeout   # default 60 seconds
```

**Fix:**
```bash
# Reduce FIN_WAIT2 timeout:
sysctl -w net.ipv4.tcp_fin_timeout=15
```

---

### EC-004 — ESTABLISHED Connections Not Using Keepalives, Silently Drop
**Category:** TCP | **Severity:** High | **Env:** Prod

**Scenario:** Database connection pool holds 100 ESTABLISHED connections. Firewall silently kills idle connections after 5 minutes. Application gets "broken pipe" error only when it tries to use the connection.

**Detection:**
```bash
# Check connection keepalive settings:
sysctl net.ipv4.tcp_keepalive_time    # default 7200s (2 hours!)
sysctl net.ipv4.tcp_keepalive_intvl  # default 75s
sysctl net.ipv4.tcp_keepalive_probes # default 9
```

**Fix:**
```bash
# Reduce to detect dead connections faster:
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=6
# Also configure at application level (more reliable):
# JDBC: autoReconnect=true, testOnBorrow=true
# SQLAlchemy: pool_pre_ping=True
```

> ⚠️ **DevOps Gotcha:** AWS NAT Gateway kills idle TCP flows after **350 seconds**. Default Linux TCP keepalive is 2 hours. Application will always see dead connections when going through NAT. Always use application-level connection validation.

---

### EC-005 — Nagle Algorithm Causes Latency in Interactive Applications
**Category:** TCP | **Severity:** Medium | **Env:** Prod

**Scenario:** SSH or remote desktop feels sluggish. Keystrokes take 200ms+ to appear. Nagle algorithm batches small packets, causing delay.

**Detection:**
```bash
# Test if TCP_NODELAY is set on a socket:
ss -i dst <ip> | grep nodelay
```

**Fix:**
```bash
# For SSH (already has TCP_NODELAY by default)
# For custom apps, set TCP_NODELAY socket option:
# setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
# Or at OS level for all connections (not recommended):
sysctl -w net.ipv4.tcp_low_latency=1  # deprecated in newer kernels
```

---

### EC-006 — `CLOSE_WAIT` Connections Indicate Application Bug
**Category:** TCP | **Severity:** High | **Env:** Prod

**Scenario:** Many connections in CLOSE_WAIT state. Server received FIN from client but application never closed the socket.

```
  Normal close:
  Client → FIN → Server  (server enters CLOSE_WAIT, sends ACK)
  Server closes socket → sends FIN → Client
  
  Bug: Server in CLOSE_WAIT but never closes socket
  → connection leaks → fd exhaustion
```

**Detection:**
```bash
ss -tan state close-wait | wc -l
# If this grows over time, it's an application leak
```

> ⚠️ **DevOps Gotcha:** CLOSE_WAIT accumulation is ALWAYS an application bug (forgetting to call `close()`). It cannot be fixed at the OS level. Check connection pooling code, HTTP client library, and make sure response bodies are fully consumed.

---

### EC-007 — IP Fragmentation Reassembly Timeout Drops Packets
**Category:** TCP/IP | **Severity:** High | **Env:** Prod

**Scenario:** Large UDP packets (DNS over 512 bytes, IPsec, NFS) get fragmented. Some fragments arrive, others are dropped. After `ipfrag_time` (30s default), partial packet is discarded silently.

**Detection:**
```bash
cat /proc/net/snmp | grep -A1 "Ip:"   # look for ReasmFails
nstat -az | grep IpReasmFails
```

---

### EC-008 — TCP Window Scaling Disabled by Broken Firewall
**Category:** TCP | **Severity:** High | **Env:** Prod

**Scenario:** High-latency WAN connection (e.g., trans-oceanic) has terrible throughput despite gigabit link. TCP window size limited to 65KB because a middlebox strips TCP window scale option.

```
  Bandwidth-Delay Product = Bandwidth × RTT
  100 Mbps × 100ms = 1.25 MB buffer needed
  Without window scaling: max 65KB → 80% throughput loss!
```

**Detection:**
```bash
ss -i dst <remote-ip> | grep rcv_space   # check window size
# Should be > 65535 if window scaling is working
```

---

### EC-009 — UDP Source Port 0 Breaks NAT Traversal
**Category:** UDP | **Severity:** Medium | **Env:** Prod

**Scenario:** Application binds UDP socket to port 0 (OS assigns ephemeral). Some NAT implementations and firewalls block outbound UDP from source port 0.

---

### EC-010 — Multicast Traffic Leaks Out of Intended Network Segment
**Category:** IP | **Severity:** Medium | **Env:** Prod

**Scenario:** Application using multicast (mDNS 224.0.0.251, Kubernetes cloud controller) floods multicast traffic to switches that forward it everywhere.

**Detection:**
```bash
tcpdump -i eth0 -n multicast
# Look for 224.x.x.x or ff02::x traffic
```

---

### EC-011 — Loopback IP Routing Mismatch With Source Routing
**Category:** IP | **Severity:** Medium | **Env:** Prod

**Scenario:** Application connects to `127.0.0.1:8080`. Response comes from `192.168.1.10` (the actual IP). Strict source routing firewall drops the response.

---

### EC-012 — `ip route add` Without Metric Overrides Default Route
**Category:** Routing | **Severity:** Critical | **Env:** Prod

**Scenario:** `ip route add 0.0.0.0/0 via 10.0.0.1` adds a second default route without specifying metric. Both default routes are active. Traffic takes unpredictable path.

**Detection:**
```bash
ip route show   # shows multiple default routes
ip route get 8.8.8.8   # shows which route is actually used
```

**Fix:**
```bash
ip route del default via 10.0.0.1   # remove unwanted route
# Or specify metric:
ip route add default via 10.0.0.2 metric 200
```

---

### EC-013 — ECMP Load Balancing Breaks Stateful Sessions
**Category:** Routing | **Severity:** High | **Env:** Prod

**Scenario:** Equal-Cost Multi-Path routing splits traffic across multiple uplinks. Stateful connections get split across paths, causing RST or timeout.

**Detection:**
```bash
ip route show   # multiple routes with same metric
traceroute --sport=12345 <dest>   # shows different paths for different sources
```

---

### EC-014 — IP Address Added to Wrong Interface Survives Reboot
**Category:** IP | **Severity:** Medium | **Env:** Prod

**Scenario:** `ip addr add 10.0.0.5/24 dev eth1` works at runtime. After reboot, the IP is gone (it wasn't persisted). Meanwhile, `eth0` config file has the same IP causing conflict.

**Fix:**
```bash
# Always persist via network config files:
# RHEL: /etc/sysconfig/network-scripts/ifcfg-eth1
# Ubuntu: /etc/netplan/01-netcfg.yaml
# Debian: /etc/network/interfaces
```

---

### EC-015 — Raw Socket Packets Bypass iptables INPUT/OUTPUT Chains
**Category:** IP | **Severity:** High | **Env:** Prod

**Scenario:** Application using raw sockets (e.g., custom ping tool, packet generator) can send/receive packets that bypass iptables INPUT and OUTPUT chains (except PREROUTING/POSTROUTING).

> ⚠️ **DevOps Gotcha:** Container runtimes and VPN tools that use raw sockets bypass your iptables OUTPUT rules. You need `iptables -t mangle` PREROUTING/POSTROUTING rules to catch raw socket traffic.

---

## DNS & Name Resolution

### EC-016 — Negative DNS Caching (NXDOMAIN) Outlasts Record Recreation
**Category:** DNS | **Severity:** High | **Env:** Prod

**Scenario:** DNS record accidentally deleted. After recreating it, some clients still get NXDOMAIN because they cached the negative response. Negative TTL is set by SOA minimum TTL.

**Detection:**
```bash
dig @8.8.8.8 host.example.com   # check upstream
dig @internal-dns host.example.com   # check local resolver
# Look for AUTHORITY SECTION: SOA record shows negative TTL
```

**Fix:**
```bash
# Short term: flush client resolver cache
systemd-resolve --flush-caches
# Or restart nscd:
systemctl restart nscd
```

> ⚠️ **DevOps Gotcha:** Set SOA negative TTL to a low value (60–300s) for zones where records change frequently. A SOA with `minimum 86400` means NXDOMAIN is cached for 24 hours.

---

### EC-017 — DNS Round-Robin Does Not Equal Load Balancing
**Category:** DNS | **Severity:** Medium | **Env:** Prod

**Scenario:** Three A records for `api.example.com`. Expecting equal distribution. Clients cache one IP and reuse it for the entire TTL period. One backend gets all traffic.

```
  DNS Round-Robin (WRONG mental model):
  Query 1 → 10.0.0.1  ← client caches this for TTL seconds
  Query 2 → 10.0.0.2  ← different client
  
  Reality: Client with 10.0.0.1 sends ALL traffic to 10.0.0.1 until TTL expires!
```

> ⚠️ **DevOps Gotcha:** DNS round-robin is client-side, stochastic, and TTL-dependent. Use a real load balancer (HAProxy, AWS ALB) for actual load distribution. DNS is for failover, not balancing.

---

### EC-018 — `resolv.conf` Overwritten by DHCP After Manual Edit
**Category:** DNS | **Severity:** Medium | **Env:** Prod

**Scenario:** Edit `/etc/resolv.conf` to add a custom DNS server. Next DHCP renewal overwrites your changes.

**Fix:**
```bash
# On systemd-resolved systems:
# Edit /etc/systemd/resolved.conf:
[Resolve]
DNS=8.8.8.8 1.1.1.1
FallbackDNS=8.8.4.4
# On NetworkManager systems:
nmcli con mod "connection-name" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con mod "connection-name" ipv4.ignore-auto-dns yes
# Make resolv.conf immutable (last resort):
chattr +i /etc/resolv.conf
```

---

### EC-019 — DNS Search Domain Causes 5-Second Delay Per Query
**Category:** DNS | **Severity:** High | **Env:** Prod

**Scenario:** With multiple search domains, each unresolvable domain must time out before the next is tried. With 5 search domains, `nslookup` for an external name takes 25+ seconds.

**Detection:**
```bash
cat /etc/resolv.conf | grep search   # count search domains
time nslookup external-site.com      # see total query time
```

**Fix:**
```bash
# Use FQDN (trailing dot) to avoid search domain expansion:
curl https://api.external.com./endpoint
# Or reduce search domains to minimum necessary
# Or set options attempts:1 in resolv.conf
```

---

### EC-020 — Split-Horizon DNS Returns Wrong IP via VPN
**Category:** DNS | **Severity:** High | **Env:** Prod

**Scenario:** VPN connected. `db.internal.company.com` should resolve to `10.0.1.5` (internal). Instead resolves to public IP because DNS query goes to public resolver, not internal DNS.

**Detection:**
```bash
dig db.internal.company.com @<internal-dns-server>   # correct
dig db.internal.company.com @8.8.8.8                  # wrong/fails
```

> ⚠️ **DevOps Gotcha:** VPN clients must push DNS server settings to clients. Split-DNS configuration pushes only internal domain queries to internal DNS. Without it, internal hostnames resolve incorrectly.

---

### EC-021 — DNS TTL Too High — Change Doesn't Propagate for Hours
**Category:** DNS | **Severity:** High | **Env:** Prod

**Scenario:** Change production A record for load failover. TTL is 86400 (24 hours). Old servers continue receiving traffic for 24 hours.

> ⚠️ **DevOps Gotcha:** Before planned migrations, reduce TTL to 60–300 seconds at least 24 hours in advance (allow current TTL to expire first). After migration, set TTL back to 3600 for normal operation.

---

### EC-022 — CNAME Cannot Coexist With Other Records at Zone Apex
**Category:** DNS | **Severity:** High | **Env:** Prod

**Scenario:** Trying to create `CNAME @ → mylb.example.com` for root domain. DNS protocol forbids CNAME at zone apex (@ record) because it would conflict with SOA and NS records.

**Fix:**
```bash
# Use ALIAS or ANAME records (Route53 Alias, Cloudflare CNAME flattening)
# These behave like CNAME but are valid at zone apex
```

---

### EC-023 — DNS over TCP Required for Large Responses (> 512 bytes)
**Category:** DNS | **Severity:** Medium | **Env:** Prod

**Scenario:** DNS response truncated because firewall blocks TCP port 53. DNSSEC records or large TXT records exceed 512-byte UDP limit. Resolver falls back to TCP but firewall blocks it.

**Detection:**
```bash
dig +notcp example.com DNSKEY   # test UDP-only
dig +tcp example.com DNSKEY     # test TCP
# If UDP truncates but TCP works: firewall blocking TCP 53
```

---

### EC-024 — DNS Cache Poisoning via Predictable Transaction IDs
**Category:** DNS | **Severity:** Critical | **Env:** Prod

**Scenario:** DNS resolver using predictable transaction IDs and no source port randomization is vulnerable to cache poisoning (Kaminsky attack).

**Detection:**
```bash
dig +short porttest.dns-oarc.net TXT   # tests resolver port randomization
```

> ⚠️ **DevOps Gotcha:** Use `DNSSEC` for zone signing and `DNS-over-TLS` or `DNS-over-HTTPS` for resolvers to prevent cache poisoning. Ensure your resolver randomizes source ports.

---

### EC-025 — `/etc/hosts` Doesn't Support Wildcards
**Category:** DNS | **Severity:** Low | **Env:** Dev

**Scenario:** Developer adds `*.local.dev 127.0.0.1` to `/etc/hosts` expecting all subdomains to resolve. Wildcards are not supported in `/etc/hosts`.

**Fix:**
```bash
# Use dnsmasq for wildcard resolution:
# /etc/dnsmasq.conf:
address=/.local.dev/127.0.0.1
systemctl restart dnsmasq
```

---

### EC-026 — DNS Lookup Timeout in Container Due to Firewall Blocking UDP 53
**Category:** DNS | **Severity:** Critical | **Env:** Prod (K8s)

**Scenario:** Container pods experience 5-second delays on every DNS lookup. UDP port 53 is blocked intermittently. Application requests time out waiting for DNS.

**Detection:**
```bash
kubectl exec -it pod -- nslookup kubernetes.default
# If it takes > 1 second, DNS is broken
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

### EC-027 — PTR Records Missing Causes Auth Log Spam and Delays
**Category:** DNS | **Severity:** Low | **Env:** Prod

**Scenario:** `sshd` with `UseDNS yes` does reverse lookup on connecting client IP. No PTR record exists — lookup times out. Each SSH connection is delayed 5–30 seconds.

**Fix:**
```bash
# In /etc/ssh/sshd_config:
UseDNS no
# Or add PTR records for all IP ranges used by connecting clients
```

---

### EC-028 — Docker's Embedded DNS Server Fails Under High Load
**Category:** DNS | **Severity:** High | **Env:** Prod

**Scenario:** Docker container uses `127.0.0.11` as DNS server (embedded). Under high query load, the embedded server drops packets causing intermittent DNS failures.

**Fix:**
```bash
# Use external DNS or add ndots option:
# In docker-compose.yml:
dns:
  - 8.8.8.8
  - 1.1.1.1
dns_search: .
```

---

### EC-029 — Kubernetes CoreDNS NXDOMAIN Storm From Bad Wildcards
**Category:** DNS | **Severity:** High | **Env:** Prod (K8s)

**Scenario:** Application code makes DNS queries for random/dynamic hostnames. Each NXDOMAIN floods CoreDNS, which throttles real DNS lookups.

**Detection:**
```bash
kubectl exec -n kube-system <coredns-pod> -- cat /tmp/...
# Check CoreDNS metrics:
# coredns_dns_request_count_total{rcode="NXDOMAIN"}
```

---

### EC-030 — TTL of 0 Causes Thundering Herd on DNS Server
**Category:** DNS | **Severity:** High | **Env:** Prod

**Scenario:** DNS record with TTL=0 means every request makes a fresh DNS lookup. At 10,000 req/s, this means 10,000 DNS queries/s — overwhelming the DNS server.

> ⚠️ **DevOps Gotcha:** TTL=0 is sometimes used for "real-time" failover, but it creates crushing load on DNS servers. Use TTL=60 as minimum and health-check-driven DNS failover (Route53 health checks) instead.

---

## Firewall & iptables/nftables

### EC-031 — iptables Rules Applied in Wrong Order Allow Bypassed Traffic
**Category:** Firewall | **Severity:** Critical | **Env:** Prod

**Scenario:** Administrator adds `ACCEPT` rule for a management IP at the bottom of the chain. A `DROP all` rule above it catches traffic first.

```
  iptables rules are evaluated TOP to BOTTOM:
  Rule 1: DROP all  ← traffic matches HERE and is dropped
  Rule 2: ACCEPT from 10.0.0.5   ← NEVER reached!
```

**Fix:**
```bash
# Insert rule at top with -I (insert) not -A (append):
iptables -I INPUT 1 -s 10.0.0.5 -j ACCEPT
# Check order:
iptables -L INPUT -n --line-numbers
```

---

### EC-032 — NAT Masquerade Rule Not Applied After Interface Rename
**Category:** Firewall | **Severity:** High | **Env:** Prod

**Scenario:** `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`. After kernel upgrade, `eth0` renamed to `enp3s0`. NAT stops working silently.

**Fix:**
```bash
# Use interface-independent approach:
iptables -t nat -A POSTROUTING -o ! lo -j MASQUERADE
# Or use nftables with dynamic interface matching
```

---

### EC-033 — `conntrack` Sees Untracked Packets as INVALID
**Category:** Firewall | **Severity:** High | **Env:** Prod

**Scenario:** `iptables -A INPUT -m state --state INVALID -j DROP` drops legitimate packets when conntrack table is full or after system restart.

---

### EC-034 — IPv6 Traffic Not Filtered by iptables (ip6tables Separate)
**Category:** Firewall | **Severity:** Critical | **Env:** Prod

**Scenario:** Security team adds strict iptables rules for IPv4. Attacker accesses service via IPv6. `ip6tables` is separate and was never configured.

**Detection:**
```bash
ip6tables -L -n    # check IPv6 rules (often empty!)
ip -6 addr         # check if IPv6 is active
```

**Fix:**
```bash
# Use nftables which handles both IPv4/IPv6 in one rule set
# Or mirror all iptables rules with ip6tables
# Or disable IPv6 if not needed:
sysctl -w net.ipv6.conf.all.disable_ipv6=1
```

---

### EC-035 — iptables `REJECT` vs `DROP` — Stealth vs Correct Behavior
**Category:** Firewall | **Severity:** Medium | **Env:** Prod

**Scenario:** Using `DROP` instead of `REJECT` causes application timeout (TCP connection hangs for full timeout period) vs immediate error.

```
  DROP:   Client waits for full timeout (20s–2min) → poor user experience
  REJECT: Client gets immediate ICMP Port Unreachable → fast failure
  
  Security implication:
  DROP: port scanner must wait for timeout → slower scans (stealth)
  REJECT: port scanner gets immediate confirmation → faster scans
```

> ⚠️ **DevOps Gotcha:** For internal traffic, use REJECT for faster failure detection. For external-facing firewalls, DROP is common to slow down port scanners. Use REJECT within Kubernetes network policies for faster pod communication errors.

---

### EC-036 — UFW Rules Break After Docker Install
**Category:** Firewall | **Severity:** Critical | **Env:** Prod

**Scenario:** UFW configured to block port 8080. Docker exposes port 8080. Docker bypasses UFW by directly adding iptables rules in DOCKER chain. Port 8080 is accessible despite UFW rule.

```
  UFW chain order:
  DOCKER-USER → DOCKER-ISOLATION-STAGE-1 → DOCKER
  ↑ Your rules go here (UFW = INPUT chain, not FORWARD)
  
  Docker adds rules to FORWARD chain, bypassing UFW INPUT rules!
```

**Fix:**
```bash
# Add rules to DOCKER-USER chain instead:
iptables -I DOCKER-USER -p tcp --dport 8080 ! -s 10.0.0.0/8 -j DROP
# Or use Docker bind to localhost only:
docker run -p 127.0.0.1:8080:8080 myapp
```

---

### EC-037 — `iptables-save` Output Doesn't Restore Load Order Correctly
**Category:** Firewall | **Severity:** Medium | **Env:** Prod

**Scenario:** `iptables-save > rules.v4` captures rules. `iptables-restore < rules.v4` on a different system creates the same rules but they interact differently with existing kernel modules.

---

### EC-038 — Rate Limiting With `--limit` Uses Token Bucket — Burst Allowed
**Category:** Firewall | **Severity:** Medium | **Env:** Prod

**Scenario:** `iptables -m limit --limit 10/s --limit-burst 100`. Attacker sends 100 packets instantly (burst allowed), then 10/s forever. Rate limit isn't as strict as assumed.

---

### EC-039 — nftables and iptables Coexistence Causes Conflicts
**Category:** Firewall | **Severity:** High | **Env:** Prod

**Scenario:** System has both `nftables` and `iptables` (via `iptables-nft` shim). Rules added via iptables commands appear in nftables tables. Double processing causes unexpected behavior.

**Detection:**
```bash
nft list ruleset    # shows ALL rules including iptables-nft translations
iptables -L -n      # iptables view
```

---

### EC-040 — PREROUTING DNAT Applied Before Routing Decision
**Category:** Firewall | **Severity:** Medium | **Env:** Prod

**Scenario:** DNAT rule in PREROUTING changes destination before routing table is consulted. This allows redirecting traffic to a local process even if the destination IP is "remote".

```
  Packet flow with NAT:
  [PREROUTING] → [Routing Decision] → [INPUT or FORWARD]
       ↑
  DNAT happens HERE — destination changes before routing!
  This is how port forwarding and transparent proxies work.
```

---

### EC-041 — `hashlimit` Module Not Available in All Kernels
**Category:** Firewall | **Severity:** Low | **Env:** Prod

**Scenario:** `iptables -m hashlimit` fails on some minimal kernel builds (containers with restricted capabilities).

---

### EC-042 — Stateful Rules in FORWARD Chain Not Cleaning Up
**Category:** Firewall | **Severity:** Medium | **Env:** Prod

**Scenario:** `iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT`. Conntrack entries for closed connections accumulate if `tcp_timeout_close_wait` isn't tuned.

---

### EC-043 — iptables Rule Counting Misleads About Real Traffic
**Category:** Firewall | **Severity:** Low | **Env:** Prod

**Scenario:** `iptables -L -n -v` shows packet counter as 0 on a rule you expect to match. Rule is correct but earlier rule already matches the traffic.

---

### EC-044 — Output Chain Doesn't Apply to Forwarded Traffic
**Category:** Firewall | **Severity:** High | **Env:** Prod

**Scenario:** `iptables -A OUTPUT -d malicious-ip -j DROP` intended to block traffic to a malicious IP. Containers use FORWARD chain, not OUTPUT. Container traffic to malicious-ip goes through.

---

### EC-045 — `ip_forward` Disabled After iptables Flush
**Category:** Firewall | **Severity:** Critical | **Env:** Prod

**Scenario:** Kubernetes node stops routing pod traffic after iptables flush. `net.ipv4.ip_forward` is still enabled, but FORWARD chain default policy became DROP after flush.

**Fix:**
```bash
iptables -P FORWARD ACCEPT
# For Kubernetes, kube-proxy will restore needed rules:
systemctl restart kube-proxy
```

---

## TLS/SSL & Certificates

### EC-046 — Certificate Chain Incomplete — Works in Browser, Fails in API Client
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Browser shows "secure" because it has intermediate CAs cached. `curl`, `wget`, Java, Python `requests` fail with "unable to verify the first certificate" because server doesn't send intermediate cert.

**Detection:**
```bash
openssl s_client -connect host:443 -showcerts 2>/dev/null | \
  openssl x509 -noout -text | grep "Issuer"
# Chain should be: leaf → intermediate → root
```

**Fix:**
```bash
# Configure nginx/Apache to include intermediate cert:
# cat server.crt intermediate.crt > fullchain.crt
# ssl_certificate /etc/ssl/certs/fullchain.crt;  ← nginx
```

---

### EC-047 — TLS 1.0/1.1 Still Enabled — PCI DSS Violation
**Category:** TLS | **Severity:** Critical | **Env:** Prod

**Scenario:** Server accepts TLS 1.0 connections. PCI DSS 3.2 requires TLS 1.2+ minimum. Security scan finds vulnerability but engineering says "it's just for legacy browsers".

**Detection:**
```bash
nmap --script ssl-enum-ciphers -p 443 host
testssl.sh host:443
openssl s_client -tls1 -connect host:443   # if it connects, TLS1.0 is enabled
```

**Fix:**
```bash
# nginx:
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
```

---

### EC-048 — Certificate Expiry Not Caught Because HTTPS Returns 200
**Category:** TLS | **Severity:** Critical | **Env:** Prod

**Scenario:** HTTP monitoring checks port 443 and gets a 200 response — monitor shows green. Certificate expired 3 days ago. Browser users are getting certificate errors.

**Detection:**
```bash
echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -dates
# Automate expiry check:
openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -checkend 604800
# exit 1 if cert expires within 7 days (604800 seconds)
```

> ⚠️ **DevOps Gotcha:** Always monitor certificate expiry SEPARATELY from HTTP status. Use Prometheus `blackbox_exporter` with `probe_ssl_earliest_cert_expiry` metric, or AWS Certificate Manager with CloudWatch alarms.

---

### EC-049 — Wildcard Certificate Doesn't Cover Sub-Subdomains
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Wildcard cert `*.example.com` covers `api.example.com` but NOT `v2.api.example.com`. Application deployed at two levels deep breaks.

```
  *.example.com covers:
  ✅ api.example.com
  ✅ www.example.com
  ❌ v2.api.example.com   ← would need *.api.example.com
```

---

### EC-050 — Self-Signed Cert in Internal CA Chain Breaks Third-Party Libraries
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Internal CA used for all internal services. Java applications (default JVM truststore), Python requests (uses certifi), and Node.js all have their own trust stores that don't include your internal CA.

**Fix:**
```bash
# Add internal CA to system trust store:
# Ubuntu:
cp internal-ca.crt /usr/local/share/ca-certificates/
update-ca-certificates
# RHEL:
cp internal-ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust

# For Java:
keytool -import -alias internal-ca -keystore $JAVA_HOME/lib/security/cacerts \
  -file internal-ca.crt -storepass changeit
```

---

### EC-051 — Let's Encrypt Rate Limits Hit During Rapid Deployments
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** CI/CD pipeline requests new Let's Encrypt cert for each deployment. Rate limit: 50 certificates per registered domain per week. Pipeline grinds to a halt.

> ⚠️ **DevOps Gotcha:** Reuse certificates across deployments. Store cert in S3/Vault and retrieve in deployment. Use wildcard certs (`*.env.example.com`) with DNS-01 challenge. Never generate new certs per deployment.

---

### EC-052 — Certificate Pinning Breaks After Legitimate Cert Rotation
**Category:** TLS | **Severity:** Critical | **Env:** Prod

**Scenario:** Mobile app or internal client has cert pinning. Certificate renewed (as planned). All pinned clients immediately fail to connect. Rollback impossible without updating all clients.

> ⚠️ **DevOps Gotcha:** Never pin the leaf certificate. Pin the intermediate CA or root CA. This survives routine leaf certificate rotation.

---

### EC-053 — OCSP Stapling Failure Causes 3-Second TLS Handshake Delay
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** OCSP stapling enabled but server fails to fetch OCSP response (firewall blocks outbound to CA's OCSP server). Browsers do OCSP check themselves, causing 3-second delay per new connection.

**Detection:**
```bash
openssl s_client -status -connect host:443   # check for OCSP response
# "OCSP response: no response sent" = stapling not working
```

---

### EC-054 — DHE Parameters Too Small (Logjam Attack)
**Category:** TLS | **Severity:** Critical | **Env:** Prod

**Scenario:** Server using 512-bit or 1024-bit Diffie-Hellman parameters, vulnerable to downgrade attack.

**Fix:**
```bash
# Generate strong DH parameters:
openssl dhparam -out /etc/ssl/dhparam.pem 4096
# nginx config:
ssl_dhparam /etc/ssl/dhparam.pem;
```

---

### EC-055 — SNI Required But Not Supported by Old Clients
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Server uses SNI to host multiple domains on one IP. Old clients (Python 2, Java 6, Windows XP) don't send SNI. Server serves wrong certificate.

---

### EC-056 — Certificate Subject Alternative Name Missing
**Category:** TLS | **Severity:** High | **Env:** Prod

**Scenario:** Certificate has correct CN (Common Name) but modern browsers (Chrome 58+) require Subject Alternative Name (SAN). Connection fails with "ERR_CERT_COMMON_NAME_INVALID".

**Detection:**
```bash
openssl x509 -in cert.crt -noout -text | grep -A3 "Subject Alternative"
```

---

### EC-057 — Mixed Content Blocks HTTPS Page Resources
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** Site served over HTTPS loads CSS/JS from `http://` URLs. Modern browsers block mixed content. Site appears broken even though HTTPS is working.

---

### EC-058 — HSTS Max-Age Too Long Locks Out HTTP-Only Environments
**Category:** TLS | **Severity:** Medium | **Env:** Prod

**Scenario:** `Strict-Transport-Security: max-age=31536000; includeSubDomains`. Development environment uses HTTP. Browsers that visited production remember HSTS and refuse HTTP connections to all subdomains.

> ⚠️ **DevOps Gotcha:** Only set `includeSubDomains` if ALL subdomains serve HTTPS. Use a shorter max-age (86400) while testing, then increase to 31536000 for production.

---

## Load Balancers & Proxies

### EC-059 — Health Check Passes But Backend Is Degraded
**Category:** LB | **Severity:** High | **Env:** Prod

**Scenario:** Load balancer health check hits `/health` endpoint which returns 200. Backend actually processing requests at 10x normal latency due to degraded database. Traffic keeps flowing to degraded backend.

> ⚠️ **DevOps Gotcha:** Implement deep health checks that validate actual backend dependencies (DB query, cache connectivity). Use response time thresholds in health checks, not just status codes.

---

### EC-060 — Session Stickiness (Affinity) Breaks After Backend Restart
**Category:** LB | **Severity:** High | **Env:** Prod

**Scenario:** Cookie-based session affinity sends user to same backend. Backend restarts, loses in-memory session data. User session invalidated — gets logged out.

> ⚠️ **DevOps Gotcha:** Never rely on in-memory sessions with multiple backends. Use external session storage (Redis, Memcached, database). Session affinity is a workaround, not a solution.

---

### EC-061 — Connection Draining Timeout Too Short
**Category:** LB | **Severity:** Medium | **Env:** Prod

**Scenario:** Backend being removed from rotation (deployment). LB connection drain timeout is 10s. Long-running requests (file uploads, reports) are terminated mid-flight.

**Fix:**
```bash
# AWS ALB: deregistration_delay.timeout_seconds = 300
# nginx upstream: fail_timeout=60s; keepalive_timeout 65;
# Match your longest expected request duration
```

---

### EC-062 — X-Forwarded-For Spoofed by Client
**Category:** LB/Proxy | **Severity:** High | **Env:** Prod

**Scenario:** Application trusts `X-Forwarded-For` header for rate limiting or geo-restriction. Client sends `X-Forwarded-For: 127.0.0.1` to bypass IP-based restrictions.

**Fix:**
```bash
# Only trust X-Forwarded-For from known proxy IPs
# nginx: set_real_ip_from 10.0.0.0/8; real_ip_header X-Forwarded-For;
# Never trust client-sent X-Forwarded-For directly
```

---

### EC-063 — LB Does Not Forward WebSocket Upgrades
**Category:** LB | **Severity:** High | **Env:** Prod

**Scenario:** WebSocket connections fail through load balancer. LB sees `Connection: upgrade` header but strips it (HTTP/1.1 hop-by-hop header).

**Fix:**
```nginx
# nginx proxy config for WebSocket:
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

---

### EC-064 — Proxy Protocol Adds Extra Header — Double-Processing
**Category:** LB | **Severity:** Medium | **Env:** Prod

**Scenario:** Enable PROXY protocol on HAProxy. Backend nginx also has PROXY protocol enabled. nginx receives PROXY protocol from both HAProxy and the TCP layer. IP parsing fails.

---

### EC-065 — HTTP/2 Backend Requires Different Load Balancer Config
**Category:** LB | **Severity:** Medium | **Env:** Prod

**Scenario:** Backend supports HTTP/2. LB configured for HTTP/1.1 only. LB downgrades protocol, losing multiplexing benefits and causing protocol negotiation errors.

---

### EC-066 — LB Keeps Unhealthy Backend in Rotation Due to Flapping
**Category:** LB | **Severity:** High | **Env:** Prod

**Scenario:** Backend alternates between healthy and unhealthy faster than the health check interval. LB oscillates backend in/out of rotation. Users get intermittent errors.

**Fix:**
```bash
# Require multiple consecutive health failures before removing:
# AWS Target Group: UnhealthyThresholdCount=3
# HAProxy: fall 3 rise 2  (3 fails to remove, 2 passes to restore)
```

---

### EC-067 — Reverse Proxy Rewrites Host Header — Backend Logs Wrong Host
**Category:** Proxy | **Severity:** Low | **Env:** Prod

**Scenario:** nginx reverse proxy. Backend application logs `Host: localhost` instead of the actual client's host. URL generation in app produces `http://localhost/path`.

**Fix:**
```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-Proto $scheme;
```

---

### EC-068 — Proxy Buffer Too Small Causes `502 Bad Gateway`
**Category:** Proxy | **Severity:** Medium | **Env:** Prod

**Scenario:** nginx proxy returns 502. Error log: "upstream sent too big header". Response headers from backend exceed nginx's default buffer size.

**Fix:**
```nginx
proxy_buffer_size 64k;
proxy_buffers 4 128k;
proxy_busy_buffers_size 256k;
```

---

### EC-069 — HTTPS Offload at LB Means Backend Doesn't Know Protocol
**Category:** LB | **Severity:** Medium | **Env:** Prod

**Scenario:** LB terminates TLS, forwards HTTP to backend. Backend redirects to `http://` instead of `https://` because it doesn't know original request was HTTPS.

**Fix:**
```nginx
# Set X-Forwarded-Proto and have app check it for redirects
# Or configure app to assume HTTPS:
# Django: SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
# Express: app.set('trust proxy', 1)
```

---

### EC-070 — HTTP Keep-Alive Between LB and Backend Not Enabled
**Category:** LB | **Severity:** Medium | **Env:** Prod

**Scenario:** LB opens new TCP connection to backend for each request. At 1000 req/s, 1000 new TCP connections/s to backend — TCP handshake overhead dominates.

**Fix:**
```nginx
upstream backend {
    server backend:8080;
    keepalive 32;    # maintain 32 idle keep-alive connections
}
proxy_http_version 1.1;
proxy_set_header Connection "";
```

---

## Container & Kubernetes Networking

### EC-071 — Pod CIDR Overlaps With Corporate Network
**Category:** K8s Networking | **Severity:** Critical | **Env:** Prod

**Scenario:** Kubernetes cluster uses `10.96.0.0/12` for pod CIDR. Corporate VPN also uses `10.100.0.0/16`. Traffic to corporate services goes into the pod network.

```
  Corporate network: 10.100.0.0/16
  K8s pod CIDR:      10.96.0.0/12 → covers 10.96.0.0 - 10.111.255.255
  10.100.0.0/16 is WITHIN 10.96.0.0/12 ← OVERLAP! Corporate traffic
  gets routed to pods instead of VPN!
```

> ⚠️ **DevOps Gotcha:** Plan CIDR ranges carefully before cluster creation. Changing CIDRs after cluster creation requires full teardown. Use `100.64.0.0/10` (IANA Shared Address Space) or document all CIDR allocations in your network plan.

---

### EC-072 — NodePort Service Exposes All Cluster Nodes
**Category:** K8s Networking | **Severity:** High | **Env:** Prod

**Scenario:** `type: NodePort` service accessible on port 30080 on ALL cluster nodes, even nodes without the target pod. Traffic is kube-proxy-forwarded. Security teams didn't expect this.

> ⚠️ **DevOps Gotcha:** NodePort services open ports on EVERY node's host network stack. Secure your cluster nodes' firewall to restrict NodePort range (30000-32767) to only trusted sources.

---

### EC-073 — Service DNS Resolution Works But Direct Pod IP Doesn't
**Category:** K8s Networking | **Severity:** Medium | **Env:** Prod (K8s)

**Scenario:** `curl http://myservice.default.svc.cluster.local` works. `curl http://10.100.1.5` (pod IP) fails. NetworkPolicy is applied to the pod but the service IP bypasses NetworkPolicy.

---

### EC-074 — iptables Mode kube-proxy Breaks With 10,000+ Services
**Category:** K8s Networking | **Severity:** High | **Env:** Prod

**Scenario:** Large cluster with many services sees increasing latency. kube-proxy in iptables mode creates O(n) rules per service. With 10,000 services, iptables rule processing is O(n) per packet.

**Fix:**
```bash
# Switch to IPVS mode:
kubectl edit configmap kube-proxy -n kube-system
# Set mode: ipvs
# IPVS uses hash tables: O(1) lookup regardless of service count
```

---

### EC-075 — Ingress Controller Not Processing Namespace-Scoped Annotations
**Category:** K8s Networking | **Severity:** Medium | **Env:** Prod

**Scenario:** Different ingress controllers (nginx vs traefik) have different annotation syntax. Copying config between clusters silently ignores annotations for the wrong controller.

---

### EC-076 — NetworkPolicy Denies All Egress — DNS Breaks
**Category:** K8s Networking | **Severity:** Critical | **Env:** Prod

**Scenario:** NetworkPolicy with `policyTypes: [Egress]` and no egress rules denies ALL outbound including DNS (port 53 to CoreDNS). All pod DNS lookups fail.

**Fix:**
```yaml
# Always include DNS exception in egress policies:
egress:
- ports:
  - port: 53
    protocol: UDP
  - port: 53
    protocol: TCP
```

---

### EC-077 — Host Network Pod Bypasses All Network Policies
**Category:** K8s Networking | **Severity:** High | **Env:** Prod

**Scenario:** Pod with `hostNetwork: true` bypasses Kubernetes network policies entirely. Security policies have no effect.

> ⚠️ **DevOps Gotcha:** Use PSP (deprecated) or OPA/Gatekeeper to prevent unauthorized use of `hostNetwork: true`. Only infrastructure pods (CNI, monitoring) should use host networking.

---

### EC-078 — Container CNI Plugin Fails — All Pod Networking Broken
**Category:** K8s Networking | **Severity:** Critical | **Env:** Prod

**Scenario:** Flannel/Calico/Weave pod crashes. New pods cannot get IPs. Existing pods lose routing to each other. Node shows NotReady if CNI is required for kubelet.

**Detection:**
```bash
kubectl get pods -n kube-system | grep -E "calico|flannel|weave"
kubectl describe node <node> | grep -A10 "Conditions"
ls /etc/cni/net.d/    # check CNI config files
```

---

### EC-079 — Docker Bridge Network Conflicts With Physical Network
**Category:** Container Networking | **Severity:** High | **Env:** Prod

**Scenario:** Docker's default bridge `docker0` uses `172.17.0.0/16`. Corporate network happens to use the same range. Containers cannot reach external IPs in that range.

**Fix:**
```json
// /etc/docker/daemon.json:
{
  "bip": "192.168.200.1/24",
  "fixed-cidr": "192.168.200.0/25",
  "default-address-pools": [{"base": "192.168.100.0/16", "size": 24}]
}
```

---

### EC-080 — Service Mesh Sidecar Intercepts Traffic and Causes Double mTLS
**Category:** K8s Networking | **Severity:** High | **Env:** Prod

**Scenario:** Application implements mTLS itself. Istio sidecar also applies mTLS. Double encryption fails cert negotiation or causes unexpected behavior.

---

### EC-081 — ClusterIP Service Not Reachable From the Node
**Category:** K8s Networking | **Severity:** High | **Env:** Prod

**Scenario:** `kubectl exec pod -- curl http://service-ip` works. SSH to the node and `curl http://service-ip` fails. ClusterIP is only routable from within the cluster (pods and cluster networking stack), not from node's host namespace.

---

### EC-082 — ExternalTrafficPolicy Local Drops Traffic on Nodes Without Pod
**Category:** K8s Networking | **Severity:** High | **Env:** Prod

**Scenario:** `externalTrafficPolicy: Local` set to preserve client IP. Traffic arriving at nodes without a matching pod is dropped (not forwarded). Uneven pod distribution causes traffic black holes.

---

### EC-083 — Container Name Resolution Fails Across Docker Compose Networks
**Category:** Container Networking | **Severity:** Medium | **Env:** Dev

**Scenario:** Service A in docker-compose can reach service B by name. After adding a third service in a different `networks` block, A cannot reach B anymore (different network).

---

### EC-084 — High Pod Density Causes IP Address Exhaustion in Subnet
**Category:** K8s Networking | **Severity:** Critical | **Env:** Prod

**Scenario:** AWS EKS node in /24 subnet (254 IPs). VPC CNI assigns IPs from the subnet. With 110 pods per node, a few nodes exhaust the /24 subnet.

> ⚠️ **DevOps Gotcha:** AWS VPC CNI uses node IP addresses for pods. Plan subnet sizes for expected pod density. Use `/19` or larger subnets for EKS worker subnets, or enable prefix delegation.

---

### EC-085 — kube-proxy IPVS Mode Loses Connections on Pod Delete
**Category:** K8s Networking | **Severity:** High | **Env:** Prod

**Scenario:** IPVS mode with `--ipvs-min-sync-period=0`. Pod deleted. IPVS immediately removes the endpoint. In-flight requests to that pod are dropped.

**Fix:**
```bash
# Use preStop hook for graceful termination:
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
# Also set terminationGracePeriodSeconds appropriately
```

---

## VPN & Tunneling

### EC-086 — WireGuard MTU Not Reduced — Fragmentation Inside Tunnel
**Category:** VPN | **Severity:** High | **Env:** Prod

**Scenario:** WireGuard tunnel on interface with MTU 1500. WireGuard adds 60-byte overhead (IPv4+UDP+WG header). Effective MTU inside tunnel is 1440. Large packets fragment or are dropped.

**Fix:**
```ini
# In WireGuard config:
[Interface]
MTU = 1420   # 1500 - 60 byte WireGuard overhead - some buffer
```

---

### EC-087 — OpenVPN Split Tunneling Sends DNS Outside VPN
**Category:** VPN | **Severity:** High | **Env:** Prod

**Scenario:** OpenVPN with split tunneling routes corporate IPs through VPN. Client DNS is still the ISP resolver. Queries for `internal.corp.com` go to ISP resolver (gets NXDOMAIN).

**Fix:**
```bash
# Push DNS in OpenVPN server config:
push "dhcp-option DNS 10.0.0.53"
push "dhcp-option DOMAIN corp.com"
```

---

### EC-088 — IPsec Renegotiation Causes 30-Second Blackout
**Category:** VPN | **Severity:** High | **Env:** Prod

**Scenario:** IPsec SA lifetime expires. During renegotiation, traffic is dropped for 20–30 seconds. Scheduled every 1–8 hours, causing regular brief outages.

**Fix:**
```bash
# Overlap SA lifetimes with margin:
# strongSwan: ikelifetime=4h; lifetime=1h; margintime=9min; rekeyfuzz=100%;
# This starts renegotiation 9 minutes before expiry
```

---

### EC-089 — GRE Tunnel Doesn't Support ECMP Due to Fixed Outer Header
**Category:** Networking | **Severity:** Medium | **Env:** Prod

**Scenario:** GRE tunnel over ECMP network. All tunnel traffic takes same ECMP path because outer header has fixed src/dst IP. ECMP hashing on outer header makes all traffic one path.

---

### EC-090 — Tunnel Interface Drops Packets Due to Missing `ip_forward`
**Category:** Tunneling | **Severity:** High | **Env:** Prod

**Scenario:** Set up IPIP or GRE tunnel. Tunnel interface is up but packets are dropped. `ip_forward` not enabled for that specific interface.

**Fix:**
```bash
sysctl -w net.ipv4.conf.tunl0.forwarding=1
sysctl -w net.ipv4.conf.all.forwarding=1
```

---

### EC-091 — VPN Reconnect Doesn't Restore Routes
**Category:** VPN | **Severity:** Medium | **Env:** Prod

**Scenario:** VPN drops and reconnects. Routes added by VPN client are gone (not re-added on reconnect). Client thinks VPN is working but traffic goes direct.

> ⚠️ **DevOps Gotcha:** Always use `ip route show table all` after VPN reconnect to verify routes are restored. Many VPN clients don't re-add routes correctly after reconnect without full restart.

---

## Network Performance & Diagnostics

### EC-092 — `ping` Passes But Application TCP Connections Fail
**Category:** Diagnostics | **Severity:** High | **Env:** Prod

**Scenario:** `ping host` succeeds (ICMP). `telnet host 443` hangs. The firewall allows ICMP but blocks TCP port 443.

```
  Testing protocol by protocol:
  ICMP ping: ✅ (layer 3 connectivity)
  TCP 80: telnet host 80  (layer 4 connectivity, HTTP)
  TCP 443: curl -v https://host  (layer 4 + TLS)
  App response: curl https://host/api  (layer 7)
```

---

### EC-093 — `traceroute` Shows * * * But Path Exists
**Category:** Diagnostics | **Severity:** Low | **Env:** Prod

**Scenario:** `traceroute` shows `* * *` at a hop. Network admin thinks the path is broken. The hop is working fine but that router drops UDP/ICMP traceroute probes.

> ⚠️ **DevOps Gotcha:** Use `traceroute -T -p 443` (TCP traceroute on port 443) to bypass firewalls that block UDP/ICMP. MTR provides continuous real-time traceroute.

---

### EC-094 — Bandwidth Test Shows 100% But Latency Is High
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** `iperf3` shows 1Gbps throughput. Application response time is 500ms. High bandwidth ≠ low latency. Buffer bloat is filling network queues.

**Detection:**
```bash
# Measure latency DURING bandwidth test (buffer bloat):
ping -i 0.1 host &
iperf3 -c host
# If ping RTT spikes during iperf, buffer bloat exists
```

**Fix:**
```bash
# Enable CoDel/FQ-CoDel queueing discipline:
tc qdisc add dev eth0 root fq_codel
```

---

### EC-095 — `ss -s` Summary Doesn't Show Per-Connection Details
**Category:** Diagnostics | **Severity:** Low | **Env:** Dev/Prod

**Scenario:** Debugging connection issue. `ss -s` shows aggregate counts. Need per-connection details including which process owns each socket.

**Detection:**
```bash
ss -tlnp   # listening, process name
ss -tanp   # all TCP, process name, numeric
ss -tanpo  # + timer info (keepalive, retransmit)
ss -i      # internal TCP info (RTT, cwnd, etc.)
```

---

### EC-096 — Network Packet Loss Only Visible Under Load
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** `ping` shows 0% packet loss. Under production load, 2% packet loss observed. The NIC ring buffer overflows under burst traffic.

**Detection:**
```bash
ethtool -S eth0 | grep -i drop
ip -s link show eth0 | grep -i drop
cat /proc/net/dev | grep eth0   # RX/TX error columns
```

**Fix:**
```bash
# Increase NIC ring buffer:
ethtool -G eth0 rx 4096 tx 4096
# Enable RPS (Receive Packet Steering) for multi-core:
echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus
```

---

### EC-097 — Jumbo Frames (MTU 9000) Enabled on Some Hops But Not All
**Category:** Performance | **Severity:** High | **Env:** Prod

**Scenario:** SAN storage network configured for jumbo frames. Server NIC set to 9000 MTU. But switch hasn't enabled jumbo frames. Packets > 1500 bytes silently dropped.

**Detection:**
```bash
ping -M do -s 8972 <storage-ip>   # 8972 + 28 headers = 9000 MTU
# ICMP error means jumbo frames not end-to-end
```

---

### EC-098 — `tc` Traffic Control Rules Clear After NIC Restart
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** `tc qdisc add dev eth0 root tbf ...` for traffic shaping. After `ip link set eth0 down/up`, all `tc` rules are cleared.

**Fix:**
```bash
# Use NetworkManager dispatch or udev rules to re-apply:
cat > /etc/NetworkManager/dispatcher.d/99-tc-rules.sh << 'EOF'
#!/bin/bash
if [ "$1" = "eth0" ] && [ "$2" = "up" ]; then
    tc qdisc add dev eth0 root tbf rate 100mbit burst 100kb latency 50ms
fi
EOF
chmod +x /etc/NetworkManager/dispatcher.d/99-tc-rules.sh
```

---

### EC-099 — Duplicate ACK Storm After Network Recovery
**Category:** TCP Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** Brief network partition recovers. Both sides have queued data. Duplicate ACKs trigger TCP fast retransmit on both sides simultaneously, flooding the link.

---

### EC-100 — Kernel's TCP Receive Buffer Auto-Tuning Disabled
**Category:** Performance | **Severity:** Medium | **Env:** Prod

**Scenario:** High-throughput application underperforms because TCP receive buffers are too small. Linux auto-tunes, but `tcp_moderate_rcvbuf` is disabled.

**Detection:**
```bash
sysctl net.ipv4.tcp_moderate_rcvbuf   # should be 1
sysctl net.ipv4.tcp_rmem   # min, default, max values
```

**Fix:**
```bash
sysctl -w net.ipv4.tcp_rmem="4096 131072 33554432"
sysctl -w net.ipv4.tcp_wmem="4096 65536 33554432"
sysctl -w net.core.rmem_max=33554432
```

---

### EC-101 — `netstat` Missing Entries vs `ss` Showing More
**Category:** Diagnostics | **Severity:** Low | **Env:** Dev/Prod

**Scenario:** `netstat -an | wc -l` shows 500 connections. `ss -an | wc -l` shows 800. `netstat` is deprecated and may miss connections in newer kernels.

> ⚠️ **DevOps Gotcha:** `netstat` uses `/proc/net/tcp` which can be inconsistent. `ss` directly queries the kernel via netlink socket. Always prefer `ss` over `netstat` on modern Linux systems.

---

### EC-102 — `tcpdump` Drops Packets at High Capture Rate
**Category:** Diagnostics | **Severity:** Medium | **Env:** Prod

**Scenario:** Running `tcpdump` on busy 10Gbps interface. Capture output shows "X packets dropped by kernel". Packets are lost in the ring buffer before tcpdump can read them.

**Fix:**
```bash
# Increase tcpdump buffer size:
tcpdump -i eth0 -B 65536 -w capture.pcap
# Or use hardware-timestamping tools:
# tcpdump -i eth0 --buffer-size=65536 -w -  | wireshark -k -i -
```

---

### EC-103 — `nmap` Port Scan Blocked by Rate Limiting Gives False "Filtered"
**Category:** Diagnostics | **Severity:** Low | **Env:** Prod

**Scenario:** `nmap -p 1-1000` shows many ports as "filtered". The server has rate limiting (hashlimit). Slow `nmap` scan shows ports as open.

```bash
# Use -T1 (sneaky) for rate-limited targets:
nmap -T1 -p 1-100 <target>
```

---

### EC-104 — `curl` vs `wget` Different TLS Behavior
**Category:** Diagnostics | **Severity:** Low | **Env:** Dev/Prod

**Scenario:** `curl https://internal.site` fails with certificate error. `wget https://internal.site` works. They use different SSL libraries (curl uses libcurl/OpenSSL, wget may use GnuTLS).

> ⚠️ **DevOps Gotcha:** When debugging TLS issues, test with both `curl -v` and `openssl s_client` to distinguish between library-specific quirks and actual certificate problems.

---

### EC-105 — Firewall Blocks ICMP Unreachable — UDP Apps Never Know Port Closed
**Category:** Diagnostics | **Severity:** High | **Env:** Prod

**Scenario:** UDP client sends to a closed port. Normally receives ICMP Port Unreachable. Firewall blocks ICMP. Client keeps retrying forever — never knows the port is closed.

> ⚠️ **DevOps Gotcha:** Never block `ICMP Type 3` (Destination Unreachable) and `ICMP Type 11` (Time Exceeded) on firewalls. These are essential for network functionality. Only block `ICMP Type 8` (echo request) if you want to hide servers from ping.

---

## Cloud Networking

### EC-106 — AWS Security Group Changes Take Effect Immediately on Open Connections
**Category:** Cloud Networking | **Severity:** High | **Env:** Prod

**Scenario:** Remove inbound rule from security group. Expect existing connections to drain. AWS Security Groups are stateful — removing a rule immediately drops new packets but may not RST existing connections (depends on flow direction).

> ⚠️ **DevOps Gotcha:** AWS SG rule changes are near-immediate. For safe traffic shifts, change SGs on a new group and use rolling deployment, not in-place rule changes on production SGs.

---

### EC-107 — VPC Peering Doesn't Transitively Route Traffic
**Category:** Cloud Networking | **Severity:** High | **Env:** Prod

**Scenario:** VPC-A peered with VPC-B. VPC-B peered with VPC-C. Cannot route traffic from VPC-A to VPC-C through VPC-B.

```
  VPC-A ←→ VPC-B ←→ VPC-C
  VPC-A CANNOT reach VPC-C via VPC-B!
  Transitive peering is NOT supported.
  
  Solutions: Full mesh OR AWS Transit Gateway
```

---

### EC-108 — ELB Access Logs Disable Silently When S3 Bucket Policy Changes
**Category:** Cloud Networking | **Severity:** Medium | **Env:** Prod

**Scenario:** S3 bucket policy updated (security hardening). ELB access logging silently stops because the ELB service account no longer has write permission. No alert, no error.

---

### EC-109 — Route Table Missing for New Availability Zone
**Category:** Cloud Networking | **Severity:** High | **Env:** Prod

**Scenario:** New subnet added in AZ-c for resilience. Route table not associated. Resources in that subnet cannot reach the internet or other VPCs.

**Detection:**
```bash
# AWS CLI:
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-xxx"
```

---

### EC-110 — Cloud Provider NAT Gateway Drops Flows Over 64k
**Category:** Cloud Networking | **Severity:** High | **Env:** Prod

**Scenario:** AWS NAT Gateway supports 55,000 simultaneous flows per gateway. High-traffic application exhausts NAT Gateway concurrent connection limit. New connections fail with no clear error.

**Detection:**
```bash
# CloudWatch metric: ErrorPortAllocation
# AWS NAT Gateway limits:
# - 55,000 simultaneous connections
# - 4 Gbps bandwidth
```

**Fix:**
```bash
# Solutions:
# 1. Multiple NAT Gateways (one per AZ) + route tables per AZ
# 2. Use NAT Instances with larger capacity
# 3. Keep connections short-lived (reduce timeout)
```

> ⚠️ **DevOps Gotcha:** AWS NAT Gateway is invisible in traditional network monitoring. Use CloudWatch `ErrorPortAllocation` metric. If you're hitting this limit, it means your architecture has too many outbound connections — rethink connection pooling.

---

## Production FAQ

**Q1: Intermittent connection failures every few minutes — where to look first?**  
A: Check TCP keepalive vs firewall timeout mismatch. `ss -tan state established | wc -l` + check if connections are being reset. Most common cause: NAT/firewall idle timeout shorter than application keepalive interval.

**Q2: DNS works on the server but not in containers.**  
A: Container uses a different resolver (`/etc/resolv.conf` inside container). Check `docker inspect container | grep Dns`. Docker's embedded DNS `127.0.0.11` may be blocking. Test with `nslookup` inside the container.

**Q3: `curl` timeout but browser works fine.**  
A: Browser uses keep-alive connections and has cached responses. `curl` opens new connection every time. Check if connection to that port works: `timeout 5 bash -c 'cat < /dev/tcp/host/443'`. Browser likely has a cached result or uses HTTP/2 multiplexing.

**Q4: High network latency during business hours only.**  
A: Look at QoS/traffic shaping policies that activate during business hours. Check interface errors for packet loss. Run `mtr host` during and outside business hours and compare hop latency.

**Q5: Can reach public internet but not specific corporate services via VPN.**  
A: Split-tunnel VPN issue. Check routing table: `ip route show`. Corporate routes should route through VPN interface (e.g., `tun0`). DNS may also need to go through VPN nameserver.

**Q6: HTTPS works but WebSockets keep dropping after 60 seconds.**  
A: AWS ALB and many load balancers have 60-second idle timeout. WebSocket connections that are idle (no ping/pong) get terminated. Enable WebSocket keepalive pings at the application level.

**Q7: Why does `iptables -L` return slowly?**  
A: `iptables -L` does reverse DNS lookup for each IP. Use `iptables -L -n` (numeric, no DNS) for instant results.

**Q8: New VM deployed but can't be reached by hostname.**  
A: DNS record not created for new VM. Check if DHCP is configured to update DNS (DDNS). In AWS/Azure/GCP, check if the VPC/VNet has DNS hostname resolution enabled and if the instance's hostname is registered.

**Q9: Traffic from containerized service missing in tcpdump on host.**  
A: Container traffic at layer 2 may go through a virtual bridge. Capture on the bridge interface: `tcpdump -i docker0` or on the veth interface: `tcpdump -i vethXXXXX`.

**Q10: Application works with IPv4 but not IPv6.**  
A: Check if the app listens on `::` (both v4+v6) or only `0.0.0.0` (v4 only). Check IPv6 routing: `ip -6 route show`. Check ip6tables rules. Verify the service has an AAAA record if using DNS.

---

> **Next:** [Cloud Edge Cases](cloud_edge_cases.md) · [Git Edge Cases](git_edge_cases.md)  
> **Back:** [Linux Edge Cases](linux_edge_cases.md) · [Batch State](batch_state.md) · [🏠 Home](../README.md)
