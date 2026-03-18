# Chapter 10: Distributed Tracing with Jaeger

> **Part IV — Traces with Jaeger**

---

## 10.1 Why Metrics Are Not Enough

Imagine you get an alert: API P95 latency exceeded 500ms. You open Grafana and see:
- Error rate: 0.2% (within SLO)
- Request rate: 450 rps (normal)
- P95 latency: 1,200ms (bad)
- P99 latency: 4,800ms (very bad)

The metrics tell you **something is slow** and roughly **how slow**. But they cannot tell you:
- Is the slowness in the application code? The database? The external API?
- Is one route slow or all routes?
- Is it a specific database query or all queries?
- Is it every request or just some of them?

For that, you need **traces**.

A trace shows the complete timeline of a single request as it flows through all the components of your system. It answers: "What happened, in what order, for how long?"

---

## 10.2 Anatomy of a Trace

```
Trace ID: 4f8b9a2c1d3e5f6a7b8c9d0e1f2a3b4c    Total: 1,247ms
│
├── [ROOT] POST /api/orders                    1,247ms  (HTTP span)
│   ├── Auth middleware                           12ms
│   ├── ExternalApiClient.AuthorizeOrderAsync    842ms  ← SLOW
│   │   └── HTTP POST https://payment-api/...   842ms  (HttpClient span)
│   │       Status: 200 OK
│   ├── OrderService.CreateOrderAsync           381ms
│   │   ├── BEGIN TRANSACTION                     1ms   (Npgsql span)
│   │   ├── INSERT INTO orders ...               82ms   (Npgsql span)
│   │   ├── INSERT INTO order_items ...          14ms   (Npgsql span, ×3)
│   │   ├── COMMIT                                3ms   (Npgsql span)
│   │   └── Redis SET order:uuid                  8ms   (StackExchange.Redis)
│   └── JSON serialization                       12ms
```

This trace shows that 842ms out of 1,247ms (67%) is spent waiting for the external payment API. The database operations are fine (99ms total). Without the trace, you would have blamed the database.

---

## 10.3 Core Concepts

### Span

A single unit of work with a start time, duration, and set of attributes.

```
Span {
    trace_id:    "4f8b9a2c..."   // Same for all spans in one request
    span_id:     "e4f5a6b7"     // Unique to this span
    parent_id:   "a1b2c3d4"     // ID of the parent span
    name:        "SELECT orders" // Human-readable operation name
    start_time:  2025-06-15T10:23:45.100Z
    end_time:    2025-06-15T10:23:45.182Z
    duration:    82ms
    status:      OK
    attributes: {
        "db.system":     "postgresql",
        "db.statement":  "SELECT * FROM orders WHERE id = $1",
        "db.row_count":  1,
        "net.peer.name": "postgres"
    }
    events: []    // Point-in-time annotations within the span
}
```

### Trace

A tree of spans sharing the same `trace_id`. Represents the complete lifecycle of one request.

### Context Propagation

The mechanism by which the trace ID travels from service to service. For HTTP calls, the W3C `traceparent` header carries it:

```
traceparent: 00-4f8b9a2c1d3e5f6a7b8c9d0e1f2a3b4c-e4f5a6b7c8d9e0f1-01
                └── trace_id ──────────────────┘ └── parent_id ──┘
```

OpenTelemetry handles this automatically for HttpClient and ASP.NET Core. You never need to write propagation code manually.

### Sampling

Recording every trace of every request would be expensive at scale. Sampling controls what fraction of traces are recorded:

| Sampling Strategy | When to Use |
|---|---|
| **Always sample** (100%) | Development, low-traffic production (< 100 rps) |
| **Head-based ratio** (e.g., 10%) | High-traffic; captures a statistical sample |
| **Tail-based** | Record all slow/error traces; sample fast/success traces |

For our demo, we always sample (100%) because traffic is low and we want to see everything.

---

## 10.4 Jaeger Architecture

```
.NET Application
    │  OTLP HTTP/gRPC (port 4318/4317)
    ▼
Jaeger Collector
    │  Stores spans
    ▼
Jaeger Storage (in-memory for demo, Elasticsearch/Cassandra for production)
    │
    ▼
Jaeger Query  ← Jaeger UI reads from this
    │
    ▼
Jaeger UI (port 16686) ← You browse traces here
```

For the demo, we use `jaegertracing/all-in-one` which bundles all components. In production, run the collector, storage, and query as separate services.

---

## 10.5 Reading Traces in the Jaeger UI

### Finding a Slow Trace

1. Open Jaeger: http://localhost:16686
2. Service: `obs-api`
3. Operation: `POST /api/orders`
4. Min Duration: `500ms` (filter for slow ones)
5. Click "Find Traces"

