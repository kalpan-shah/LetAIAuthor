# Chapter 11: Grafana — Building the RED Dashboard

> **Part V — Grafana Dashboards**

---

## 11.1 Grafana Fundamentals

Grafana is a visualization layer — it does not store data. It queries Prometheus (and Jaeger, Loki, etc.) and renders the results. Every panel is a query plus a visualization type.

**Key concepts:**
- **Dashboard:** A collection of panels arranged in rows
- **Panel:** One visualization (graph, stat, table, gauge, etc.)
- **Data source:** Where Grafana fetches data (Prometheus, Jaeger, Loki)
- **Variable:** A dropdown that filters all panels (e.g., `$route`, `$instance`)
- **Time range:** Controls the `[5m]` window in PromQL queries

---

## 11.2 Dashboard Variables

Variables make dashboards interactive. Define them in Dashboard Settings → Variables:

```
Variable: route
Type: Query
Data source: Prometheus
Query: label_values(http_server_request_duration_seconds_count, route)
Multi-value: true
Include All option: true
Default: All
```

```
Variable: instance
Type: Query
Data source: Prometheus
Query: label_values(http_server_request_duration_seconds_count, instance)
Multi-value: true
```

Now every panel can use `$route` and `$instance` in its query:
```promql
histogram_quantile(0.95,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket{
      route=~"$route",
      instance=~"$instance"
    }[5m])
  )
)
```

---

## 11.3 The RED Dashboard: Panel by Panel

### Panel 1: Request Rate (Time Series)

**Title:** Requests per Second
**Type:** Time Series
**Query:**
```promql
sum by (route) (
  rate(http_server_request_duration_seconds_count{route=~"$route"}[5m])
)
```
**Visualization settings:**
- Unit: `reqps` (requests/second)
- Legend: `{{route}}`
- Thresholds: None (traffic is informational)
- Fill opacity: 10

**What to look for:** Sudden drops (upstream broke), unexpected spikes (load test or attack), steady growth (organic traffic).

---

### Panel 2: Error Rate (Stat + Time Series)

**Title:** Error Rate (5xx)
**Type:** Stat (big number) + Time Series below

**Stat query:**
```promql
sum(rate(http_server_request_duration_seconds_count{
  status_code=~"5..",
  route=~"$route"
}[5m]))
/
sum(rate(http_server_request_duration_seconds_count{route=~"$route"}[5m]))
```
**Unit:** Percent (0.0–1.0)
**Thresholds:**
- Green: 0–0.5%
- Yellow: 0.5–1%
- Red: > 1%
**Color mode:** Background (entire panel turns red on breach)

**Time Series query (for trend):**
```promql
sum by (route, status_code) (
  rate(http_server_request_duration_seconds_count{
    status_code=~"[45]..",
    route=~"$route"
  }[5m])
)
/ sum by (route) (
  rate(http_server_request_duration_seconds_count{route=~"$route"}[5m])
)
```

---

### Panel 3: Latency Heatmap (Heatmap)

**Title:** Request Duration Distribution
**Type:** Heatmap
**Query:**
```promql
sum by (le) (
  rate(http_server_request_duration_seconds_bucket{
    route=~"$route"
  }[5m])
)
```
**Format:** Heatmap (Grafana handles the bucket grouping)
**Color scheme:** From green (short) to red (long)

The heatmap is the best visualization for latency distribution over time. It shows:
- The "bulk" of requests (the bright band) = where most latency lives
- Outliers (occasional dots in the red zone)
- Bimodal distributions (two bright bands = two distinct populations)

---

### Panel 4: Latency Percentiles (Time Series)

**Title:** Request Latency (P50 / P95 / P99)
**Type:** Time Series
**Queries:**

```promql
# P50 (median)
histogram_quantile(0.50,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket{route=~"$route"}[5m])
  )
)
```
```promql
# P95
histogram_quantile(0.95,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket{route=~"$route"}[5m])
  )
)
```
```promql
# P99
histogram_quantile(0.99,
  sum by (le) (
    rate(http_server_request_duration_seconds_bucket{route=~"$route"}[5m])
  )
)
```

