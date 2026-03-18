# Chapter 3: RED, USE & SLOs — Frameworks for Thinking About Performance

> **Part I — Foundations**

The four golden signals are the *what*. RED and USE are the *how* — structured methodologies for systematically applying those signals to different types of system components.

---

## 3.1 The RED Method (For Services)

**RED** was introduced by Tom Wilkie at Weaveworks. It applies to **request-handling services** — anything that receives and processes requests: HTTP APIs, gRPC services, message consumers.

| Letter | Signal | Question |
|---|---|---|
| **R** | Rate | How many requests per second is this service handling? |
| **E** | Errors | What fraction of those requests are failing? |
| **D** | Duration | How long do requests take? |

### Applying RED to Our .NET API

For the `/api/orders` endpoint:

```
Rate:     rate(http_requests_total{route="/api/orders"}[5m])
Errors:   rate(http_requests_total{route="/api/orders", status=~"5.."}[5m])
          / rate(http_requests_total{route="/api/orders"}[5m])
Duration: histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket{route="/api/orders"}[5m])
          )
```

**The power of RED:** If Rate is normal, Errors are 0%, and Duration is meeting SLO, the service is healthy — end of story. You don't need to look at anything else.

RED is a **triage protocol**. Start here. If these are fine, users are fine.

### RED Dashboard Layout

A good RED dashboard has three rows per service:

```
Row 1: Rate (RPS)         ← Traffic volume and trends
Row 2: Error Rate (%)     ← Health signal
Row 3: Duration (P50/P95/P99) ← Performance signal
```

---

## 3.2 The USE Method (For Resources)

**USE** was introduced by Brendan Gregg. It applies to **infrastructure resources** — CPU, memory, disks, network, connection pools.

| Letter | Signal | Question |
|---|---|---|
| **U** | Utilization | What fraction of the resource's capacity is being used? |
| **S** | Saturation | How much work is queued waiting for the resource? |
| **E** | Errors | Is the resource returning errors? |

The key insight: **utilization predicts saturation, saturation predicts latency**. By the time latency degrades, you're already saturated. By the time you're saturated, you should have caught it in utilization.

### Applying USE to PostgreSQL

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| **CPU** | `pg_cpu_usage_percent` | N/A (CPU has no queue) | N/A |
| **Connections** | `pg_stat_activity.count / max_connections` | `pg_stat_activity` waiting on lock/connection | `pg_stat_database.deadlocks` |
| **Disk I/O** | `pg_io_utilization_percent` | I/O wait time | `pg_stat_bgwriter.buffers_backend_fsync` |
| **Memory** | `pg_shared_buffers_used / pg_shared_buffers` | `pg_stat_bgwriter.checkpoints_req` (forced) | — |
| **Locks** | `pg_locks.granted = true / total` | `pg_locks.granted = false` (waiting) | `pg_stat_database.deadlocks` |

### Applying USE to .NET Runtime

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| **Thread Pool** | `ThreadPool.ThreadCount / ThreadPool.GetMaxThreads` | `ThreadPool.PendingWorkItemCount` | — |
| **Memory** | `GC.GetTotalMemory / process.WorkingSet` | `GC.CollectionCount` (Gen2 frequency) | `OutOfMemoryException` count |
| **HTTP Connections** | `HttpClient connections active / max` | `HttpClient queue depth` | `SocketException` count |
| **CPU** | `process.cpu_usage` | — | — |

---

## 3.3 RED vs USE — Which One When?

| Component Type | Method | Reason |
|---|---|---|
| HTTP API endpoints | RED | Request-driven; user experience is measured in requests |
| gRPC services | RED | Same — request-driven |
| Background workers | RED | They process jobs — jobs are their "requests" |
| CPU | USE | Not request-driven; has utilization and saturation |
| Memory | USE | Not request-driven |
| Database connection pool | USE | A pool is a resource with capacity |
| Disk | USE | A resource with capacity |
| The database as a service | RED | Also receives queries as "requests" |
| The database as a resource | USE | Has CPU, connections, disk |

**Use both.** RED for the API layer, USE for the infrastructure layer. Together they cover the full picture.

---

## 3.4 Service Level Objectives (SLOs)

An SLO is a target value for a service level indicator (SLI), measured over a time window.

### SLIs: What You Measure

SLIs are the specific metrics you use to assess whether your service is meeting its commitments. Good SLIs:

- Are measurable in real time
- Directly represent user experience
- Have clear good/bad thresholds

| SLI Type | Example |
|---|---|
| **Availability** | `% of requests returning non-5xx` |
| **Latency** | `% of requests completing in < 500ms` |
| **Throughput** | `requests processed per second > 100` |
| **Error rate** | `% of requests returning errors < 1%` |

### SLOs: Your Targets

```
SLO: 99.5% of /api/orders requests complete in < 500ms, measured over 30 days.

Translation:
  - In a 30-day period (2,592,000 seconds)
  - Of all requests to /api/orders
  - 99.5% must complete within 500ms
  - 0.5% may take longer — this is your error budget
```

