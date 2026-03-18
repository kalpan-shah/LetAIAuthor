# Chapter 15: Deep Dive — Availability Engineering

> **Part VI — Deep Dives**

---

## 15.1 What "Availability" Actually Means

"Five nines" (99.999%) availability is a common aspiration. Understanding what it means in calendar time:

| Availability | Downtime per Year | Downtime per Month | Downtime per Day |
|---|---|---|---|
| 99% (two nines) | 3.65 days | 7.3 hours | 14.4 minutes |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes | 1.44 minutes |
| 99.95% | 4.38 hours | 21.9 minutes | 43.2 seconds |
| 99.99% (four nines) | 52.6 minutes | 4.4 minutes | 8.6 seconds |
| 99.999% (five nines) | 5.26 minutes | 25.9 seconds | 0.86 seconds |

**The critical insight:** 99.999% means your total allowed downtime per year is just over 5 minutes. That's less time than a typical incident investigation takes. Achieving five nines requires fundamentally different architecture, not just better monitoring.

### Availability is a Request-Level Measurement

Modern availability is not measured by "is the server running?". It's measured by:

```
Availability = (Successful Requests) / (Total Requests)
```

A server can be "running" while returning 100% errors. That's 0% availability. A server can be restarting while all in-flight requests succeed. That's 100% availability for those requests.

---

## 15.2 Measuring Availability in Prometheus

```promql
# Instantaneous availability (last 5 minutes)
1 - (
  sum(rate(http_server_request_duration_seconds_count{status_code=~"5.."}[5m]))
  /
  sum(rate(http_server_request_duration_seconds_count[5m]))
)

# 30-day availability (true SLO window)
sum(increase(http_server_request_duration_seconds_count{status_code!~"5.."}[30d]))
/ sum(increase(http_server_request_duration_seconds_count[30d]))

# Error budget consumed this month (in minutes of equivalent downtime)
(1 - sum(increase(http_server_request_duration_seconds_count{status_code!~"5.."}[30d]))
     / sum(increase(http_server_request_duration_seconds_count[30d])))
/ (1 - 0.999)   # Normalize to budget fraction
* 43.8           # Minutes in the 30-day budget for 99.9% SLO
```

---

## 15.3 The Error Budget in Practice

### Budget Policy Enforcement

Define explicit rules for what happens as the error budget drains:

| Budget Remaining | Policy |
|---|---|
| > 50% | Normal operations. Deploy freely. |
| 25–50% | Extra scrutiny on risky changes. Post-deploy monitoring required (30 min). |
| 10–25% | No risky deploys. All changes require SRE review. |
| 0–10% | Feature freeze. Only reliability improvements allowed. |
| 0% (exhausted) | Full feature freeze until next 30-day window. Incident review required. |

This policy makes reliability a shared responsibility between product and engineering — both teams have skin in the game.

### Grafana Error Budget Policy Panel

```promql
# Policy level (0=Normal, 1=Caution, 2=Restricted, 3=Frozen)
(
  # remaining budget fraction
  (
    sum(increase(http_server_request_duration_seconds_count{status_code!~"5.."}[30d]))
    / sum(increase(http_server_request_duration_seconds_count[30d]))
    - 0.999
  ) / (1 - 0.999)
)
# Map to policy level using thresholds in Grafana stat panel:
# > 0.5 → 0 (Normal)
# > 0.25 → 1 (Caution)
# > 0.10 → 2 (Restricted)
# > 0 → 3 (Frozen)
```

---

## 15.4 Availability Anti-Patterns

### Anti-Pattern 1: Synchronous Dependency on Non-Critical Services

```csharp
// BAD: Order creation fails if the notification service is down
app.MapPost("/api/orders", async (CreateOrderRequest req, ...) => {
    var order = await orderService.CreateAsync(req);
    await notificationService.SendEmailAsync(order);  // If this fails, order fails!
    return Results.Created($"/api/orders/{order.Id}", order);
});

// GOOD: Notify asynchronously — order succeeds even if notification fails
app.MapPost("/api/orders", async (CreateOrderRequest req, ...) => {
    var order = await orderService.CreateAsync(req);
    // Fire and forget — or better, publish to NATS and let worker retry
    _ = notificationService.SendEmailAsync(order)
          .ContinueWith(t => logger.LogError(t.Exception, "Notification failed"),
                        TaskContinuationOptions.OnlyOnFaulted);
    return Results.Created($"/api/orders/{order.Id}", order);
});
```

**Observability signature:** `order.creation.errors_total` spikes when `external_api.errors_total` spikes, even though the external service is for notifications (non-critical).

---

### Anti-Pattern 2: No Circuit Breaker on External Dependencies

Without a circuit breaker, if the external API takes 10 seconds to time out:
- Each request occupies a thread for 10 seconds
- Thread pool saturates in seconds
- All requests (even those not hitting the external API) begin queuing
- System appears completely down