**Legend labels:** P50, P95, P99
**Unit:** Seconds → Grafana auto-formats as "847 ms"
**Thresholds:**
- Yellow: 0.5s (SLO boundary)
- Red: 1.0s

**Add reference line:** Constant at 0.5 (your SLO threshold)

---

### Panel 5: Latency by Route (Table)

**Title:** Latency by Route (last 5 min)
**Type:** Table
**Query:**
```promql
# P50 per route
histogram_quantile(0.50,
  sum by (route, le) (
    rate(http_server_request_duration_seconds_bucket[5m])
  )
)
```
Use Grafana "Join" transformation to show P50, P95, P99 side-by-side:

| Route | P50 | P95 | P99 | Error Rate |
|---|---|---|---|---|
| GET /api/orders | 12ms | 45ms | 124ms | 0.0% |
| POST /api/orders | 234ms | 892ms | 4,812ms | 0.3% |
| GET /api/reports/revenue | 1,240ms | 4,200ms | 8,910ms | 0.0% |

**Apply thresholds by column:**
- P95: Green < 200ms, Yellow < 500ms, Red > 500ms
- Error Rate: Green < 0.5%, Yellow < 1%, Red > 1%

---

### Panel 6: Apdex Score (Gauge)

**Title:** Apdex Score
**Type:** Gauge

Apdex (Application Performance Index) is a standardised satisfaction score:
- **Satisfied:** Response < T (target, 500ms)
- **Tolerating:** T < Response < 4T (500ms–2000ms)
- **Frustrated:** Response > 4T (> 2000ms)

Score = (Satisfied + 0.5 × Tolerating) / Total
- 1.0 = perfect
- 0.5 = half of users frustrated
- < 0.5 = serious degradation

```promql
(
  sum(rate(http_server_request_duration_seconds_bucket{le="0.5"}[5m]))
  +
  0.5 * (
    sum(rate(http_server_request_duration_seconds_bucket{le="2.0"}[5m]))
    - sum(rate(http_server_request_duration_seconds_bucket{le="0.5"}[5m]))
  )
)
/
sum(rate(http_server_request_duration_seconds_count[5m]))
```

**Thresholds:**
- Red: < 0.7 (unacceptable)
- Yellow: 0.7–0.94 (good)
- Green: 0.94–1.0 (excellent)

---

## 11.4 Dashboard JSON for Automated Provisioning

Grafana dashboards can be stored as JSON files and auto-provisioned on startup. Here's the structure:

```json
{
  "uid":   "red-dashboard",
  "title": "API — RED Dashboard",
  "tags":  ["api", "red", "performance"],
  "time":  { "from": "now-1h", "to": "now" },
  "refresh": "30s",
  "templating": {
    "list": [
      {
        "name": "route",
        "type": "query",
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "definition": "label_values(http_server_request_duration_seconds_count, route)",
        "includeAll": true,
        "multi": true,
        "current": { "value": "$__all" }
      }
    ]
  },
  "panels": [
    {
      "id": 1,
      "type": "timeseries",
      "title": "Request Rate (RPS)",
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
      "targets": [{
        "datasource": { "type": "prometheus" },
        "expr": "sum by (route) (rate(http_server_request_duration_seconds_count{route=~\"$route\"}[5m]))",
        "legendFormat": "{{route}}"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "custom": { "fillOpacity": 10 }
        }
      }
    }
  ]
}
```

Save this as `deploy/grafana/dashboards/red-dashboard.json` and Grafana loads it on startup via the provisioning configuration.

---

## 11.5 How to Read the RED Dashboard

When you get a latency alert, open the RED dashboard and work through it in order:

1. **Rate panel:** Is traffic normal? A spike in traffic could explain latency rise.
2. **Error rate panel:** Are requests failing? If yes, what status codes?
3. **Latency percentile panel:** Which percentiles are elevated? P50 elevated = widespread issue. Only P99 elevated = tail latency / outlier issue.
4. **Route table:** Which specific routes are slow? This tells you where to look.
5. **Heatmap:** Is the distribution bimodal? (Two populations = one code path is fast, one is slow)

If the RED dashboard points to a specific route being slow → open Jaeger, filter by that route, min duration = P95 value → find the trace that shows where the time goes.
