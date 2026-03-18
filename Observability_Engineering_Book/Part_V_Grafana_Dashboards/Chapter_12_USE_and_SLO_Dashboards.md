# Chapter 12: Grafana — USE Dashboard & SLO / Availability Dashboard

> **Part V — Grafana Dashboards**

---

## 12.1 The USE Dashboard

The USE dashboard focuses on infrastructure resources: the things your application runs *on* and *in*. Where the RED dashboard answers "is the service healthy?", the USE dashboard answers "is the infrastructure that supports the service healthy?"

---

### USE Row 1: .NET Runtime Health

#### Panel: Thread Pool

```promql
# Worker threads in use
dotnet_thread_pool_threads_count

# Work items queued (saturation indicator)
dotnet_thread_pool_queue_length
```

**Visualization:** Two time series overlaid.

**What to look for:**
- `queue_length` rising above 0 = thread pool is saturated; requests are queuing at the runtime level. This happens when you have too many blocking (synchronous) operations.
- `threads_count` near the maximum = thread pool has exhausted its allocated threads.

**When it fires:**
1. Sync-over-async code (calling `.Result` or `.Wait()` on async methods)
2. CPU-intensive synchronous work
3. Too many concurrent long-running operations

---

#### Panel: Garbage Collector

```promql
# Gen2 GC collection rate — the expensive one
rate(dotnet_gc_collections_total{generation="gen2"}[5m])

# GC pause ratio — fraction of time spent paused in GC
dotnet_gc_pause_ratio

# Heap size by generation
dotnet_gc_heap_size_bytes
```

**Thresholds:**
- Gen2 > 0.1 collections/sec → Warning
- GC pause ratio > 0.05 (5% of time in GC) → Warning

**What causes Gen2 pressure:**
- Large objects (> 85KB) that skip Gen0/Gen1 directly to the Large Object Heap (LOH)
- Memory leaks that cause object promotion through generations
- Large, long-lived caches

---

#### Panel: Memory Usage

```promql
# Working set (physical RAM used by the process)
process_working_set_bytes / 1024 / 1024

# Private memory (committed virtual address space)
process_private_memory_bytes / 1024 / 1024

# GC heap (managed heap only)
dotnet_gc_heap_size_bytes / 1024 / 1024
```

**Visualization:** Stacked area chart
**Unit:** Megabytes

**Alarm pattern:** Linear growth over hours/days = memory leak. Sawtooth pattern = normal GC behaviour.

---

#### Panel: CPU

```promql
# CPU seconds consumed per second = CPU percentage
rate(process_cpu_time_seconds_total[5m]) * 100
```

**Unit:** Percent
**Threshold:** > 80% sustained for 5 minutes = Warning

---

### USE Row 2: HTTP Connection Pool

```promql
# Active HTTP client connections (to external API)
http_client_open_connections{state="active"}

# Idle connections (wasted but available)
http_client_open_connections{state="idle"}

# Duration of establishing new connections (saturation indicator)
histogram_quantile(0.95,
  rate(http_client_connection_duration_seconds_bucket[5m])
)
```

---

### USE Row 3: PostgreSQL

#### Panel: Connection Pool Saturation

```promql
# .NET-side connection pool (what the app sees)
db_client_connections_usage{state="used",  pool_name=~".*obsdb.*"}
db_client_connections_usage{state="idle",  pool_name=~".*obsdb.*"}
db_client_connections_usage{state="pending", pool_name=~".*obsdb.*"}  # Saturation!

# DB-side connections (what PostgreSQL sees)
pg_stat_database_numbackends{datname="obsdb"}
```

**The saturation indicator is `pending` connections** — requests waiting for a connection to become available. When this is non-zero, your application is queuing work at the connection pool level. Latency will rise.

---

#### Panel: Buffer Cache Hit Rate

```promql
pg:buffer_cache_hit_ratio:rate5m
```

**Visualization:** Gauge (not time series)
**Thresholds:**
- Green: > 99%
- Yellow: 95–99%
- Red: < 95%

**If this is low:** Your `shared_buffers` setting in PostgreSQL is too small for your working set. Either increase it (typically 25–40% of RAM) or investigate queries that read entire tables.

---

#### Panel: Database Transaction Rate

```promql
# Commits per second
rate(pg_stat_database_xact_commit_total{datname="obsdb"}[5m])

# Rollbacks per second (should be very low)
rate(pg_stat_database_xact_rollback_total{datname="obsdb"}[5m])

# Rollback ratio (should be near 0%)
rate(pg_stat_database_xact_rollback_total{datname="obsdb"}[5m])
/ rate(pg_stat_database_xact_commit_total{datname="obsdb"}[5m])
```

---

#### Panel: Lock Contention

```promql
# Queries waiting on a lock right now
pg_lock_waits_waiting

# Deadlock rate
rate(pg_stat_database_deadlocks_total{datname="obsdb"}[5m])
```

**Anything > 0 for deadlocks is notable.** Deadlocks indicate concurrent transactions are accessing resources in conflicting orders. Fix by standardizing the order of table/row access across all transactions.

---

#### Panel: Table Bloat

