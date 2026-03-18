# Chapter 19: Structured Logging, Distributed Tracing & SLOs

> **Part VI — SRE & Observability**

---

## 19.1 Structured Logging with slog

Every log line is a JSON object. No free-text strings. This makes Loki indexing fast and log queries deterministic.

```go
// internal/middleware/logger.go
package middleware

import (
    "log/slog"
    "net/http"
    "time"

    "github.com/go-chi/chi/v5/middleware"
    "go.opentelemetry.io/otel/trace"
)

func StructuredLogger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        ww    := middleware.NewWrapResponseWriter(w, r.ProtoMajor)
        span  := trace.SpanFromContext(r.Context())

        next.ServeHTTP(ww, r)

        slog.InfoContext(r.Context(), "http.request",
            "request_id",  middleware.GetReqID(r.Context()),
            "trace_id",    span.SpanContext().TraceID().String(),
            "method",      r.Method,
            "path",        r.URL.Path,
            "status",      ww.Status(),
            "bytes",       ww.BytesWritten(),
            "duration_ms", time.Since(start).Milliseconds(),
            "remote_addr", r.RemoteAddr,
        )
    })
}
```

**Example log output (JSON, ingested by Loki):**
```json
{
  "time":        "2025-06-15T10:23:45.123Z",
  "level":       "INFO",
  "msg":         "http.request",
  "request_id":  "e3f2a1",
  "trace_id":    "4f8b9a2c1d3e5f6a7b8c9d0e1f2a3b4c",
  "method":      "POST",
  "path":        "/api/v1/tenants/t-abc123/devices",
  "status":      201,
  "bytes":       1842,
  "duration_ms": 847,
  "remote_addr": "10.0.0.5:54321"
}
```

---

## 19.2 Log Masking Middleware

All log fields with sensitive names must be masked before emission:

```go
// internal/middleware/log_masker.go
package middleware

import (
    "log/slog"
    "strings"
)

// SensitiveFields lists field names whose values must never appear in logs.
var SensitiveFields = []string{
    "password", "passwd", "secret", "private_key", "private",
    "token", "api_key", "apikey", "auth", "credential",
    "cert", "certificate", "x-api-key",
}

type maskingHandler struct{ h slog.Handler }

func (m *maskingHandler) Handle(ctx interface{ Done() <-chan struct{} }, r slog.Record) error {
    safe := slog.NewRecord(r.Time, r.Level, r.Message, r.PC)
    r.Attrs(func(a slog.Attr) bool {
        if isSensitive(a.Key) {
            safe.AddAttrs(slog.String(a.Key, "[REDACTED]"))
        } else {
            safe.AddAttrs(a)
        }
        return true
    })
    return m.h.Handle(ctx.(interface {
        Done() <-chan struct{}
        Err() error
    }), safe)
}

func isSensitive(key string) bool {
    lower := strings.ToLower(key)
    for _, s := range SensitiveFields {
        if strings.Contains(lower, s) {
            return true
        }
    }
    return false
}
```

---

## 19.3 Loki + Fluent Bit Setup

```yaml
# deploy/docker/docker-compose.yml additions

  loki:
    image: grafana/loki:3.0.0
    command: -config.file=/etc/loki/loki.yaml
    volumes:
      - ./loki/loki.yaml:/etc/loki/loki.yaml:ro
    ports: ['3100:3100']
    networks: [iot-net]

  fluent-bit:
    image: fluent/fluent-bit:3.0
    volumes:
      - ./fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    depends_on: [loki]
    networks: [iot-net]
```

```ini
# deploy/docker/fluent-bit/fluent-bit.conf
[INPUT]
    Name              tail
    Path              /var/lib/docker/containers/*/*.log
    Parser            docker
    Tag               docker.*
    Refresh_Interval  5

[FILTER]
    Name  grep
    Match docker.*
    Regex log iotplatform

[OUTPUT]
    Name          loki
    Match         docker.*
    Host          loki
    Port          3100
    Labels        job=iot-platform
    Label_Keys    $level,$trace_id,$request_id
    Line_Format   json
```

---

## 19.4 Distributed Tracing with OpenTelemetry

```go
// internal/telemetry/tracing.go
package telemetry

import (
    "context"
    "fmt"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func InitTracer(ctx context.Context, serviceName, otlpEndpoint string) (func(), error) {
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint(otlpEndpoint),
        otlptracehttp.WithInsecure(), // use TLS in production
    )
    if err != nil {
        return nil, fmt.Errorf("OTLP exporter: %w", err)
    }

    res, _ := resource.New(ctx,
        resource.WithAttributes(semconv.ServiceName(serviceName)),
    )

    provider := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        // Use sdktrace.TraceIDRatioBased(0.1) in high-traffic production
        sdktrace.WithSampler(sdktrace.AlwaysSample()),
    )
    otel.SetTracerProvider(provider)
    return func() { provider.Shutdown(context.Background()) }, nil
}
```

