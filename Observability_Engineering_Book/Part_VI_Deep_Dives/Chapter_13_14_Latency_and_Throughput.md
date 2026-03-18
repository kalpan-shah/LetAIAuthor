# Chapter 13: Deep Dive — Latency Analysis

> **Part VI — Deep Dives**

---

## 13.1 What Actually Causes Latency?

Latency has four sources in a typical .NET API + PostgreSQL system:

```
Total Request Latency
 = Network (client → server)
 + Middleware processing
 + Application logic (business code)
 + External service calls
 + Database query time
   = Query planning
   + Lock wait time
   + Data read (buffer cache or disk)
   + Result serialization
 + Network (server → client)
```

Traces decompose this sum. Without traces, you can only measure the total. With traces, every term is visible.

---

## 13.2 Tail Latency: The 1% That Matters

The most important latency concept in production systems is **tail latency** — the slowest fraction of requests. Why does it matter so much?

### The Fan-Out Problem

If one request calls 10 services, each with P99 = 10ms, the composite P99 is:

```
P(all 10 complete in < 10ms) = 0.99^10 = 0.904

Therefore: P(at least one takes > 10ms) = 9.6%
```

What was a 1% outlier on each service becomes a 9.6% outlier on the composed request. In a microservices architecture with 50 dependencies, this becomes:

```
P(at least one slow) = 1 - 0.99^50 = 39.5%
```

Nearly 40% of all requests would be slow. Tail latency compounds in fan-out architectures. **Controlling P99 at each service is critical.**

### Measuring Tail Latency

```promql
# P99 over last 5 minutes
histogram_quantile(0.99,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket[5m])
  )
)

# P999 (one-in-a-thousand) — relevant at > 1000 rps
histogram_quantile(0.999,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket[5m])
  )
)

# The "max" via the +Inf bucket
# (approximate: upper bound of the slowest bucket containing data)
histogram_quantile(1.0,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket[5m])
  )
)
```

---

## 13.3 Common Latency Anti-Patterns in .NET

### Anti-Pattern 1: Sync-Over-Async

```csharp
// BAD: .Result blocks a thread pool thread while waiting
var result = _httpClient.GetAsync("/api/data").Result;

// GOOD: await frees the thread while waiting
var result = await _httpClient.GetAsync("/api/data");
```

**Observability signature:**
- `dotnet_thread_pool_queue_length` > 0 and growing
- P99 latency much higher than P95 (blocking creates queue spikes)
- CPU usage LOW despite high request count (threads are blocked, not working)

---

### Anti-Pattern 2: N+1 Query Problem

```csharp
// BAD: One query to get orders, then one query per order for items
var orders = await conn.QueryAsync<Order>("SELECT * FROM orders");
foreach (var order in orders) {
    // This executes one SELECT per order — 100 orders = 101 queries
    order.Items = await conn.QueryAsync<OrderItem>(
        "SELECT * FROM order_items WHERE order_id = @id",
        new { id = order.Id }
    );
}

// GOOD: One query with JOIN
var orders = await conn.QueryAsync<Order, OrderItem, Order>(
    """
    SELECT o.*, oi.*
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    """,
    (order, item) => { order.Items.Add(item); return order; }
);
```

**Observability signature:**
- In Jaeger: the trace shows 100+ `SELECT FROM order_items` spans, each fast (2ms), but summing to 200ms
- `pg_stat_statements` shows very high `calls` count for the items query
- `rate(pg_stat_database_xact_commit_total[5m])` much higher than `rate(http_requests_total[5m])` × expected queries per request

---

### Anti-Pattern 3: Missing Async in EF Core

```csharp
// BAD: Synchronous queries block thread pool
var orders = dbContext.Orders.ToList();

// GOOD: Async queries free the thread while the DB responds
var orders = await dbContext.Orders.ToListAsync();
```

---

### Anti-Pattern 4: Large Object Heap Pressure

```csharp
// BAD: Returning large byte arrays causes LOH allocations
byte[] data = File.ReadAllBytes(largePath); // > 85KB → LOH

// GOOD: Stream instead of buffering
using var stream = File.OpenRead(largePath);
await stream.CopyToAsync(response.Body);
```

**Observability signature:**
- `dotnet_gc_heap_size_bytes{generation="loh"}` growing
- `rate(dotnet_gc_collections_total{generation="gen2"}[5m])` elevated
- `dotnet_gc_pause_ratio` > 0.02 (2% of time in GC)
- Sawtooth + rising baseline in `process_working_set_bytes`

---

## 13.4 Database Latency Deep Dive

### The EXPLAIN ANALYZE Method

