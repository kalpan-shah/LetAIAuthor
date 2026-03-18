# Chapter 9: Recording Rules, Alerting & Prometheus Best Practices

> **Part III — Metrics with Prometheus**

---

## 9.1 Recording Rules: Pre-Computing Expensive Queries

Some PromQL queries are expensive to compute on every dashboard load — especially long time-range queries or complex joins. Recording rules pre-compute these expressions and store the result as a new metric. Grafana then queries the pre-computed metric instead.

```yaml
# deploy/prometheus/rules/recording_rules.yml
groups:

  - name: obs-api-sli
    interval: 1m   # Compute every 1 minute
    rules:

    # ── Availability SLI ────────────────────────────────────────────────────
    # Fraction of requests succeeding (non-5xx) over 5-minute windows
    - record: sli:request_success_rate:ratio_rate5m
      expr: |
        1 - (
          sum(rate(http_server_request_duration_seconds_count{status_code=~"5.."}[5m]))
          /
          sum(rate(http_server_request_duration_seconds_count[5m]))
        )

    # Over 1-hour window (for slower-burning SLO calculations)
    - record: sli:request_success_rate:ratio_rate1h
      expr: |
        1 - (
          sum(rate(http_server_request_duration_seconds_count{status_code=~"5.."}[1h]))
          /
          sum(rate(http_server_request_duration_seconds_count[1h]))
        )

    # Over 6-hour window
    - record: sli:request_success_rate:ratio_rate6h
      expr: |
        1 - (
          sum(rate(http_server_request_duration_seconds_count{status_code=~"5.."}[6h]))
          /
          sum(rate(http_server_request_duration_seconds_count[6h]))
        )

    # ── Latency SLI ─────────────────────────────────────────────────────────
    # Fraction of requests completing under 500ms (SLO threshold)
    - record: sli:request_latency_500ms:ratio_rate5m
      expr: |
        sum(rate(http_server_request_duration_seconds_bucket{le="0.5"}[5m]))
        /
        sum(rate(http_server_request_duration_seconds_count[5m]))

    # ── Error Budget Burn Rate ───────────────────────────────────────────────
    # How fast are we burning through the error budget?
    # Burn rate of 1.0 = consuming budget exactly on pace
    # Burn rate of 14.4 = will exhaust 30-day budget in 2 days
    - record: slo:error_budget_burn_rate:ratio_rate1h
      expr: |
        (1 - sli:request_success_rate:ratio_rate1h) / (1 - 0.999)

    - record: slo:error_budget_burn_rate:ratio_rate6h
      expr: |
        (1 - sli:request_success_rate:ratio_rate6h) / (1 - 0.999)

    # ── Request Rate ────────────────────────────────────────────────────────
    - record: job:http_requests:rate5m
      expr: sum by (route, method) (
              rate(http_server_request_duration_seconds_count[5m])
            )

    # ── Latency Percentiles (pre-computed for dashboard performance) ────────
    - record: job:http_request_duration_p50:rate5m
      expr: |
        histogram_quantile(0.50,
          sum by (route, le) (
            rate(http_server_request_duration_seconds_bucket[5m])
          )
        )

    - record: job:http_request_duration_p95:rate5m
      expr: |
        histogram_quantile(0.95,
          sum by (route, le) (
            rate(http_server_request_duration_seconds_bucket[5m])
          )
        )

    - record: job:http_request_duration_p99:rate5m
      expr: |
        histogram_quantile(0.99,
          sum by (route, le) (
            rate(http_server_request_duration_seconds_bucket[5m])
          )
        )

  - name: obs-db-derived
    interval: 30s
    rules:

    # Buffer cache hit ratio — saves repeated computation in dashboards
    - record: pg:buffer_cache_hit_ratio:rate5m
      expr: |
        rate(pg_stat_database_blks_hit_total{datname="obsdb"}[5m])
        /
        (
          rate(pg_stat_database_blks_hit_total{datname="obsdb"}[5m])
          + rate(pg_stat_database_blks_read_total{datname="obsdb"}[5m])
        )

    - record: pg:connection_saturation:current
      expr: |
        pg_stat_database_numbackends{datname="obsdb"} / 50
```

---

## 9.2 Alerting Rules

Alerts evaluate PromQL expressions. When an expression returns a non-empty vector, the alert fires.