```csharp
// Add Microsoft.Extensions.Http.Resilience NuGet package

builder.Services.AddHttpClient<ExternalApiClient>()
    .AddStandardResilienceHandler(opts => {
        // Circuit breaker: open after 50% failure rate, stay open for 30s
        opts.CircuitBreaker = new HttpCircuitBreakerStrategyOptions {
            SamplingDuration       = TimeSpan.FromSeconds(30),
            MinimumThroughput      = 10,
            FailureRatio           = 0.5,
            BreakDuration          = TimeSpan.FromSeconds(30),
        };
        // Timeout: fail fast instead of waiting 10 seconds
        opts.TotalRequestTimeout = new HttpTimeoutStrategyOptions {
            Timeout = TimeSpan.FromSeconds(3),
        };
        // Retry with backoff (only for transient errors)
        opts.Retry = new HttpRetryStrategyOptions {
            MaxRetryAttempts = 2,
            BackoffType      = DelayBackoffType.Exponential,
            UseJitter        = true,
        };
    });
```

**Observability signature after adding circuit breaker:**
- `external_api.timeouts_total` still spikes when the API is slow
- But API P95 latency drops from 10,000ms to 3,100ms (timeout instead of wait)
- Error rate increases slightly (circuit breaker failures) but recovers in 30s
- Thread pool queue depth stays near 0 (fast failures free threads quickly)

---

## 15.5 Health Check Design

A well-designed health check accurately represents whether the service can serve traffic:

```csharp
// GET /readyz — Kubernetes readiness probe
app.MapGet("/readyz", async (
    NpgsqlDataSource db,
    IConnectionMultiplexer redis,
    HttpClient httpClient,
    CancellationToken ct) =>
{
    var checks = new Dictionary<string, object>();
    var healthy = true;

    // Check 1: Database connectivity
    try {
        await using var conn = await db.OpenConnectionAsync(ct);
        var result = await conn.ExecuteScalarAsync<int>("SELECT 1");
        checks["database"] = new { status = "ok", latency_ms = stopwatch.ElapsedMs };
    } catch (Exception ex) {
        checks["database"] = new { status = "error", error = ex.Message };
        healthy = false;
    }

    // Check 2: Redis connectivity
    try {
        var db2 = redis.GetDatabase();
        await db2.PingAsync();
        checks["redis"] = new { status = "ok" };
    } catch (Exception ex) {
        checks["redis"] = new { status = "error", error = ex.Message };
        // Redis degraded but not fatal (we can serve without cache)
        // Don't set healthy = false for non-critical dependencies
    }

    // Check 3: Can we reach the external API? (optional, non-fatal)
    // Don't include this — if external API is down, we still want to
    // accept traffic and use the circuit breaker

    return healthy
        ? Results.Ok(new { status = "ready", checks })
        : Results.ServiceUnavailable(new { status = "not ready", checks });
});

// GET /healthz — Kubernetes liveness probe (simpler: are we running?)
app.MapGet("/healthz", () => Results.Ok(new { status = "alive" }));
```

**Key decisions:**
- Only include **critical** dependencies in readiness (things without which you truly cannot serve)
- Non-critical dependencies (cache, notifications) should not fail readiness
- Liveness should be trivially simple — if it fails, Kubernetes will restart the pod

---

# Chapter 16: Production Patterns — Alerting & Incident Response

> **Part VII — Production Patterns**

---

## 16.1 The Anatomy of a Good Alert

A good alert has exactly these properties:

1. **Actionable:** The person paged knows what to do
2. **Accurate:** It fires when something is wrong, not when things are normal
3. **Urgent:** Severity matches the actual impact
4. **Linked:** Includes a runbook URL and dashboard URL

A bad alert has:
- No action the on-call can take
- Fires for transient issues that self-resolve
- Pages at 3 AM for non-user-facing issues
- No context about what to look at

---

## 16.2 Alert Classification

| Class | Definition | Response Time | Who Gets Paged |
|---|---|---|---|
| P1 Critical | User-facing service is broken or severely degraded | Immediate (< 5 min) | On-call engineer + lead |
| P2 Warning | Degradation predicted, trending toward P1 | Within 30 minutes | On-call engineer |
| P3 Info | Potential issue, no immediate user impact | Next business day | Slack notification only |
| Noise | Self-resolves, no action needed | Never | Nobody (investigate and remove) |

---

## 16.3 Alert Design Patterns

### Pattern 1: Symptom-Based Alerting

Alert on what the user experiences, not on internal implementation details.

```yaml
# GOOD: The symptom (user-visible)
- alert: HighAPILatency
  expr: |
    histogram_quantile(0.95, ...) > 0.5
  annotations:
    summary: "Users experiencing slow responses"

# BAD: The cause (internal)
- alert: HighCPU
  expr: process_cpu_time > 0.8
  annotations:
    summary: "CPU is high"  # So what? Are users affected?
```

CPU being high is not itself a problem. It's a problem if it's causing latency or errors. Alert on the effect.

### Pattern 2: Avoid Alert Fatigue

Alert fatigue is the state where engineers stop caring about alerts because too many fire for non-issues. Signs:
- Alerts that fire and self-resolve regularly
- The same alert fires multiple times per week but nothing changes
- On-call engineers acknowledge alerts without investigating

**Remediation:**
- Increase `for:` duration (5m → 15m) to require sustained conditions
- Raise thresholds to actual SLO boundaries
- Add inhibition rules: suppress warnings when a critical alert is active

