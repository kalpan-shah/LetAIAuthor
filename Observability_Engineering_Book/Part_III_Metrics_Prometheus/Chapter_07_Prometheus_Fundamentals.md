# Chapter 7: Prometheus Fundamentals & PromQL

> **Part III — Metrics with Prometheus**

---

## 7.1 What Prometheus Actually Is

Prometheus is a **pull-based time-series database** with a built-in query language. It scrapes metrics endpoints (like `/metrics`) at regular intervals, stores the data locally, and makes it queryable.

```
Your .NET App               Prometheus
  /metrics endpoint   ←────── scrape every 15s ──────
  Exposes current values     Stores as time series
                             Evaluates alert rules
                             Answers PromQL queries
```

"Pull-based" means Prometheus decides when to scrape — not the application. This is different from systems like StatsD where the application pushes metrics. Pull-based has important advantages:
- If the application is down, Prometheus knows (scrape fails)
- No need to configure where to send metrics (Prometheus finds the target)
- Service discovery can dynamically update the scrape list

---

## 7.2 The Data Model

Every Prometheus time series is uniquely identified by:
1. A **metric name** (e.g., `http_server_request_duration_seconds_bucket`)
2. A set of **labels** (key-value pairs that add dimensions)

```
metric_name{label1="value1", label2="value2"} → sequence of (timestamp, value) pairs
```

Examples:
```
http_server_request_duration_seconds_bucket{
  route="/api/orders",
  method="GET",
  status_code="200",
  le="0.1"          ← "less than or equal to" — histogram bucket boundary
} = 1450

http_server_request_duration_seconds_bucket{
  route="/api/orders",
  method="GET",
  status_code="200",
  le="0.25"
} = 2100
```

---

## 7.3 The Four Metric Types

### Counter

A monotonically increasing number. Never decreases. Resets to 0 on restart.

```csharp
var counter = meter.CreateCounter<long>("orders.created.total");
counter.Add(1); // Each time an order is created
```

```
# Prometheus exposition format
# HELP orders_created_total Total number of orders created
# TYPE orders_created_total counter
orders_created_total 1847
```

**What you can do with a counter:**
```promql
# Rate of increase over last 5 minutes
rate(orders_created_total[5m])

# Total increase over the last hour
increase(orders_created_total[1h])
```

**When to use:** Anything that goes up: requests, errors, bytes sent, items processed.

### Gauge

A value that can go up or down. Represents a current snapshot.

```csharp
var gauge = meter.CreateObservableGauge<int>(
    "db.connections.active",
    () => GetActiveConnectionCount()
);
```

```
db_connections_active 23
```

**What you can do with a gauge:**
```promql
# Current value
db_connections_active

# How close to the limit?
db_connections_active / 50  # 50 = max_connections
```

**When to use:** Temperatures, queue depths, connection counts, memory usage.

### Histogram

Counts observations in configurable buckets. Used for latency and sizes.

```csharp
var histogram = meter.CreateHistogram<double>(
    "order.creation.duration",
    unit: "s"
);
histogram.Record(0.847); // Record a measurement of 847ms
```

In Prometheus, this creates three series:
```
# The cumulative bucket counts
order_creation_duration_seconds_bucket{le="0.1"}  245
order_creation_duration_seconds_bucket{le="0.5"}  892
order_creation_duration_seconds_bucket{le="1.0"}  1145
order_creation_duration_seconds_bucket{le="+Inf"} 1200

# Total sum of all observations
order_creation_duration_seconds_sum  847.231

# Total count of all observations
order_creation_duration_seconds_count  1200
```

**What you can do with a histogram:**
```promql
# P95 latency over last 5 minutes
histogram_quantile(0.95,
  rate(order_creation_duration_seconds_bucket[5m])
)

# Fraction of requests under 500ms (SLI)
rate(order_creation_duration_seconds_bucket{le="0.5"}[5m])
/ rate(order_creation_duration_seconds_count[5m])
```

**When to use:** Request latency (always!), response size, queue wait time.

### Summary

Pre-computes quantiles in the application process. **Generally avoid for latency.**

Summaries cannot be aggregated across instances. If you have 3 API pods and want the P95 across all of them, summaries can't give you that. Histograms can. Use histograms.

---

## 7.4 PromQL: The Query Language

PromQL is a functional query language. Every query returns either:
- An **instant vector:** one value per time series at a single point in time
- A **range vector:** a series of values over a time window
- A **scalar:** a single number

### Selectors