### Error Budgets: The Most Important Concept

**Error budget = 100% - SLO target**

For a 99.5% SLO: `error budget = 0.5%`

In a 30-day window with 1,000 requests/minute:
```
Total requests:  1,000 × 60 × 24 × 30 = 43,200,000
Error budget:    43,200,000 × 0.005    = 216,000 slow requests allowed
```

The error budget is what makes SLOs actionable:
- **Budget > 50% remaining:** Deploy freely. Reliability is good.
- **Budget 25–50% remaining:** Be cautious with risky deployments.
- **Budget < 25% remaining:** Freeze risky changes. Focus on reliability.
- **Budget exhausted:** No feature deployments until next window. Fix reliability first.

### Prometheus SLO Recording Rules

```yaml
# In Prometheus rules file

groups:
  - name: slo-rules
    interval: 1m
    rules:

    # SLI: fraction of requests completing in < 500ms
    - record: slo:api_latency_sli:ratio_rate5m
      expr: |
        sum(rate(http_request_duration_seconds_bucket{
          route="/api/orders", le="0.5"
        }[5m]))
        /
        sum(rate(http_request_duration_seconds_count{
          route="/api/orders"
        }[5m]))

    # SLI: availability (non-5xx)
    - record: slo:api_availability_sli:ratio_rate5m
      expr: |
        1 - (
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))
        )

    # Error budget burn rate (30d window, 1h indicator)
    - record: slo:error_budget_burn_rate:ratio_rate1h
      expr: |
        (1 - slo:api_availability_sli:ratio_rate5m) / (1 - 0.995)
```

---

## 3.5 The Percentile Lie: Understanding Histograms

One of the most misunderstood things in performance monitoring is what percentiles mean and how they should be calculated.

### Wrong Way: Client-Side Summaries

Many libraries compute percentiles on the client (in the application process) and export the pre-computed values to Prometheus. This is called a **summary**.

```
# Bad: pre-computed in the application
http_request_duration_seconds{quantile="0.99"} 0.847
```

**Why this is wrong:** You cannot aggregate pre-computed percentiles across multiple instances. If instance A reports P99 = 100ms and instance B reports P99 = 900ms, the true P99 across both instances is *not* 500ms. There is no mathematical operation to combine them correctly.

### Right Way: Server-Side Histograms

A histogram exports a counter for each latency bucket:

```
# Good: raw bucket counts, aggregated by Prometheus
http_request_duration_seconds_bucket{le="0.1"}   1450
http_request_duration_seconds_bucket{le="0.25"}  2100
http_request_duration_seconds_bucket{le="0.5"}   2400
http_request_duration_seconds_bucket{le="1.0"}   2490
http_request_duration_seconds_bucket{le="2.5"}   2499
http_request_duration_seconds_bucket{le="+Inf"}  2500
```

Prometheus can now compute the percentile server-side with `histogram_quantile()`, and it works correctly across any number of instances:

```promql
# P99 correctly aggregated across all API instances
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

**Always use histograms, not summaries, for latency metrics.** .NET's OpenTelemetry SDK generates histograms by default. Never use `Histogram.Observe` with pre-defined quantiles.

---

## 3.6 Choosing Bucket Boundaries

Histogram accuracy depends on your bucket boundaries being appropriate for your expected latency range. The `histogram_quantile()` function uses linear interpolation within a bucket, so coarse buckets give imprecise results.

```csharp
// Good bucket boundaries for an API expected to respond in 1-500ms
var boundaries = new double[] {
    0.005,   //   5ms
    0.010,   //  10ms
    0.025,   //  25ms
    0.050,   //  50ms
    0.100,   // 100ms
    0.250,   // 250ms
    0.500,   // 500ms  ← SLO threshold
    1.000,   //   1s
    2.500,   // 2.5s
    5.000,   //   5s   ← timeout threshold
    10.000,  //  10s
};
```

If your SLO is "99% of requests under 500ms", ensure `0.5` is a bucket boundary. This makes `histogram_quantile(0.99, ...)` accurate at exactly the boundary that matters.

---

## 3.7 Summary: The Mental Model for This Book

By the end of the next section you will have a running system that demonstrates all of this. Keep this mental model:

```
Users send requests
    │
    ▼ (measure with RED)
Your .NET API handles them
    │
    ├── Correctly?     → Error rate
    ├── Fast enough?   → Duration (P95, P99)
    └── How many?      → Rate (RPS)
    │
    ▼ (measure with USE)
PostgreSQL stores/retrieves data
    ├── How busy?      → Utilization
    ├── Any queue?     → Saturation
    └── Any errors?    → Errors (deadlocks, failed queries)
    │
    ▼
Combined in Grafana as:
    ├── RED dashboard  → "Is the service healthy?"
    ├── USE dashboard  → "Is the infrastructure healthy?"
    └── SLO dashboard  → "Are we meeting our commitments?"
```

Part II builds this stack from scratch.