### Pattern 3: Multi-Window Alerting for SLOs

(Covered in Chapter 9, but worth reinforcing)

Single-window alerts have a fundamental trade-off: short windows are noisy (false positives from transient spikes), long windows are slow (you miss fast-moving incidents).

The solution: require elevated burn rate in **both** a short window and a medium window:

```yaml
# Fires only if burn rate is high in BOTH windows (reduces false positives)
- alert: SLOBudgetBurnCritical
  expr: |
    slo:error_budget_burn_rate:ratio_rate1h  > 14.4
    AND
    slo:error_budget_burn_rate:ratio_rate6h  > 6
  for: 2m   # Still require a short sustained period
```

---

## 16.4 The Incident Response Workflow

When the alert fires:

### Step 1: Triage (< 5 minutes)

Open the RED dashboard. Answer three questions:
1. **What is the impact?** (error rate, latency — how bad?)
2. **What is the scope?** (all routes or one route? all instances or one?)
3. **When did it start?** (look at the time the line starts climbing)

### Step 2: Investigate the Cause (5–20 minutes)

Based on the RED dashboard:
- **Error rate elevated:** Check logs for error patterns. Open Jaeger and filter for error traces.
- **Latency elevated:** Open Jaeger, filter for slow traces, look for the long span.
- **Traffic dropped to zero:** Check Kubernetes pod status, deployment events.

```bash
# Quick triage commands
kubectl get pods -n obs-demo -o wide          # Are pods running?
kubectl logs -l app=api -n obs-demo --tail=50 | jq 'select(.level=="Error")'
kubectl top pods -n obs-demo                  # CPU/memory?
kubectl describe deployment api -n obs-demo   # Recent events?
```

### Step 3: Mitigate (immediate action)

Mitigation stops the bleeding before root cause is found:
- **Rollback:** `kubectl rollout undo deployment/api`
- **Scale out:** `kubectl scale deployment/api --replicas=10`
- **Feature flag:** Disable the offending feature via config
- **Circuit breaker:** If external API is causing cascading failure, configure circuit breaker

### Step 4: Resolve (root cause fix)

Only after mitigation and communication. Don't rush to root cause while users are impacted.

### Step 5: Post-Mortem (within 5 business days)

A blameless post-mortem answers:
- Timeline: what happened, in what order, with exact timestamps
- Detection: how did we find out? (alert, customer complaint, monitoring?)
- Impact: how many users affected? for how long?
- Root cause: what was the actual technical cause?
- Contributing factors: what made this worse?
- Action items: what changes prevent recurrence?

---

## 16.5 The Runbook Template

Each alert should link to a runbook. Here's the template:

```markdown
# Runbook: HighAPILatency

## Alert Description
API P95 latency has exceeded 500ms (SLO boundary) for more than 5 minutes.

## Impact
Users are experiencing slow responses. Sustained > 5 minutes will breach the monthly SLO.

## Immediate Steps

1. **Open the RED Dashboard**
   https://grafana.internal/d/red/api-red?var-route=All

2. **Identify scope**
   - Is it all routes or one specific route?
   - Is the Latency by Route table showing one bad route?

3. **If one route is slow — check traces**
   - Open Jaeger: https://jaeger.internal
   - Filter: service=obs-api, operation=GET /api/[slow route], min duration=500ms
   - Look for the longest span in the trace waterfall
   - Common culprits: DB query, external API call, cache miss

4. **If all routes are slow — check resources**
   - Open USE Dashboard: https://grafana.internal/d/use/
   - Check: DB connection pool saturation
   - Check: Thread pool queue length
   - Check: GC pause ratio

5. **If DB is the cause**
   - Run: `psql $DB_URL -c "SELECT pid, state, query_start, query FROM pg_stat_activity WHERE state='active' ORDER BY query_start;"`
   - Look for long-running queries or lock waits

## Escalation
If not resolved within 15 minutes: page the database team.
If not resolved within 30 minutes: escalate to engineering lead.

## Related Alerts
- DatabaseConnectionPoolSaturation
- DatabaseLockWaits
- ErrorBudgetBurnRateCritical
```

---

## 16.6 The Final Architecture: Putting It All Together

```
Production Traffic
    │
    ▼
.NET Minimal API
    │ emits metrics via /metrics      → Prometheus (stores, alerts)
    │ emits traces via OTLP           → Jaeger (displays traces)
    │ emits logs via stdout (JSON)    → Loki (via Fluent Bit)
    │ serves requests
    ▼
PostgreSQL
    ▲
postgres_exporter                     → Prometheus (pg_stat_* metrics)

Prometheus                            → Alertmanager → PagerDuty/Slack

Grafana
    ← Prometheus (metrics)
    ← Jaeger     (traces, linked from metric exemplars)
    ← Loki       (logs, linked from trace_id in traces)
```

The three pillars are linked:
- **Metric** spike → click Jaeger exemplar → open **trace** → see db.statement → search **logs** by trace_id → see full error context

This is complete observability: you can follow a signal from the high-level metric down to the individual log line without switching tools or writing queries.
