# 🐘 PostgreSQL Configuration — Solution Architect Reference

> Every parameter a solution architect must know, when to tune it, and why.

---

## 1. How PostgreSQL Config Works

```
┌──────────────────────────────────────────────────────────────┐
│  CONFIG FILES (in data directory, e.g. /var/lib/postgresql)  │
│                                                              │
│  postgresql.conf   → Main config (performance, memory, etc.) │
│  pg_hba.conf       → Client authentication rules             │
│  pg_ident.conf     → OS-to-PG user mapping                   │
│                                                              │
│  APPLY CHANGES:                                              │
│  Some params → reload only:  SELECT pg_reload_conf();        │
│                          or: pg_ctl reload                   │
│  Some params → RESTART required (marked below with ⚠️)       │
│                                                              │
│  CHECK CURRENT VALUE:                                        │
│  SHOW shared_buffers;                                        │
│  SELECT name, setting, unit FROM pg_settings WHERE ...;      │
│                                                              │
│  AZURE DATABASE FOR POSTGRESQL:                              │
│  No direct file access → use Server Parameters blade        │
│  Some params are NOT configurable on managed services        │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Memory Parameters

### shared_buffers ⚠️ (restart required)

```
What:    PostgreSQL's own page cache (shared across all connections)
Default: 128MB (way too low for production)
Set to:  25% of total RAM

Examples:
  8GB RAM server  → shared_buffers = 2GB
  16GB RAM server → shared_buffers = 4GB
  64GB RAM server → shared_buffers = 16GB

WHY 25% and not more?
PostgreSQL ALSO uses the OS page cache (filesystem cache).
If you give PG 75% of RAM, the OS has nothing left for its own cache.
The two caches work together — PG's for frequently accessed pages,
OS cache for less-frequent reads.

Azure Flexible Server: auto-tuned based on SKU, usually ~25% already.

Check effectiveness:
  SELECT
    sum(heap_blks_read) AS disk_reads,
    sum(heap_blks_hit) AS cache_hits,
    round(sum(heap_blks_hit) * 100.0 /
      (sum(heap_blks_hit) + sum(heap_blks_read)), 2) AS cache_hit_ratio
  FROM pg_statio_user_tables;
  -- Target: > 99% cache hit ratio
```

### effective_cache_size (no restart)

```
What:    Hint to query planner about TOTAL cache available
         (shared_buffers + OS page cache). Does NOT allocate memory.
Default: 4GB
Set to:  50-75% of total RAM

Examples:
  8GB RAM  → effective_cache_size = 6GB
  16GB RAM → effective_cache_size = 12GB
  64GB RAM → effective_cache_size = 48GB

WHY it matters:
A larger value tells the planner "lots of data will be cached"
→ planner prefers index scans over sequential scans
→ better query plans for large tables
```

### work_mem (no restart)

```
What:    Memory per SORT / HASH operation per query per connection
Default: 4MB
Set to:  Carefully — it multiplies fast

CALCULATION:
  A single complex query might do 5+ sorts/hashes simultaneously.
  100 connections × 5 operations × work_mem = total memory consumed.

  100 connections × 5 × 64MB = 32GB → OOM kill!
  100 connections × 5 × 16MB = 8GB  → manageable on 32GB server

RULE OF THUMB:
  Total RAM / max_connections / 4 (rough starting point)

  8GB, 50 connections  → work_mem = 32-40MB
  16GB, 100 connections → work_mem = 32MB
  64GB, 200 connections → work_mem = 64MB

HOW TO KNOW if it's too low:
  EXPLAIN ANALYZE your slow queries.
  If you see "Sort Method: external merge Disk" → work_mem too low.
  Want to see: "Sort Method: quicksort Memory: XXkB"

Set per-session for specific workloads:
  SET work_mem = '256MB';   -- only for this session
  -- run your analytical query
  RESET work_mem;
```

### maintenance_work_mem (no restart)

```
What:    Memory for maintenance operations: VACUUM, CREATE INDEX, ALTER TABLE
Default: 64MB
Set to:  256MB - 2GB depending on RAM

Not per-connection — only a few maintenance operations run at once.
Larger = faster VACUUM and index creation.

  8GB RAM   → maintenance_work_mem = 512MB
  16GB RAM  → maintenance_work_mem = 1GB
  64GB RAM  → maintenance_work_mem = 2GB
```

### huge_pages ⚠️ (restart required)

```
What:    Use OS huge pages (2MB instead of 4KB page sizes)
Default: try
Set to:  on (for servers with >= 8GB shared_buffers)