**Instrumenting a service method:**
```go
// internal/device/service.go — add spans to key operations
func (s *Service) Provision(ctx context.Context, req ProvisionReq) (*ProvisionResult, error) {
    tracer := otel.Tracer("device-service")
    ctx, span := tracer.Start(ctx, "device.provision")
    defer span.End()

    span.SetAttributes(
        attribute.String("tenant.id",  req.TenantID.String()),
        attribute.String("device.name", req.Name),
    )

    // ... provisioning logic ...
    // span.RecordError(err) on failures
}
```

---

## 19.5 SLOs and Error Budgets

| SLO | Target | Error Budget (30 days) |
|---|---|---|
| Device provisioning success rate | ≥ 99.5% | ~3.6 hours of allowed failures |
| API availability (HTTP non-5xx) | ≥ 99.9% | ~43 minutes |
| API P95 latency < 2 seconds | ≥ 99% | ~7 hours above threshold |
| Certificate rotation: zero missed | 100% rotated before expiry | 0 — no exceptions |
| Audit log completeness | 100% events logged | 0 — every operation must be recorded |

**Error budget Prometheus recording rule:**
```yaml
# In prometheus rules file
- record: iot:api_availability:ratio_rate30d
  expr: |
    1 - (
      rate(iotplatform_api_errors_total[30d])
      /
      rate(iotplatform_api_request_duration_seconds_count[30d])
    )

- record: iot:error_budget_remaining:ratio
  expr: |
    (iot:api_availability:ratio_rate30d - 0.999) / (1 - 0.999)
```

> **SRE Policy:** When `iot:error_budget_remaining:ratio < 0.5` (50% budget consumed), the engineering lead is paged. When `< 0` (budget exhausted), feature deployments are frozen until the budget resets at the start of the next calendar month.

---

## 19.6 Incident Response Runbooks

### Runbook: Device Provisioning Failure Spike

```
Symptom: HighProvisioningErrorRate alert fires.
         Provisioning error rate > 1% sustained for > 2 minutes.

Step 1 — Identify error pattern:
  kubectl logs -l app=iot-api -n iot-platform --since=10m \
    | jq 'select(.level=="ERROR") | {msg, error, provider, tenant_id}'

  Common patterns:
  a) "vault pki issue failed"    → Vault PKI unreachable (check VaultUnreachable alert)
  b) "provider registration"     → AWS/Azure API error (check provider logs + quotas)
  c) "device quota exceeded"     → Tenant at limit (expected, not a bug)
  d) "database connection"       → PostgreSQL issue (check DatabaseUnreachable alert)

Step 2 — Check provider-specific errors:
  # For AWS adapter
  kubectl logs -l app=iot-api | jq 'select(.msg | contains("CreateThing"))'

  # For Vault
  vault audit list  # Check Vault audit log

Step 3 — Immediate mitigation:
  # If Vault is down → escalate to P1 (all provisioning blocked)
  # If one provider failing → check if other providers are healthy
  # If quota errors → expected, notify tenant ops team

Step 4 — Verify recovery:
  # Watch error rate return to normal:
  watch -n5 'curl -s localhost:9090/api/v1/query?query=\
    rate(iotplatform_api_errors_total[1m]) | jq'

Step 5 — Post-incident:
  Write blameless post-mortem within 5 business days.
  File bug if root cause was platform code.
```

### Runbook: Certificate Rotation Failure

```
Symptom: CertificatesExpiringSoon fires AND rotation has not completed
         within 24 hours of the initial alert.

Step 1 — Find failed rotations in audit log:
  GET /api/v1/tenants/{id}/audit-logs
    ?action=device.cert.rotate&outcome=failure&since=24h

Step 2 — Check error from logs:
  kubectl logs -l app=iot-api | jq 'select(.action=="device.cert.rotate")'

Step 3 — Manual rotation for stuck devices:
  POST /api/v1/tenants/{tid}/devices/{did}/certificates/rotate
  -H "Authorization: Bearer $ADMIN_TOKEN"

Step 4 — If Vault PKI is the issue:
  vault pki health-check pki/
  # Restore from Vault snapshot if corrupted
  # After Vault recovery, ExpiryMonitor will auto-retry in ≤ 6 hours

Step 5 — Verify fix:
  GET /api/v1/tenants/{tid}/devices/{did}/certificates
  # Confirm new cert not_after > (today + 30 days)
```