```yaml
# deploy/prometheus/rules/alerts.yml
groups:

  - name: obs-api-availability
    rules:

    # ── Critical: SLO burn rate too high ────────────────────────────────────
    # This is the "multi-window, multi-burn-rate" alert pattern from the SRE book.
    # Fast burn (14.4x over 1h): will exhaust 30d budget in 2 days
    - alert: ErrorBudgetBurnRateCritical
      expr: |
        slo:error_budget_burn_rate:ratio_rate1h > 14.4
        AND slo:error_budget_burn_rate:ratio_rate6h > 6
      for: 2m
      labels:
        severity: critical
        team:     platform
      annotations:
        summary:     "Error budget burning too fast — P1 incident"
        description: |
          Error budget burn rate is {{ $value | humanize }}x.
          At this rate, the 30-day error budget will be exhausted in
          {{ printf "%.1f" (720 / $value) }} hours.
        runbook: "https://runbooks.internal/api/error-budget-burn"
        dashboard: "https://grafana.internal/d/slo/api-slo"

    # Slow burn (3x over 6h): will exhaust budget in < 10 days
    - alert: ErrorBudgetBurnRateHigh
      expr: |
        slo:error_budget_burn_rate:ratio_rate6h > 3
      for: 15m
      labels:
        severity: warning
      annotations:
        summary:     "Error budget draining faster than expected"
        description: "Burn rate {{ $value | humanize }}x over last 6h"

  - name: obs-api-latency
    rules:

    # P95 latency above SLO threshold
    - alert: HighAPILatency
      expr: |
        histogram_quantile(0.95,
          sum by (route, le) (
            rate(http_server_request_duration_seconds_bucket[5m])
          )
        ) > 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary:     "API P95 latency > 500ms on route {{ $labels.route }}"
        description: "P95 = {{ $value | humanizeDuration }}"

    # P99 latency critical
    - alert: CriticalAPILatency
      expr: |
        histogram_quantile(0.99,
          sum by (le) (
            rate(http_server_request_duration_seconds_bucket[5m])
          )
        ) > 2.0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "API P99 latency > 2 seconds"

  - name: obs-api-traffic
    rules:

    # Traffic drops to zero (app is not receiving requests)
    - alert: NoIncomingTraffic
      expr: |
        sum(rate(http_server_request_duration_seconds_count[5m])) == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Zero incoming traffic — is the app running?"

    # Traffic spike (> 10x normal, may indicate load test or attack)
    - alert: UnexpectedTrafficSpike
      expr: |
        sum(rate(http_server_request_duration_seconds_count[5m]))
        >
        10 * sum(rate(http_server_request_duration_seconds_count[1h] offset 1d))
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Traffic spike: {{ $value | humanize }} rps"

  - name: obs-database
    rules:

    # Connection pool saturation
    - alert: DatabaseConnectionPoolSaturation
      expr: pg:connection_saturation:current > 0.85
      for: 3m
      labels:
        severity: warning
      annotations:
        summary:     "PostgreSQL connection pool > 85% full"
        description: "{{ $value | humanizePercentage }} of connections in use"
        runbook: "https://runbooks.internal/db/connection-pool"

    - alert: DatabaseConnectionPoolExhausted
      expr: pg:connection_saturation:current > 0.95
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "PostgreSQL connection pool almost full — new connections will fail"

    # Buffer cache degradation
    - alert: DatabaseCacheHitRateLow
      expr: pg:buffer_cache_hit_ratio:rate5m < 0.90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary:     "PostgreSQL buffer cache hit rate {{ $value | humanizePercentage }}"
        description: |
          Below 90% means many queries are reading from disk.
          Consider increasing shared_buffers or investigate recent queries.

    # Deadlocks
    - alert: DatabaseDeadlocks
      expr: increase(pg_stat_database_deadlocks_total{datname="obsdb"}[5m]) > 0
      labels:
        severity: warning
      annotations:
        summary: "PostgreSQL deadlocks detected: {{ $value }} in last 5m"

    # Lock waits
    - alert: DatabaseLockWaits
      expr: pg_lock_waits_waiting > 5
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "{{ $value }} queries waiting on database locks"

  - name: obs-runtime
    rules:

    # .NET Gen2 GC pressure
    - alert: HighGCPressure
      expr: |
        rate(dotnet_gc_collections_total{generation="gen2"}[5m]) > 0.1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Frequent Gen2 GC — potential memory pressure or leak"

    # Thread pool saturation
    - alert: ThreadPoolSaturation
      expr: dotnet_thread_pool_queue_length > 100
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Thread pool queue length {{ $value }} — CPU-bound saturation"

    # Memory usage
    - alert: HighMemoryUsage
      expr: |
        process_working_set_bytes / (1024 * 1024 * 1024) > 1.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "API memory usage {{ $value | humanize }}GB"
```

---

## 9.3 Multi-Window, Multi-Burn-Rate Alerting

The standard approach for SLO-based alerting uses multiple time windows to balance false positives (too noisy) against false negatives (too slow to detect):

| Burn Rate | Detection Window | Alert Severity | SLO Budget Exhaustion |
|---|---|---|---|
| 14.4x | 1h | Critical (P1) | In ~2 days |
| 6x | 6h | Critical (P1) | In ~5 days |
| 3x | 1d | Warning (P2) | In ~10 days |
| 1x | 3d | Info | In ~30 days (pace) |

The rule fires only if **both** windows show elevated burn rate, reducing false positives from transient spikes.

```promql
# Critical: Fast burn in short window AND sustained in medium window
slo:error_budget_burn_rate:ratio_rate1h  > 14.4
AND
slo:error_budget_burn_rate:ratio_rate6h  > 6
```

---

## 9.4 Alert Routing with Alertmanager

```yaml
# alertmanager.yml (add this service to docker-compose.yml)
route:
  receiver: default
  group_by: [alertname, severity]
  group_wait:      30s    # Collect alerts for 30s before sending first notification
  group_interval:  5m     # Wait 5m before sending updates on existing group
  repeat_interval: 4h     # Resend unresolves every 4h

  routes:
    - match: { severity: critical }
      receiver: pagerduty
      group_wait: 10s    # Faster for critical
      repeat_interval: 1h

    - match: { severity: warning }
      receiver: slack

receivers:
  - name: default
    slack_configs:
      - api_url: $SLACK_WEBHOOK_URL
        channel: '#observability-alerts'
        title:   '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        text:    '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: pagerduty
    pagerduty_configs:
      - service_key: $PAGERDUTY_SERVICE_KEY
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

  - name: slack
    slack_configs:
      - api_url: $SLACK_WEBHOOK_URL
        channel: '#platform-warnings'

inhibit_rules:
  # If a critical alert fires, suppress warnings about the same alertname
  - source_match: { severity: critical }
    target_match: { severity: warning }
    equal:        [alertname]
```
