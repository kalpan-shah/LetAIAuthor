# Chapter 18: Metrics with Prometheus & Grafana

> **Part VI — SRE & Observability**

Metrics answer questions like: how many devices provisioned in the last hour? What is the P95 latency of the provisioning API? How many certificates are expiring in the next 30 days? Without metrics, you find out about problems from customer complaints. With good metrics, your alerts fire before customers notice.

---

## 18.1 Instrumentation Strategy

The platform instruments three layers:

- **RED metrics per service endpoint:** Rate (requests/sec), Errors (errors/sec), Duration (latency histogram). Applied to every HTTP route via the Prometheus middleware.
- **USE metrics per infrastructure component:** Utilisation, Saturation, Errors for the PostgreSQL connection pool, Redis, NATS, and Vault. These indicate infrastructure health.
- **Business metrics:** Active devices per tenant, certificates expiring soon, provisioning success/failure rates. These measure platform health from the customer's perspective.

---

## 18.2 Prometheus Metric Definitions

```go
// internal/metrics/metrics.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // ── HTTP RED Metrics ──────────────────────────────────────────────────────
    APILatency = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "iotplatform", Subsystem: "api",
        Name:    "request_duration_seconds",
        Help:    "HTTP request latency by route and status",
        Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0},
    }, []string{"method", "route", "status"})

    APIErrors = promauto.NewCounterVec(prometheus.CounterOpts{
        Namespace: "iotplatform", Subsystem: "api",
        Name: "errors_total",
        Help: "Total API errors by route and error code",
    }, []string{"route", "code"})

    // ── Device Business Metrics ───────────────────────────────────────────────
    DeviceProvisionDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "iotplatform", Subsystem: "device",
        Name:    "provision_duration_seconds",
        Help:    "End-to-end device provisioning time including cert issuance",
        Buckets: []float64{0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0},
    }, []string{"provider", "outcome"})

    ActiveDevices = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Namespace: "iotplatform", Subsystem: "device",
        Name: "active_total",
        Help: "Number of active devices per provider",
    }, []string{"provider"})

    // ── Certificate Metrics ───────────────────────────────────────────────────
    CertsExpiringSoon = promauto.NewGauge(prometheus.GaugeOpts{
        Namespace: "iotplatform", Subsystem: "certificate",
        Name: "expiring_soon_total",
        Help: "Certificates expiring within the configured warning window (default 30 days)",
    })

    CertRotationDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "iotplatform", Subsystem: "certificate",
        Name:    "rotation_duration_seconds",
        Help:    "Certificate rotation latency",
        Buckets: prometheus.DefBuckets,
    }, []string{"provider", "outcome"})

    // ── Vault Metrics ─────────────────────────────────────────────────────────
    VaultOperationDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "iotplatform", Subsystem: "vault",
        Name:    "operation_duration_seconds",
        Help:    "Vault PKI and KV operation latency",
        Buckets: prometheus.DefBuckets,
    }, []string{"operation", "outcome"})
)

// Usage in device service:
// timer := prometheus.NewTimer(
//     metrics.DeviceProvisionDuration.WithLabelValues(provider, "success"))
// defer timer.ObserveDuration()
```

---

## 18.3 Prometheus Middleware

```go
// internal/middleware/prometheus.go
package middleware

import (
    "net/http"
    "strconv"
    "time"

    "github.com/go-chi/chi/v5"
    "github.com/your-org/iot-platform/internal/metrics"
)

func Prometheus(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        ww    := newWrappedWriter(w)

        next.ServeHTTP(ww, r)

        route  := chi.RouteContext(r.Context()).RoutePattern()
        status := strconv.Itoa(ww.statusCode)

        metrics.APILatency.WithLabelValues(r.Method, route, status).
            Observe(time.Since(start).Seconds())

        if ww.statusCode >= 500 {
            metrics.APIErrors.WithLabelValues(route, "server_error").Inc()
        } else if ww.statusCode >= 400 {
            metrics.APIErrors.WithLabelValues(route, "client_error").Inc()
        }
    })
}
```