```promql
# Dead tuple ratio per table
pg_stat_user_tables_n_dead_tup{relname="orders"}
/ pg_stat_user_tables_n_live_tup{relname="orders"}
```

**Threshold:** > 20% dead tuples → VACUUM needed urgently.

**Side effect of high bloat:** Table scans read more pages (including dead tuples), using more I/O and memory. Indexes also bloat. Performance degrades gradually.

---

#### Panel: Sequential Scans (Missing Index Detector)

```promql
# Rate of sequential scans — a spike means a query plan changed or an index was dropped
rate(pg_stat_user_tables_seq_scan{relname="orders"}[5m])
rate(pg_stat_user_tables_seq_scan{relname="order_items"}[5m])
```

---

### USE Row 4: Redis

```promql
# Memory usage vs max
redis_memory_used_bytes / redis_config_maxmemory_bytes

# Hit rate
rate(redis_keyspace_hits_total[5m])
/ (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

# Evictions (cache items being removed due to memory pressure)
rate(redis_evicted_keys_total[5m])

# Connected clients
redis_connected_clients
```

**If eviction rate > 0:** Redis is full. Increase `maxmemory`, reduce TTLs, or cache fewer items.

---

## 12.2 The SLO & Availability Dashboard

This dashboard answers: "Are we keeping our promises?"

### Panel 1: Current SLO Status (Stat panels)

```promql
# Availability SLO: 99.9% — current 30-day value
(
  sum(increase(http_server_request_duration_seconds_count{status_code!~"5.."}[30d]))
  / sum(increase(http_server_request_duration_seconds_count[30d]))
) * 100
```
**Unit:** Percent
**Color thresholds:**
- Green: > 99.9 (meeting SLO)
- Red: < 99.9 (breaching SLO)

```promql
# Latency SLO: 95% of requests < 500ms (30-day)
sum(increase(http_server_request_duration_seconds_bucket{le="0.5"}[30d]))
/ sum(increase(http_server_request_duration_seconds_count[30d])) * 100
```

---

### Panel 2: Error Budget Remaining (Gauge)

```promql
# Remaining availability error budget (%)
(
  sum(increase(http_server_request_duration_seconds_count{status_code!~"5.."}[30d]))
  / sum(increase(http_server_request_duration_seconds_count[30d]))
  - 0.999
) / (1 - 0.999) * 100
```

**Visualization:** Gauge from 0 to 100
**Thresholds:**
- Green: 50–100% remaining
- Yellow: 25–50% remaining
- Orange: 10–25% remaining
- Red: 0–10% remaining

This is the most important panel on the dashboard. If this gauge is in the red zone, all risky deployments should be frozen.

---

### Panel 3: Error Budget Burn Rate (Time Series)

```promql
# Burn rate over 1-hour window
slo:error_budget_burn_rate:ratio_rate1h

# Burn rate over 6-hour window (less noisy, shows trends)
slo:error_budget_burn_rate:ratio_rate6h
```

Add threshold lines:
- 1.0 = "on pace" (exactly using the budget at the right rate)
- 6.0 = "alert threshold" (will exhaust budget in 5 days)
- 14.4 = "critical threshold" (will exhaust budget in 2 days)

---

### Panel 4: Error Budget Consumption Over Time (Bar Chart)

```promql
# Daily error budget consumed (what fraction of daily budget each day used)
sum_over_time(
  (1 - sli:request_success_rate:ratio_rate5m)[30d:1d]
) / (1 - 0.999)
```

**Visualization:** Bar chart (one bar per day)

This shows whether bad days are isolated events or a growing trend. If the bars are getting taller over time, reliability is degrading.

---

### Panel 5: SLO Burn Rate Alert Status (Table)

```
SLO             | Current | 1h Burn | 6h Burn | Status
Availability    | 99.93%  | 0.8x    | 1.1x    | ✅ OK
P95 Latency     | 94.1%   | 12.0x   | 8.4x    | 🔴 BREACHING
P99 Latency     | 98.7%   | 2.1x    | 1.8x    | ⚠️ ELEVATED
```

This table gives the on-call engineer a complete SLO health summary in one glance.

---

## 12.3 Dashboard Design Principles

### Visual Hierarchy

Put the most important information at the top-left (where eyes go first):
1. Current SLO status (top row, stat panels)
2. Error budget remaining (gauge, prominent)
3. Trend over time (lower rows)

### Colour Consistency

Use consistent colours across all dashboards:
- **Green:** Healthy, within SLO
- **Yellow/Orange:** Warning, elevated but not breaching
- **Red:** Breaching SLO or critical alert

### Time Range Defaults

| Dashboard | Default Time Range | Reasoning |
|---|---|---|
| RED Dashboard | Last 1 hour | Operational; you're investigating now |
| USE Dashboard | Last 1 hour | Operational; current resource state |
| SLO Dashboard | Last 30 days | SLOs are defined over 30-day windows |

### Annotations

Add Prometheus alert firing events as annotations on all time series:

In Dashboard Settings → Annotations → Add:
```
Data source: Prometheus
Expression:  ALERTS{alertname=~".*"}
Step:        60s
```

This adds vertical lines to all time series showing exactly when alerts fired — invaluable for correlating "when did the alert fire?" with "what was the metric doing at that exact time?"
