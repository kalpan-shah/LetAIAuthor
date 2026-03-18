# Chapter 8: PostgreSQL Metrics with postgres_exporter

> **Part III — Metrics with Prometheus**

---

## 8.1 Why PostgreSQL Needs Special Instrumentation

Unlike the .NET application (where you control the code), PostgreSQL is a black box from an instrumentation perspective. You can't add OpenTelemetry to it. Instead, PostgreSQL exposes its internal state through **system views** (`pg_stat_*`), and `postgres_exporter` translates those views into Prometheus metrics.

`postgres_exporter` connects to PostgreSQL like a regular client and runs queries against the `pg_stat_*` views on every scrape. The results become Prometheus metrics.

---

## 8.2 The Key PostgreSQL Statistics Views

### pg_stat_activity — The Running Process List

```sql
-- Who is connected and what are they doing?
SELECT
    pid,
    usename,
    application_name,
    state,          -- 'active', 'idle', 'idle in transaction', 'waiting'
    wait_event_type,
    wait_event,
    query_start,
    NOW() - query_start AS query_duration,
    LEFT(query, 100) AS query_snippet
FROM pg_stat_activity
WHERE datname = 'obsdb'
ORDER BY query_start;
```

This view is the most important for saturation analysis. Key states:
- `active`: Running a query right now
- `idle`: Connected but not doing anything (potential pool bloat)
- `idle in transaction`: Started a transaction but hasn't done anything — **dangerous**, can hold locks
- `waiting`: Blocked waiting for a lock — **serious**, indicates contention

### pg_stat_database — Per-Database Aggregate Stats

```sql
SELECT
    datname,
    numbackends,            -- Active connections
    xact_commit,            -- Committed transactions
    xact_rollback,          -- Rolled-back transactions
    blks_read,              -- Disk reads
    blks_hit,               -- Buffer cache hits
    blks_hit::float / NULLIF(blks_hit + blks_read, 0) AS cache_hit_ratio,
    tup_returned,           -- Rows returned by SELECT
    tup_fetched,            -- Rows fetched (filtered by index)
    tup_inserted,           -- Rows inserted
    tup_updated,            -- Rows updated
    tup_deleted,            -- Rows deleted
    deadlocks,              -- Deadlock count (should be 0)
    temp_bytes              -- Temp files used (indicates sort/hash spill to disk)
FROM pg_stat_database
WHERE datname = 'obsdb';
```

**The buffer cache hit ratio** is critical. If it drops below 95%, PostgreSQL is reading data from disk instead of memory. This dramatically impacts latency.

### pg_stat_statements — Query Performance

```sql
-- Requires pg_stat_statements extension (enabled in our docker-compose)
SELECT
    calls,
    total_exec_time / calls   AS avg_ms,
    max_exec_time              AS max_ms,
    stddev_exec_time           AS stddev_ms,
    rows / calls               AS avg_rows,
    100 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0)
                               AS cache_hit_ratio,
    LEFT(query, 150)           AS query
FROM pg_stat_statements
WHERE calls > 50
ORDER BY total_exec_time DESC
LIMIT 20;
```

This shows you **the actual worst queries** — by total time, not average. A query called 10,000 times at 10ms total is less important than a query called 100 times at 1,000ms total.

### pg_locks — Lock Contention

```sql
-- Current lock waits
SELECT
    blocked.pid             AS blocked_pid,
    blocked_activity.query  AS blocked_query,
    blocking.pid            AS blocking_pid,
    blocking_activity.query AS blocking_query,
    NOW() - blocked_activity.query_start AS wait_duration
FROM pg_locks AS blocked
JOIN pg_stat_activity AS blocked_activity ON blocked.pid = blocked_activity.pid
JOIN pg_locks AS blocking
    ON blocking.locktype = blocked.locktype
    AND blocking.relation IS NOT DISTINCT FROM blocked.relation
    AND blocking.granted = true
    AND blocking.pid != blocked.pid
JOIN pg_stat_activity AS blocking_activity ON blocking.pid = blocking_activity.pid
WHERE NOT blocked.granted;
```

### pg_stat_user_tables — Table Access Patterns

```sql
SELECT
    relname               AS table_name,
    seq_scan,             -- Full table scans (high = missing index)
    seq_tup_read,         -- Rows read by full scans
    idx_scan,             -- Index scans
    idx_tup_fetch,        -- Rows fetched via index
    n_tup_ins,            -- Inserts
    n_tup_upd,            -- Updates
    n_tup_del,            -- Deletes
    n_live_tup,           -- Live rows
    n_dead_tup,           -- Dead rows (awaiting VACUUM)
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY seq_scan DESC;
```