When you identify a slow query in a Jaeger trace (e.g., a SELECT taking 4,000ms), the next step is understanding *why* it's slow:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, json_agg(oi.*) AS items
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.customer_email = 'user@example.com'
GROUP BY o.id;
```

**Reading the output:**
```
GroupAggregate  (cost=0.00..98423.00 rows=1 width=532)
                (actual time=3892.123..3892.457 rows=3 loops=1)
  Buffers: shared hit=2 read=24891              ← 24891 disk reads! (cache miss)
  ->  Hash Join  (cost=...)
        ->  Seq Scan on orders                   ← FULL TABLE SCAN
              Filter: (customer_email = 'user@example.com')
              Rows Removed by Filter: 49997       ← Scanned all 50000 rows
```

**The smoking gun:** `Seq Scan on orders` — scanning all 50,000 rows to find 3. Fix:
```sql
CREATE INDEX CONCURRENTLY idx_orders_customer_email ON orders(customer_email);
```

After adding the index:
```
Index Scan using idx_orders_customer_email on orders
  (actual time=0.089..0.094 rows=3 loops=1)
  Buffers: shared hit=5                         ← 5 buffer hits, no disk reads
Total time: 2.847ms                             ← Down from 3,892ms
```

### Correlating EXPLAIN with Prometheus

```promql
# Before adding index: high seq_scan rate
rate(pg_stat_user_tables_seq_scan{relname="orders"}[5m]) = 847

# After adding index: seq_scan drops, idx_scan rises
rate(pg_stat_user_tables_seq_scan{relname="orders"}[5m]) = 2
rate(pg_stat_user_tables_idx_scan{relname="orders"}[5m])  = 1240

# Buffer cache hit rate recovers
pg:buffer_cache_hit_ratio:rate5m: 0.94 → 0.999
```

---

## 13.5 Latency Budget: Knowing Where to Optimize

When a request takes too long, allocate a "latency budget" — how much each layer is *allowed* to use:

| Layer | Allowed | Actual (Bad Day) | Gap |
|---|---|---|---|
| Middleware | 5ms | 5ms | 0ms |
| Auth validation | 10ms | 12ms | 2ms |
| Business logic | 20ms | 18ms | 0ms |
| External API | 200ms | 842ms | 642ms |
| Database | 100ms | 98ms | 0ms |
| Serialization | 10ms | 12ms | 0ms |
| **Total** | **345ms** | **987ms** | **642ms** |

The trace makes this table trivially accurate. The optimization target is immediately obvious: negotiate a lower latency guarantee with the external API, add a circuit breaker, or pre-cache authorization results.

---

# Chapter 14: Deep Dive — Throughput & Saturation

> **Part VI — Deep Dives**

---

## 14.1 Little's Law: The Foundation of Throughput Analysis

**Little's Law** (from queueing theory) states:

```
L = λ × W

Where:
  L = average number of requests in the system (concurrency)
  λ = average arrival rate (throughput, requests/second)
  W = average time a request spends in the system (latency)
```

This relationship is fundamental. Rearranging:

```
Throughput = Concurrency / Latency

If latency doubles and concurrency is fixed:
  Throughput halves.

If you want to double throughput without changing latency:
  You must double concurrency (more threads, more connections, more instances).
```

### Practical Application

If your API handles 100 rps and your average latency is 100ms:
```
L = 100 rps × 0.100 s = 10 concurrent requests
```

You need at least 10 "slots" (threads/goroutines/connections) to sustain 100 rps at 100ms latency. If your thread pool has 8 workers, you will queue. This is saturation.

---

## 14.2 Measuring Throughput

```promql
# Current RPS (all routes)
sum(rate(http_server_request_duration_seconds_count[5m]))

# RPS by route
sum by (route, method) (
  rate(http_server_request_duration_seconds_count[5m])
)

# Successful RPS (non-5xx)
sum(rate(http_server_request_duration_seconds_count{status_code!~"5.."}[5m]))

# DB transaction throughput
rate(pg_stat_database_xact_commit_total{datname="obsdb"}[5m])