WHY: Reduces Translation Lookaside Buffer (TLB) misses.
     With 8GB shared_buffers at 4KB pages = 2 million page table entries.
     With 2MB huge pages = only 4096 entries.

Requires OS configuration first:
  sysctl -w vm.nr_hugepages=4096  # Must match shared_buffers / 2MB
```

---

## 3. Connection Parameters

### max_connections ⚠️ (restart required)

```
What:    Maximum concurrent connections
Default: 100
Set to:  DON'T increase blindly — use connection pooling instead

EVERY CONNECTION COSTS:
  ~10MB of RAM (backend process + work_mem potential)
  CPU context switching overhead
  Lock contention

500 connections × 10MB = 5GB just for connection overhead!

RULE: max_connections should be low (100-200).
      Use PgBouncer for connection pooling (handles 10,000+ clients → 50 PG connections).

┌──────────────────────────────────────────────────────────────┐
│  WITHOUT POOLING:                                            │
│  500 app instances → 500 PG connections (most idle)         │
│  PG: high memory, lock contention, CPU thrashing            │
│                                                              │
│  WITH PGBOUNCER:                                             │
│  500 app instances → PgBouncer (pool) → 50 PG connections  │
│  PG: low memory, efficient, fast                            │
└──────────────────────────────────────────────────────────────┘

Azure Flexible Server: has built-in PgBouncer. Enable it.
```

### idle_in_transaction_session_timeout (no restart)

```
What:    Kill connections that sit idle INSIDE an open transaction
Default: 0 (disabled — dangerous!)
Set to:  30s - 5min

WHY: Idle-in-transaction connections hold locks and prevent VACUUM.
     A developer forgets to close a transaction → table can't be vacuumed for hours
     → table bloats → performance degrades.

  idle_in_transaction_session_timeout = '60s'

Check for idle-in-transaction:
  SELECT pid, state, now() - state_change AS duration, query
  FROM pg_stat_activity
  WHERE state = 'idle in transaction'
  ORDER BY duration DESC;
```

### statement_timeout (no restart)

```
What:    Kill any single query running longer than this
Default: 0 (disabled)
Set to:  30s for web apps, 0 for batch/analytics

  statement_timeout = '30s'     -- web API backend
  -- Override per-session for batch jobs:
  SET statement_timeout = '10min';
```

---

## 4. Write-Ahead Log (WAL) & Durability

```
┌──────────────────────────────────────────────────────────────┐
│  WAL — HOW POSTGRES ENSURES DATA SAFETY                      │
│                                                              │
│  Client writes data                                         │
│       │                                                      │
│       ▼                                                      │
│  Change written to WAL (sequential write — FAST)            │
│       │                                                      │
│       ▼                                                      │
│  PG acknowledges commit to client (data is safe now)        │
│       │                                                      │
│       ▼ (later, asynchronously)                             │
│  Checkpoint: flush dirty pages from shared_buffers to disk  │
│                                                              │
│  If PG crashes after commit but before checkpoint:          │
│  → Replay WAL on startup → data recovered ✅                │
└──────────────────────────────────────────────────────────────┘
```

### wal_level ⚠️

```
What:    How much information is written to WAL
Options: minimal | replica | logical
Default: replica

  replica  → enough for physical replication (standby servers)
  logical  → enough for logical replication (CDC, Debezium, DMS)

Set to logical if:
  - You're doing Change Data Capture (CDC)
  - Using logical replication to another PG or external system
  - Azure DMS (Database Migration Service) requires it
```

### max_wal_size & min_wal_size (no restart)

```
What:    Controls when PG triggers a checkpoint
Default: max_wal_size = 1GB, min_wal_size = 80MB

If WAL grows to max_wal_size → forced checkpoint.
Larger value = fewer checkpoints = better write performance, longer crash recovery.

  Light OLTP:   max_wal_size = 2GB
  Heavy writes:  max_wal_size = 8-16GB
  Bulk imports:  max_wal_size = 32GB (temporarily)
```

### checkpoint_completion_target (no restart)

```
What:    Spread checkpoint I/O over this fraction of the checkpoint interval
Default: 0.9 (good — leave it)

0.9 = spread writes over 90% of time between checkpoints = smooth I/O
0.5 = rush all writes in first half = I/O spikes
```

### synchronous_commit (no restart)

```
What:    Whether to wait for WAL write to disk before confirming commit
Default: on (safest)

  on       → commit waits for WAL flush to disk → safest, slower
  off      → commit returns before WAL flush → faster, risk of last ~600ms of data on crash
  local    → commit waits for local WAL flush but NOT replica

