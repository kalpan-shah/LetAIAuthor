# Chapter 2: The Four Golden Signals

> **Part I — Foundations**

Google's Site Reliability Engineering book introduced the concept of the **four golden signals**: the minimum set of metrics you need to understand the health of any service. They are: **Latency**, **Traffic**, **Errors**, and **Saturation**.

These four signals are not arbitrary. They map directly to the experience of the users of your system. If you can only instrument four things, instrument these.

---

## 2.1 Signal 1: Latency

**Definition:** The time it takes to service a request.

Latency is not one number — it's a distribution. A system that responds to 99% of requests in 50ms but takes 30 seconds for the other 1% is not a "50ms system". That 1% is a person waiting 30 seconds. If you have 1,000 users, that's 10 people every single minute with a terrible experience.

### Why Averages Are Dangerous

```
Request durations (milliseconds):
10, 12, 11, 13, 10, 9, 12, 8, 11, 3000

Average: (10+12+11+13+10+9+12+8+11+3000) / 10 = 309 ms
P50:     11 ms
P95:     (would need more data, but trending toward 3000)
P99:     3000 ms
```

The average (309ms) is worse than 90% of requests but far better than the outlier. **It describes no user's actual experience.** The percentiles tell the truth:
- P50 = 11ms: the median user waits 11ms
- P99 = 3000ms: the worst 1% wait 3 seconds

### What to Track

| Percentile | What it tells you |
|---|---|
| P50 (median) | The typical user experience |
| P90 | 9 out of 10 users experience this or better |
| P95 | The near-worst-case; good for SLO definitions |
| P99 | The tail; catches outliers and worst-case paths |
| P99.9 (P999) | Only relevant at very high request volumes |

> **Rule of thumb:** Define your SLO on P95 or P99. Monitor all percentiles on your dashboard. Page on P95. Never alert on average.

### Latency Includes Success AND Failure

A critical but often missed point: **measure the latency of failed requests separately from successful ones**. A request that fails immediately in 1ms looks great in your average but represents a broken experience. A slow failure (5 seconds before returning an error) is the worst of both worlds.

```
Good breakdown:
  http_request_duration_seconds{status="2xx", route="/orders"} histogram
  http_request_duration_seconds{status="5xx", route="/orders"} histogram
```

---

## 2.2 Signal 2: Traffic

**Definition:** How much demand is being placed on your system.

Traffic is your throughput measurement. For an HTTP API, it's requests per second (RPS). For a database, it's queries per second. For a message queue, it's messages consumed per second.

Traffic is important for:
- **Capacity planning:** How much more can we handle?
- **Anomaly detection:** A sudden traffic drop often means something upstream broke.
- **Correlation:** Did latency spike because traffic spiked?

### Traffic Anomaly Patterns

```
Normal:    ████████████████ 500 rps
Spike:     ████████████████████████████ 2000 rps → latency rises, saturation increases
Drop:      ████   20 rps → upstream broke, loadbalancer misconfigured
Zero:      _______ 0 rps → application is not receiving traffic at all
```

### How to Measure

In Prometheus, traffic is derived from a counter:

```promql
# Requests per second over the last 5 minutes
rate(http_requests_total[5m])

# Broken down by route and status
sum by (route, status) (rate(http_requests_total[5m]))
```

---

## 2.3 Signal 3: Errors

**Definition:** The rate of requests that fail.

"Fail" means different things in different contexts:
- HTTP 5xx responses (explicit server errors)
- HTTP 4xx that represent service failures (not user mistakes like 404)
- HTTP 200 responses with an error payload (your API returns `{"success": false}`)
- Requests that time out before completing
- Requests that succeed but return corrupt/empty data

### Error Rate vs Error Count

**Error count** tells you volume. **Error rate** (errors / total requests) tells you proportion. Both matter:

- 100 errors/min at 100 req/min = 100% error rate. Everything is broken.
- 100 errors/min at 10,000 req/min = 1% error rate. Concerning but may be within SLO.
- 5 errors/min at 5 req/min = 100% error rate. Your 2 AM batch job is completely broken.

### What Not to Count as Errors