---

## 18.4 Prometheus Alerting Rules

```yaml
# deploy/docker/prometheus/alerts.yml
groups:
  - name: iot-platform
    rules:

    # Critical: provisioning error rate > 1% for 2 minutes
    - alert: HighProvisioningErrorRate
      expr: |
        rate(iotplatform_api_errors_total{route=~".*/devices$"}[5m])
        /
        rate(iotplatform_api_request_duration_seconds_count{route=~".*/devices$"}[5m])
        > 0.01
      for: 2m
      labels: { severity: critical, team: platform }
      annotations:
        summary: "Device provisioning error rate > 1% for 2m"
        runbook: "https://runbooks.internal/iot/provisioning-failures"

    # Warning: certificates expiring within 30 days
    - alert: CertificatesExpiringSoon
      expr: iotplatform_certificate_expiring_soon_total > 0
      for: 30m
      labels: { severity: warning }
      annotations:
        summary: "{{ $value }} certificates need rotation within 30 days"

    # Warning: API P95 latency > 2 seconds
    - alert: SlowAPILatency
      expr: |
        histogram_quantile(0.95,
          rate(iotplatform_api_request_duration_seconds_bucket[5m])
        ) > 2.0
      for: 5m
      labels: { severity: warning }
      annotations:
        summary: "API P95 latency > 2s for 5 minutes"

    # Critical: Vault unreachable
    - alert: VaultUnreachable
      expr: up{job="vault"} == 0
      for: 1m
      labels: { severity: critical }
      annotations:
        summary: "Vault is down — certificate operations will fail immediately"

    # Critical: PostgreSQL unreachable
    - alert: DatabaseUnreachable
      expr: up{job="postgres"} == 0
      for: 1m
      labels: { severity: critical }
      annotations:
        summary: "PostgreSQL unreachable — platform is fully degraded"

    # Warning: tenant device quota > 90%
    - alert: TenantQuotaNearLimit
      expr: |
        iotplatform_device_active_total
        / iotplatform_tenant_max_devices
        > 0.9
      for: 10m
      labels: { severity: warning }
      annotations:
        summary: "Tenant device quota > 90% for {{ $labels.tenant_id }}"
```

---

## 18.5 Grafana Dashboard Panels

| Panel | PromQL Query |
|---|---|
| Provisioning Rate (req/min) | `rate(iotplatform_device_provision_duration_seconds_count[1m]) * 60` |
| Provisioning P95 Latency | `histogram_quantile(0.95, rate(iotplatform_device_provision_duration_seconds_bucket[5m]))` |
| Active Devices (by provider) | `iotplatform_device_active_total` |
| API Error Rate (%) | `rate(iotplatform_api_errors_total[5m]) / rate(iotplatform_api_request_duration_seconds_count[5m]) * 100` |
| Certs Expiring Soon | `iotplatform_certificate_expiring_soon_total` |
| Vault PKI P95 Latency | `histogram_quantile(0.95, rate(iotplatform_vault_operation_duration_seconds_bucket{operation="issue_cert"}[5m]))` |
| API P95 Latency | `histogram_quantile(0.95, rate(iotplatform_api_request_duration_seconds_bucket[5m]))` |
| DB Connection Pool Usage | `iotplatform_db_pool_acquired / iotplatform_db_pool_max` |

---

## 18.6 Prometheus Configuration

```yaml
# deploy/docker/prometheus/prometheus.yml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/alerts.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: [alertmanager:9093]

scrape_configs:
  - job_name: iot-api
    static_configs:
      - targets: [api:8080]
    metrics_path: /metrics

  - job_name: vault
    static_configs:
      - targets: [vault:8200]
    metrics_path: /v1/sys/metrics
    params:
      format: [prometheus]
    bearer_token: root  # Use Vault token in dev; service token in prod

  - job_name: postgres
    static_configs:
      - targets: [postgres-exporter:9187]

  - job_name: nats
    static_configs:
      - targets: [nats:8222]
    metrics_path: /metrics
```