Use 'off' ONLY for data you can afford to lose (session data, logs, caches).
Never 'off' for financial or user data.

Can set per-transaction:
  SET LOCAL synchronous_commit = 'off';
  INSERT INTO logs (...);  -- fast, acceptable loss
  COMMIT;
```

---

## 5. Autovacuum — The Most Misunderstood System

```
┌──────────────────────────────────────────────────────────────┐
│  WHY VACUUM EXISTS                                           │
│                                                              │
│  PostgreSQL uses MVCC (Multi-Version Concurrency Control):  │
│  UPDATE doesn't modify the row — it creates a NEW version   │
│  and marks the OLD version as "dead".                       │
│                                                              │
│  DELETE doesn't remove the row — it marks it as "dead".     │
│                                                              │
│  Dead rows pile up → table bloats → queries scan more data  │
│  → performance degrades → disk fills up.                    │
│                                                              │
│  VACUUM reclaims dead row space.                            │
│  ANALYZE updates table statistics for the query planner.    │
│                                                              │
│  Autovacuum = automatic background process that runs both.  │
│  NEVER DISABLE AUTOVACUUM.                                  │
└──────────────────────────────────────────────────────────────┘
```

### Key autovacuum parameters

```
# How often autovacuum checks for work (default: 1min — fine)
autovacuum_naptime = '1min'

# Trigger vacuum when this many dead tuples exist
autovacuum_vacuum_threshold = 50            # base count
autovacuum_vacuum_scale_factor = 0.1        # + 10% of table size

# Formula: vacuum when dead_tuples > threshold + scale_factor × table_rows
# Default: vacuum when dead > 50 + 0.1 × table_rows
# 1M row table: vacuum at 100,050 dead rows (10% dead) → too late!

# FOR LARGE TABLES, lower the scale factor:
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.01);  -- 1%
ALTER TABLE orders SET (autovacuum_vacuum_threshold = 1000);

# How many autovacuum workers can run simultaneously
autovacuum_max_workers = 3                  # default, increase for many tables

# Speed: how fast autovacuum can work before yielding
autovacuum_vacuum_cost_delay = 2ms          # default 2ms (PG14+)
# Lower = more aggressive (faster cleanup, more I/O)
# Set to 0 for dedicated DB servers with fast SSDs
```

### Check autovacuum health

```sql
-- Tables that need vacuuming (most dead rows)
SELECT relname, n_live_tup, n_dead_tup,
  round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Tables with bloat (space wasted)
SELECT tablename,
  pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size
FROM pg_tables WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;

-- Is autovacuum currently running?
SELECT pid, query, state, wait_event, now() - query_start AS duration
FROM pg_stat_activity WHERE query LIKE 'autovacuum%';
```

---

## 6. Query Planner Parameters

### random_page_cost / seq_page_cost

```
What:    Planner's cost estimate for random vs sequential disk reads
Default: random_page_cost = 4.0, seq_page_cost = 1.0

This tells the planner: "random I/O is 4× slower than sequential."
On SSDs, random I/O is nearly as fast as sequential.

For SSD storage (Azure Premium SSD, NVMe):
  random_page_cost = 1.1    # Nearly same as sequential
  seq_page_cost = 1.0

For HDD:
  random_page_cost = 4.0    # Keep default

WHY: If random_page_cost is too high on SSDs, the planner avoids
index scans and does sequential scans instead → slow queries.
```

### default_statistics_target

```
What:    How many rows ANALYZE samples per column for statistics
Default: 100
Set to:  200-500 for large tables with skewed data

More samples = better statistics = better query plans.
Cost: ANALYZE takes slightly longer.

Per-column override for important columns:
  ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
  ANALYZE orders;
```

### jit (no restart, PG 12+)

```
What:    Just-In-Time compilation of queries
Default: on (PG 12+)

Good for: long-running analytical queries (aggregate millions of rows).
Bad for: short OLTP queries (JIT compilation overhead > query time).