**High `seq_scan` on large tables** = missing index. Every full table scan you see here is potentially a slow query visible in your latency metrics.

---

## 8.3 Default postgres_exporter Metrics

After installing postgres_exporter, these metrics appear in Prometheus automatically:

```
# Connection count
pg_stat_database_numbackends{datname="obsdb"} 23

# Transaction rates
pg_stat_database_xact_commit_total{datname="obsdb"} 147823
pg_stat_database_xact_rollback_total{datname="obsdb"} 12

# Cache hit ratio components
pg_stat_database_blks_hit_total{datname="obsdb"}  9283712
pg_stat_database_blks_read_total{datname="obsdb"} 18294

# Deadlocks (should be 0)
pg_stat_database_deadlocks_total{datname="obsdb"} 0

# Temp bytes (spilling to disk)
pg_stat_database_temp_bytes_total{datname="obsdb"} 0

# Lock metrics
pg_locks_count{datname="obsdb", mode="RowShareLock"} 15
pg_locks_count{datname="obsdb", mode="RowExclusiveLock"} 3

# Table metrics
pg_stat_user_tables_seq_scan{relname="orders"} 182
pg_stat_user_tables_idx_scan{relname="orders"} 49283
pg_stat_user_tables_n_dead_tup{relname="orders"} 4823
```

---

## 8.4 The Most Useful PostgreSQL PromQL Queries

### Buffer Cache Hit Ratio (Should Be > 95%)
```promql
# Current buffer cache hit ratio
rate(pg_stat_database_blks_hit_total{datname="obsdb"}[5m])
/
(
  rate(pg_stat_database_blks_hit_total{datname="obsdb"}[5m])
  + rate(pg_stat_database_blks_read_total{datname="obsdb"}[5m])
)
```

### Active Connections vs Maximum
```promql
# Connection saturation ratio
pg_stat_database_numbackends{datname="obsdb"}
/ 50  # max_connections from postgresql.conf
```

### Transaction Rate
```promql
# Transactions per second
rate(pg_stat_database_xact_commit_total{datname="obsdb"}[5m])
+ rate(pg_stat_database_xact_rollback_total{datname="obsdb"}[5m])
```

### Rollback Ratio (High = Application Errors)
```promql
rate(pg_stat_database_xact_rollback_total{datname="obsdb"}[5m])
/
rate(pg_stat_database_xact_commit_total{datname="obsdb"}[5m])
```

### Dead Tuple Bloat (Need for VACUUM)
```promql
# Ratio of dead rows to live rows per table
pg_stat_user_tables_n_dead_tup{relname="orders"}
/ pg_stat_user_tables_n_live_tup{relname="orders"}
```

### Full Table Scan Rate (Index Missing?)
```promql
# Rate of sequential scans — spike here = query plan changed
rate(pg_stat_user_tables_seq_scan{relname="orders"}[5m])
```

### Rows Returned vs Rows Fetched (Index Efficiency)
```promql
# High ratio = index is very selective
# Low ratio = index scans many rows then filters — may need better index
rate(pg_stat_database_tup_fetched_total{datname="obsdb"}[5m])
/ rate(pg_stat_database_tup_returned_total{datname="obsdb"}[5m])
```

---

## 8.5 Detecting the Missing Index Problem

One of the most common performance issues in PostgreSQL is a missing index causing full table scans. Here's how observability detects it:

**What you'll see in Grafana:**
1. `seq_scan` rate for the `orders` table spikes
2. `db_client_operation_duration_seconds` P99 climbs
3. `process_cpu_time_seconds_total` for the postgres container rises
4. API latency P99 climbs (because DB calls are slower)

**What you'll see in Jaeger:**
- The span for `SELECT FROM orders WHERE customer_email = $1` shows `db.rows_affected = 10000` (full scan) with duration 4,000ms

**The fix:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_email ON orders(customer_email);
```

**What happens after the fix (visible in the same dashboards within 2 minutes):**
- `seq_scan` drops back to near 0
- `idx_scan` increases
- DB query latency drops from 4,000ms to 2ms
- API P99 latency drops proportionally

This is the observability feedback loop in action: you saw the problem, fixed it, and confirmed the fix worked — all without restarting anything or asking anyone.

---

## 8.6 pg_stat_statements in Prometheus

Custom queries in `queries.yaml` expose the top queries:

```promql
# Average query duration for each query shape
pg_slow_queries_avg_ms

# Calls per minute for the slowest query
rate(pg_slow_queries_calls[5m]) * 60
```

In practice, you'll correlate these with Jaeger: Jaeger shows *which span* is slow; `pg_stat_statements` shows *which SQL text* is responsible across all requests.