You'll see a list like:
```
POST /api/orders   1,247ms   15 spans   6 min ago
POST /api/orders   892ms     15 spans   8 min ago
POST /api/orders   134ms     15 spans   12 min ago
```

### Reading the Flame Graph

Click the 1,247ms trace to open it. Jaeger shows a waterfall/flame graph:

```
█████████████████████████████████████ POST /api/orders [1247ms]
  ██ Auth middleware [12ms]
  ████████████████████ ExternalApiClient [842ms]
  ██████████ OrderService.CreateOrderAsync [381ms]
      █ BEGIN [1ms]
      ████ INSERT orders [82ms]
      ██ INSERT items [14ms] × 3
      █ COMMIT [3ms]
      █ Redis SET [8ms]
  ██ Serialize [12ms]
```

The width of each bar is proportional to its duration relative to the total trace duration. The slowest span is immediately obvious.

### Span Attributes as Evidence

Click on "ExternalApiClient [842ms]":

```
Span Details:
  http.method:       POST
  http.url:          http://slowapi:4999/authorize
  http.status_code:  200
  http.duration_ms:  842
  service.name:      obs-api
  net.peer.name:     slowapi
  net.peer.port:     4999
```

This confirms: the external API returned 200 OK but took 842ms. It's not an error. It's a slow dependency. You now know where to file the bug.

---

## 10.6 Correlating Traces with Metrics

The most powerful observability technique is **exemplars**: embedding trace IDs directly into Prometheus histogram metrics.

When you record a metric observation, you can attach the current trace ID:

```csharp
// Recording a histogram observation with an exemplar (trace ID)
var traceId = Activity.Current?.TraceId.ToString();

_metrics.OrderCreationDuration.Record(
    elapsed.TotalSeconds,
    // The trace_id tag becomes an exemplar in Prometheus
    new KeyValuePair<string, object?>("trace_id", traceId)
);
```

In Grafana, when you view the P99 latency histogram:
1. You see a spike at 4,800ms
2. Grafana shows a dot on the histogram: "There's a trace here"
3. Click the dot → opens Jaeger with that exact trace
4. Immediately see what was slow

This closes the loop: **from metric alert → to root cause trace → in one click**.

---

## 10.7 Span Events: Point-in-Time Annotations

Events are timestamps within a span that mark important moments:

```csharp
// Adding events to an activity
using var activity = ObsTelemetry.StartOrderActivity("create", orderId.ToString());

activity?.AddEvent(new ActivityEvent("cache_miss",
    tags: new ActivityTagsCollection { { "cache_key", cacheKey } }
));

// ... DB query ...

activity?.AddEvent(new ActivityEvent("db_query_complete",
    tags: new ActivityTagsCollection {
        { "rows_returned", rowCount },
        { "query_ms",      sw.ElapsedMilliseconds }
    }
));
```

In Jaeger, events appear as vertical lines on the span timeline, showing exactly when things happened within a long operation.

---

## 10.8 Tracing vs Logging: When to Use Each

| Question | Best Tool |
|---|---|
| "Which span is slow?" | Trace |
| "What SQL query ran?" | Trace (db.statement attribute) |
| "Why did the span fail?" | Both: trace (exception) + log (full stack trace) |
| "How many errors in last 5min?" | Metrics (counter) |
| "What was the exact error message?" | Logs |
| "What did the user send in the request?" | Logs |
| "How does latency trend over 7 days?" | Metrics |
| "Find the trace for this specific order_id" | Logs (search by order_id) → then trace |

The three pillars are complementary. Traces tell you *where* the time went. Metrics tell you *whether* this is normal. Logs tell you *exactly what happened*.

---

## 10.9 Production Jaeger: Moving Beyond In-Memory

For production, replace the in-memory Jaeger backend:

```yaml
# Option 1: Jaeger with Elasticsearch backend
jaeger-collector:
  image: jaegertracing/jaeger-collector:1.58
  environment:
    SPAN_STORAGE_TYPE: elasticsearch
    ES_SERVER_URLS:    http://elasticsearch:9200
    ES_INDEX_PREFIX:   jaeger

jaeger-query:
  image: jaegertracing/jaeger-query:1.58
  environment:
    SPAN_STORAGE_TYPE: elasticsearch
    ES_SERVER_URLS:    http://elasticsearch:9200

# Option 2: Grafana Tempo (cheaper, object storage backend)
tempo:
  image: grafana/tempo:2.4.1
  command: ["-config.file=/etc/tempo.yaml"]
  volumes:
    - ./tempo/tempo.yaml:/etc/tempo.yaml
    - tempo_data:/tmp/tempo
  # Stores traces in S3, GCS, or local disk
  # Costs 90% less than Elasticsearch at scale
```

Grafana Tempo is the recommended production backend for teams already using Grafana — it integrates natively and costs far less than Elasticsearch for trace storage.