For web APIs (OLTP): jit = off
For analytics/reporting: jit = on
```

---

## 7. Replication Parameters

```
┌──────────────────────────────────────────────────────────────┐
│  REPLICATION ARCHITECTURES                                   │
│                                                              │
│  1. STREAMING REPLICATION (physical):                       │
│     Primary ──WAL──► Standby (exact byte-for-byte copy)     │
│     Use for: HA failover, read replicas                     │
│                                                              │
│  2. LOGICAL REPLICATION:                                     │
│     Primary ──changes──► Subscriber (table-level, filtered)  │
│     Use for: CDC, cross-version migration, selective sync   │
│                                                              │
│  Azure Flexible Server: uses streaming replication          │
│  for read replicas and zone-redundant HA automatically.     │
└──────────────────────────────────────────────────────────────┘

# Primary server settings:
wal_level = replica             # or 'logical' for logical replication
max_wal_senders = 10            # max replication connections
max_replication_slots = 10      # prevent WAL cleanup before replica consumes it
hot_standby = on                # allow queries on standby

# Synchronous replication (zero data loss, higher latency):
synchronous_standby_names = 'replica1'    # name of standby
# Commit waits for replica to confirm WAL receipt.
# If replica is down → all writes BLOCK. Use with caution.
```

---

## 8. Logging Parameters

```
# What to log
log_statement = 'ddl'             # none | ddl | mod | all
  # ddl  = log CREATE, ALTER, DROP (recommended for prod)
  # mod  = + INSERT, UPDATE, DELETE (noisy but useful for audit)
  # all  = everything (very noisy — only for debugging)

# Slow query logging
log_min_duration_statement = 1000   # Log queries taking > 1 second (in ms)
  # Set to 500ms for busy OLTP, 5000ms for analytics

# Log format
log_line_prefix = '%m [%p] %q%u@%d '
  # %m = timestamp, %p = PID, %u = user, %d = database
  # Result: "2025-07-16 10:30:00 [12345] myuser@mydb SELECT..."

# Connection logging
log_connections = on
log_disconnections = on

# Checkpoint logging
log_checkpoints = on              # Log when checkpoints happen + duration

# Lock wait logging
log_lock_waits = on               # Log when a query waits > deadlock_timeout for a lock
deadlock_timeout = '1s'

# Auto-explain: log EXPLAIN for slow queries
shared_preload_libraries = 'auto_explain'   # ⚠️ restart required
auto_explain.log_min_duration = '3s'        # Explain queries > 3s
auto_explain.log_analyze = on               # Include actual times
```

---

## 9. Security Parameters

```
# Authentication (pg_hba.conf)
# TYPE  DATABASE  USER      ADDRESS          METHOD
  host  all       all       10.0.0.0/8       scram-sha-256
  host  all       all       0.0.0.0/0        reject

# Encryption
ssl = on                          # ⚠️ restart required
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_min_protocol_version = 'TLSv1.2'

# Password encryption
password_encryption = 'scram-sha-256'   # NOT md5 (weak)

# Row-level security (per-table)
ALTER TABLE customer_data ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON customer_data
  USING (tenant_id = current_setting('app.current_tenant')::int);