User errors (HTTP 404, 400, 401, 403) are typically not errors in the service-health sense — they represent valid requests that the user got wrong. Monitor them separately for security purposes (spike in 401s = brute force attempt) but don't include them in your service error rate.

```promql
# Error rate: 5xx only
rate(http_requests_total{status=~"5.."}[5m])
/ rate(http_requests_total[5m])

# Success rate (for SLO burn rate)
1 - (
  rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m])
)
```

---

## 2.4 Signal 4: Saturation

**Definition:** How "full" your service is — what fraction of a resource's capacity is in use.

Saturation is the signal that tells you *before* the system breaks. Latency and errors are **reactive** — they tell you something has gone wrong. Saturation is **predictive** — it tells you something is about to go wrong.

### Common Saturation Points in a .NET + PostgreSQL System

| Resource | Saturation Metric | Critical Threshold |
|---|---|---|
| .NET thread pool | queued work items, worker threads used | > 80% of max threads |
| HTTP connection pool | active connections / max connections | > 80% |
| PostgreSQL connections | active / max_connections | > 85% |
| PostgreSQL lock waits | pg_locks waiting | > 0 is notable, > 10 is critical |
| Memory | Working set bytes / available | > 85% |
| CPU | CPU % across all cores | > 80% sustained |
| Disk I/O | I/O utilization | > 70% (writes are worse than reads) |
| Network | bandwidth utilization | > 70% |

### The Saturation → Latency Cascade

Saturation causes latency, which causes errors. The cascade looks like this:

```
DB connection pool fills up (saturation)
    ↓
New requests queue waiting for a connection (latency increases)
    ↓
Queue wait exceeds HTTP timeout (errors increase)
    ↓
Upstream retries amplify load (traffic increases)
    ↓
Everything gets worse faster
```

If you observe saturation early enough, you can intervene *before* latency and errors degrade. This is why saturation is the most operationally valuable signal — but also the hardest to measure, because you need to know your resource limits.

---

## 2.5 The Four Signals in Practice

Here's how the four signals interact during a real incident:

### Scenario: Database Index Dropped Accidentally

```
T+0:00   Index dropped. SELECT queries begin full table scans.
T+0:30   P99 latency climbs: 200ms → 4,000ms (Latency signal fires)
T+1:00   Requests begin timing out: error rate 0% → 3% (Error signal fires)
T+1:30   DB CPU: 15% → 92% (Saturation signal fires)
T+2:00   Traffic begins to self-throttle as client timeouts fail fast
T+2:30   On-call receives alert. Opens Grafana.
T+3:00   Jaeger trace shows: 3.8s of 4.0s total latency in one DB call
T+3:30   pg_stat_statements shows sequential scan on orders table
T+4:00   Index recreated. All signals return to normal.
```

Total incident duration: 4 minutes from detection to resolution. **Without** the observability stack, this incident would last until the first user complaint — potentially hours.

---

## 2.6 Availability: The Derived Signal

Availability is often listed as a fifth signal, but it's actually a *derived* metric:

```
Availability = (Total Requests - Failed Requests) / Total Requests
             = Success Rate
             = 1 - Error Rate
```

For an SLO like "99.9% availability over 30 days", you're really saying: "no more than 0.1% of requests may fail over any 30-day window."

We'll cover availability engineering — SLOs and error budgets — deeply in Chapter 22.

---

## 2.7 The Relationship Between Signals

```
             ┌─────────────────────────────────────────────┐
             │              USER EXPERIENCE                │
             │  "Is the system doing useful work for me?" │
             └────────────┬───────────────────────────────┘
                          │
              ┌───────────┴────────────┐
              │                        │
         LATENCY                   ERRORS
    "How fast is it?"         "Is it working?"
              │                        │
              └───────────┬────────────┘
                          │
                     THROUGHPUT
               "How much is it doing?"
                          │
                          │
                    SATURATION
              "How close to the limit?"
                  (predicts the above)
```

The signals form a hierarchy: saturation predicts latency/error issues, latency and errors directly describe user experience, and throughput contextualises all of them.

Next, we look at two structured methodologies for applying these signals: RED and USE.