# Cache throughput
rate(cache_hits_total[5m]) + rate(cache_misses_total[5m])
```

---

## 14.3 Throughput Ceilings: Finding Your Limits

Every system has a throughput ceiling — the maximum sustainable request rate before latency degrades unacceptably.

**How to find yours:**
1. Run k6 with increasing virtual users
2. Watch the latency percentile panels in Grafana
3. The ceiling is when P95 latency begins rising faster than linearly with load

```javascript
// load-test.js: ramp to find the ceiling
export const options = {
  stages: [
    { duration: '2m', target: 10  },
    { duration: '2m', target: 25  },
    { duration: '2m', target: 50  },
    { duration: '2m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '2m', target: 400 },
    { duration: '2m', target: 0   },
  ],
};
```

The resulting Grafana graph will show latency increasing steeply at the ceiling point. That's your capacity limit.

---

## 14.4 Saturation: The Early Warning System

Saturation is the most powerful signal because it predicts problems before they happen. These are the saturation metrics for our system:

### Application-Level Saturation

```promql
# Thread pool queue depth (> 0 = saturated)
dotnet_thread_pool_queue_length

# Thread pool utilization
dotnet_thread_pool_threads_count / 64  # 64 = default max worker threads on 4-core
```

**Queue depth > 0** means the thread pool has no free workers. New work is queuing. If this persists, latency climbs, then errors start (timeouts), then cascading failures begin.

### Database Connection Pool Saturation

```promql
# Pending connections in the Npgsql pool — THE most critical saturation signal
db_client_connections_usage{state="pending", pool_name=~".*obsdb.*"}
```

When `pending > 0`, your application is waiting for a database connection. Every millisecond of connection wait adds directly to request latency.

**Root causes:**
- Too many concurrent requests (scale out)
- Pool size too small for the traffic (increase `Maximum Pool Size` in connection string)
- Queries taking too long (queries hold connections; slow queries = fewer connections for others)
- Transaction not closed promptly (connections held open unnecessarily)

```csharp
// BAD: Connection held for the entire business logic duration
var conn = await _db.OpenConnectionAsync();
var order = await _someService.DoComplexBusiness(); // conn is held during this!
await conn.ExecuteAsync("INSERT ...");
conn.Dispose();

// GOOD: Open connection only when needed, close immediately after
var order = await _someService.DoComplexBusiness(); // no connection held
await using var conn = await _db.OpenConnectionAsync();
await conn.ExecuteAsync("INSERT ...");
// conn disposed at end of using block
```

### Redis Saturation

```promql
# Memory utilization
redis_memory_used_bytes / redis_config_maxmemory_bytes

# Blocked clients (waiting for BLPOP/BRPOP)
redis_blocked_clients

# Eviction rate (> 0 = memory full, data being lost)
rate(redis_evicted_keys_total[5m])
```

---

## 14.5 The Saturation → Latency → Error Cascade

This is the pattern you'll see when a system is overloaded:

```
T+0:00  DB connection pool: 8/10 connections used (80%)
T+0:30  Traffic spike: DB pool reaches 10/10 (100% saturation)
T+1:00  Pending connections: 0 → 5 → 15 (queue building)
T+1:30  API P95 latency: 200ms → 800ms (connection wait added)
T+2:00  API P99 latency: 500ms → 4000ms (some requests queue for seconds)
T+2:30  Client timeouts start: error rate 0% → 2% → 8%
T+3:00  Frustrated clients retry: traffic spike amplifies the problem
T+3:30  Full cascade: system appears "down" despite pool only being full
```

**The intervention that works:**
1. Immediately identify: `db_client_connections_usage{state="pending"}` > 0 (Prometheus alert)
2. Identify root cause: is it traffic spike, slow queries, or pool too small?
3. Short-term: scale API horizontally (more pods = more pool connections available)
4. Medium-term: fix the slow queries that hold connections too long
5. Long-term: right-size the connection pool and consider PgBouncer for connection multiplexing

---

## 14.6 PgBouncer: Connection Pool Multiplexing

When connection saturation is structural (many API instances × minimum pool size > pg's max_connections), consider PgBouncer:

```yaml
# Add to docker-compose.yml
pgbouncer:
  image: edoburu/pgbouncer:1.22.1
  environment:
    DB_HOST:     postgres
    DB_NAME:     obsdb
    DB_USER:     obs
    DB_PASSWORD: obs123
    POOL_MODE:   transaction    # Best mode for stateless APIs
    MAX_CLIENT_CONN: 500        # Clients can request up to 500 connections
    DEFAULT_POOL_SIZE: 20       # But only 20 are used toward PostgreSQL
  ports:
    - "5433:5432"
  networks: [obs-net]
```

In `transaction` pool mode, a PostgreSQL connection is borrowed for the duration of a single transaction, then returned. This allows 500 client connections to share 20 PostgreSQL connections, because no real PostgreSQL query happens between transactions.

**Prometheus impact:** After adding PgBouncer:
- `pg_stat_database_numbackends` drops from ~50 to ~20
- `db_client_connections_usage{state="pending"}` drops to 0
- API latency P95 drops by the amount previously spent waiting for connections