```

---

## 10. Azure-Specific Considerations

```
┌──────────────────────────────────────────────────────────────┐
│  AZURE DB FOR POSTGRESQL — WHAT'S DIFFERENT                  │
│                                                              │
│  Managed for you (can't change):                            │
│  ├── shared_buffers (auto-tuned by SKU)                     │
│  ├── huge_pages (managed by Azure)                          │
│  ├── SSL certificates (Azure managed)                       │
│  ├── Physical replication setup (read replicas)             │
│  └── Backup retention & PITR (configurable in portal)       │
│                                                              │
│  You MUST configure:                                        │
│  ├── work_mem, maintenance_work_mem                         │
│  ├── max_connections (or enable PgBouncer)                  │
│  ├── log_min_duration_statement                             │
│  ├── autovacuum scale factors for large tables              │
│  ├── random_page_cost = 1.1 (Azure uses SSDs)              │
│  ├── idle_in_transaction_session_timeout                    │
│  └── statement_timeout                                      │
│                                                              │
│  Built-in PgBouncer: enable in Server Parameters            │
│  → pgbouncer.enabled = true                                │
│  → pgbouncer.default_pool_size = 50                        │
│  → pgbouncer.pool_mode = transaction                       │
│                                                              │
│  IOPS: tied to storage size on Azure                        │
│  Burstable: 3 IOPS per GB provisioned                      │
│  General Purpose: tiered based on compute SKU              │
│  → If queries are slow, check if you're hitting IOPS limit │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. Quick Sizing Guide

```
┌──────────────────────┬────────────┬────────────┬─────────────┐
│ Parameter            │ 8GB / 2CPU │ 32GB / 8CPU│ 128GB/32CPU │
├──────────────────────┼────────────┼────────────┼─────────────┤
│ shared_buffers       │ 2GB        │ 8GB        │ 32GB        │
│ effective_cache_size │ 6GB        │ 24GB       │ 96GB        │
│ work_mem             │ 16MB       │ 64MB       │ 128MB       │
│ maintenance_work_mem │ 512MB      │ 1GB        │ 2GB         │
│ max_connections      │ 100        │ 200        │ 300         │
│ max_wal_size         │ 2GB        │ 8GB        │ 16GB        │
│ random_page_cost     │ 1.1 (SSD)  │ 1.1        │ 1.1         │
│ effective_io_concurr │ 200 (SSD)  │ 200        │ 200         │
│ autovac_max_workers  │ 3          │ 5          │ 8           │
└──────────────────────┴────────────┴────────────┴─────────────┘
```

---

## 12. Essential Diagnostic Queries

```sql
-- Active queries right now
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state != 'idle' ORDER BY duration DESC;

-- Table sizes
SELECT tablename,
  pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total,
  pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_only,
  pg_size_pretty(pg_indexes_size(tablename::regclass)) AS indexes
FROM pg_tables WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;

-- Index usage (find unused indexes — they slow down writes)
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
-- idx_scan = 0 → index is NEVER used → consider dropping

-- Cache hit ratio (should be > 99%)
SELECT
  sum(heap_blks_hit) * 100.0 / (sum(heap_blks_hit) + sum(heap_blks_read)) AS ratio
FROM pg_statio_user_tables;

-- Connection count by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

-- Blocking queries (who is blocking whom)
SELECT blocked.pid AS blocked_pid, blocked.query AS blocked_query,
       blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks gl ON gl.locktype = bl.locktype AND gl.relation = bl.relation AND gl.granted
JOIN pg_stat_activity blocking ON blocking.pid = gl.pid
WHERE blocked.pid != blocking.pid;

-- Replication lag (on primary)
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

---

## 13. Production Checklist

```
┌──────────────────────────────────────────────────────────────┐
│  POSTGRESQL PRODUCTION CHECKLIST                             │
│                                                              │
│  Memory:                                                     │
│  [ ] shared_buffers = 25% of RAM                            │
│  [ ] effective_cache_size = 50-75% of RAM                   │
│  [ ] work_mem calculated per connection count               │
│  [ ] maintenance_work_mem = 512MB-2GB                       │
│                                                              │
│  Connections:                                                │
│  [ ] PgBouncer enabled (transaction pooling mode)           │
│  [ ] max_connections kept low (100-200)                     │
│  [ ] idle_in_transaction_session_timeout = 60s              │
│  [ ] statement_timeout set for web apps (30s)               │
│                                                              │
│  WAL & Durability:                                          │
│  [ ] max_wal_size tuned (8GB+ for write-heavy)              │
│  [ ] checkpoint_completion_target = 0.9                     │
│  [ ] synchronous_commit = on (unless acceptable loss)       │
│                                                              │
│  Autovacuum:                                                │
│  [ ] autovacuum = on (NEVER disable)                        │
│  [ ] Large tables: custom scale_factor (0.01-0.05)          │
│  [ ] Monitor dead tuple count regularly                     │
│                                                              │
│  Query Planner:                                              │
│  [ ] random_page_cost = 1.1 (SSD storage)                  │
│  [ ] default_statistics_target = 200+                       │
│  [ ] jit = off for OLTP workloads                           │
│                                                              │
│  Logging:                                                    │
│  [ ] log_min_duration_statement = 1000 (slow query log)     │
│  [ ] log_checkpoints = on                                   │
│  [ ] log_lock_waits = on                                    │
│  [ ] log_connections = on                                   │
│                                                              │
│  Security:                                                   │
│  [ ] ssl = on, TLS 1.2 minimum                              │
│  [ ] password_encryption = scram-sha-256                    │
│  [ ] pg_hba.conf restricts to known subnets only            │
│  [ ] No trust or md5 auth methods                           │
│                                                              │
│  Monitoring:                                                 │
│  [ ] Cache hit ratio > 99%                                  │
│  [ ] Connection count < 80% of max                          │
│  [ ] Replication lag < 1 second                             │
│  [ ] Dead tuple ratio < 10% on all tables                   │
│  [ ] Unused indexes identified and removed                  │
│  [ ] Bloated tables identified and vacuumed                 │
└──────────────────────────────────────────────────────────────┘
```