```promql
# Select all time series matching the metric name
http_server_request_duration_seconds_count

# Filter by label equality
http_server_request_duration_seconds_count{route="/api/orders"}

# Filter by label regex
http_server_request_duration_seconds_count{status_code=~"5.."}

# Exclude a label value
http_server_request_duration_seconds_count{route!="/metrics"}
```

### Range Vectors and Functions

```promql
# Rate: per-second rate of increase averaged over 5 minutes
# (Use this on counters to get requests-per-second)
rate(http_server_request_duration_seconds_count[5m])

# irate: instantaneous rate using last two data points
# (More responsive to spikes, noisier)
irate(http_server_request_duration_seconds_count[1m])

# increase: total increase over a time window
increase(http_server_request_duration_seconds_count[1h])

# delta: change in a gauge over a time window
delta(db_connections_active[5m])
```

### Aggregation Operators

```promql
# Sum across all routes and methods
sum(rate(http_server_request_duration_seconds_count[5m]))

# Sum, but keep the route label (group by route)
sum by (route) (rate(http_server_request_duration_seconds_count[5m]))

# Average CPU across all instances
avg by (instance) (process_cpu_time_seconds_total)

# Maximum active connections across all instances
max(db_connections_active)

# Count of instances with error rate > 5%
count(
  rate(http_server_request_duration_seconds_count{status_code=~"5.."}[5m])
  / rate(http_server_request_duration_seconds_count[5m])
  > 0.05
)
```

---

## 7.5 The Most Important PromQL Queries for Our System

### Traffic (Rate)
```promql
# Requests per second by route
sum by (route, method) (
  rate(http_server_request_duration_seconds_count[5m])
)
```

### Error Rate
```promql
# Fraction of requests returning 5xx errors
sum(rate(http_server_request_duration_seconds_count{status_code=~"5.."}[5m]))
/
sum(rate(http_server_request_duration_seconds_count[5m]))
```

### Latency (P95)
```promql
# P95 request latency, by route
histogram_quantile(0.95,
  sum by (route, le) (
    rate(http_server_request_duration_seconds_bucket[5m])
  )
)
```

### Cache Hit Rate
```promql
# Fraction of lookups served from cache
rate(cache_hits_total[5m])
/
(rate(cache_hits_total[5m]) + rate(cache_misses_total[5m]))
```

### Database Connection Pool Saturation
```promql
# What fraction of the connection pool is in use?
db_client_connections_usage{state="used", pool_name="obsdb"}
/
(
  db_client_connections_usage{state="used",  pool_name="obsdb"}
  + db_client_connections_usage{state="idle", pool_name="obsdb"}
)
```

### External API Latency
```promql
# P95 latency of calls to external API
histogram_quantile(0.95,
  sum by (le) (
    rate(external_api_duration_seconds_bucket[5m])
  )
)
```

---

## 7.6 Prometheus Metric Naming Conventions

Good metric names follow a pattern: `{namespace}_{subsystem}_{name}_{unit}`

| Good | Bad | Why |
|---|---|---|
| `http_server_request_duration_seconds` | `api_latency` | Missing unit; ambiguous namespace |
| `db_client_connections_usage` | `connections` | Too ambiguous |
| `order_creation_errors_total` | `orderErrors` | Not snake_case; missing `_total` suffix |
| `process_cpu_seconds_total` | `cpu` | Missing unit and total suffix |

**Rules:**
- Snake_case only
- `_total` suffix for counters
- `_seconds`, `_bytes`, `_ratio` for units (always SI base units)
- `_bucket`, `_sum`, `_count` are reserved for histograms (auto-added)
- No hyphens, no camelCase

---

## 7.7 Cardinality: The Prometheus Killer

Cardinality is the number of unique label combinations. High cardinality destroys Prometheus performance.

```
# Low cardinality: 5 routes × 5 methods × 3 status classes = 75 series
http_server_request_duration_seconds_count{route, method, status_code}

# HIGH CARDINALITY: unique user IDs as labels = millions of series
http_server_request_duration_seconds_count{user_id="uuid"}  ← NEVER DO THIS
```

**What to never use as labels:**
- User IDs
- Order IDs (any UUID)
- Email addresses
- Full URLs (without normalization)
- IP addresses (unless you have a small known set)
- Timestamps

**What is fine as labels:**
- Route patterns (`/api/orders/{id}` is one label value)
- HTTP method (GET, POST, etc.)
- Status code class (2xx, 4xx, 5xx) or exact code
- Service names
- Environments (production, staging)
- Regions (us-east-1, eu-west-1)

The `.NET Minimal API` mapping already normalizes routes — `/api/orders/abc123` and `/api/orders/def456` both become `/api/orders/{id}` as a label. This is correct behaviour.
