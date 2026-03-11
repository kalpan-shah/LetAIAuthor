# Chapter 14: Observability & Monitoring

> *"You can't fix what you can't see."*

---

## 14.1 Introduction

Operating a multi-tenant IoT control plane requires comprehensive observability. You need to answer questions like:

- "Why did tenant Acme's provisioning take 12 seconds when it usually takes 2?"
- "How many certificates are expiring in the next 7 days across all tenants?"
- "Is the Azure provider adapter experiencing higher error rates than AWS?"

This chapter covers the complete observability stack: metrics, distributed traces, structured logs, and production dashboards.

---

## 14.2 The Observability Stack

```
Application Code
      │
      ▼
┌─────────────────────────┐
│  OpenTelemetry SDK       │  ← Instrument once, export anywhere
│  (Traces + Metrics + Logs)│
└─────────┬───────────────┘
          │ OTLP
          ▼
┌─────────────────────────┐
│  OTel Collector          │  ← Route, filter, sample
│  (Pipeline processor)    │
└────┬─────┬──────┬───────┘
     │     │      │
     ▼     ▼      ▼
┌──────┐┌──────┐┌──────┐
│Prom. ││Jaeger││ Loki │
│      ││/Tempo││      │
└──┬───┘└──┬───┘└──┬───┘
   │       │       │
   └───────┼───────┘
           │
     ┌─────▼─────┐
     │  Grafana   │  ← Unified dashboards & alerting
     └───────────┘
```

---

## 14.3 Custom Platform Metrics

🐍 **Python — Business metrics with OpenTelemetry**:

```python
# app/observability/metrics.py
from opentelemetry import metrics

meter = metrics.get_meter("iot-platform", "1.0.0")

# ── Counters ──
devices_provisioned = meter.create_counter(
    name="iot.devices.provisioned",
    description="Total devices successfully provisioned",
    unit="1",
)

provisioning_failures = meter.create_counter(
    name="iot.devices.provisioning_failures",
    description="Total provisioning failures",
    unit="1",
)

certificates_issued = meter.create_counter(
    name="iot.certificates.issued",
    description="Total certificates issued",
    unit="1",
)

certificates_revoked = meter.create_counter(
    name="iot.certificates.revoked",
    description="Total certificates revoked",
    unit="1",
)

# ── Histograms ──
provisioning_duration = meter.create_histogram(
    name="iot.provisioning.duration",
    description="Time to provision a device end-to-end",
    unit="s",
)

provider_api_duration = meter.create_histogram(
    name="iot.provider.api_duration",
    description="Duration of provider API calls",
    unit="s",
)

# ── Gauges (via UpDownCounter) ──
active_devices = meter.create_up_down_counter(
    name="iot.devices.active",
    description="Current number of active devices",
    unit="1",
)

# Usage:
# devices_provisioned.add(1, {"tenant_id": "acme", "provider": "aws"})
# provisioning_duration.record(2.5, {"tenant_id": "acme", "provider": "aws"})
```

🔵 **Go — Business metrics with Prometheus**:

```go
// internal/observability/metrics.go
package observability

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    DevicesProvisioned = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "iot_devices_provisioned_total",
            Help: "Total number of devices successfully provisioned",
        },
        []string{"tenant_id", "provider"},
    )

    ProvisioningDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "iot_provisioning_duration_seconds",
            Help:    "Time to provision a device end-to-end",
            Buckets: []float64{0.1, 0.5, 1, 2, 5, 10, 30},
        },
        []string{"tenant_id", "provider"},
    )

    ProviderAPIDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "iot_provider_api_duration_seconds",
            Help:    "Duration of individual provider API calls",
            Buckets: []float64{0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
        },
        []string{"provider", "operation"},
    )

    ActiveDevicesGauge = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "iot_devices_active",
            Help: "Current number of active devices per tenant",
        },
        []string{"tenant_id"},
    )
)
```

---

## 14.4 Distributed Tracing

A device provisioning request spans multiple services and external APIs. Tracing shows the full picture:

```
Trace: provision-device
│
├─ [2.1s] POST /api/v1/devices/{id}/provision
│  ├─ [5ms]  validate_tenant
│  ├─ [10ms] check_quota (Redis)
│  ├─ [800ms] provider.create_device (AWS IoT)
│  ├─ [650ms] provider.create_certificate (AWS IoT)
│  ├─ [200ms] provider.attach_cert_to_device (AWS IoT)
│  ├─ [150ms] provider.create_policy (AWS IoT)
│  ├─ [100ms] provider.attach_policy_to_cert (AWS IoT)
│  ├─ [50ms]  database.insert_certificate
│  ├─ [30ms]  database.insert_policy
│  ├─ [20ms]  database.update_device_status
│  └─ [5ms]   nats.publish_audit_event
```

🐍 **Python — Adding custom spans**:

```python
# app/services/device_service.py (enhanced with tracing)
from opentelemetry import trace

tracer = trace.get_tracer("iot-platform.device-service")


class DeviceService:
    async def provision_device(self, tenant_id: str, device_id: str) -> dict:
        with tracer.start_as_current_span("provision_device") as span:
            span.set_attribute("tenant.id", tenant_id)
            span.set_attribute("device.id", device_id)

            with tracer.start_as_current_span("validate_tenant"):
                device = await self._get_device(tenant_id, device_id)

            with tracer.start_as_current_span("check_quota"):
                await self.quotas.check_quota(tenant_id, "device")

            with tracer.start_as_current_span("provider.create_device") as pspan:
                pspan.set_attribute("provider.name", provider.provider_name)
                result = await provider.create_device(req)
                pspan.set_attribute("provider.device_id", result.provider_device_id)

            # ... remaining steps with their own spans ...

            span.set_status(trace.StatusCode.OK)
            return result
```

---

## 14.5 Structured Logging

```python
# app/observability/logger.py
import structlog
import logging

def setup_logging(level: str = "INFO", format: str = "json"):
    """Configure structured logging with correlation IDs."""
    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer() if format == "json"
            else structlog.dev.ConsoleRenderer(),
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

# Usage:
# log = structlog.get_logger()
# log.info("device.provisioned", tenant_id="acme", device_id="xyz", duration_ms=2100)
# Output: {"event": "device.provisioned", "tenant_id": "acme", "device_id": "xyz", "duration_ms": 2100, "timestamp": "..."}
```

---

## 14.6 Key Dashboards

### Platform Overview Dashboard

| Panel | Query | Purpose |
|-------|-------|---------|
| Devices provisioned/hour | `rate(iot_devices_provisioned_total[1h])` | Throughput |
| P99 provisioning latency | `histogram_quantile(0.99, iot_provisioning_duration_seconds)` | Performance |
| Active devices by tenant | `iot_devices_active` | Capacity |
| Provider API error rate | `rate(iot_provider_api_errors_total[5m])` | Reliability |
| Certificates expiring (7d) | Custom query against DB | Operational |

### Provider Health Dashboard

| Panel | Query | Purpose |
|-------|-------|---------|
| API latency by provider | `iot_provider_api_duration_seconds{provider=~"aws|azure"}` | SLA monitoring |
| Rate limit utilization | `iot_provider_rate_limit_tokens / iot_provider_rate_limit_max` | Capacity planning |
| Auth failures by provider | `iot_provider_auth_failures_total` | Credential issues |

---

## 14.7 Alerting Rules

```yaml
# deployments/prometheus-alerts.yml
groups:
  - name: iot-platform
    rules:
      - alert: HighProvisioningLatency
        expr: histogram_quantile(0.99, rate(iot_provisioning_duration_seconds_bucket[5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 provisioning latency > 10s"

      - alert: ProviderAPIErrors
        expr: rate(iot_provider_api_errors_total[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Provider API error rate elevated"

      - alert: CertificatesExpiringSoon
        expr: iot_certificates_expiring_7d > 0
        labels:
          severity: warning
        annotations:
          summary: "{{ $value }} certificates expiring within 7 days"

      - alert: QuotaNearLimit
        expr: iot_quota_usage_ratio > 0.9
        labels:
          severity: warning
        annotations:
          summary: "Tenant {{ $labels.tenant_id }} approaching {{ $labels.resource }} quota"
```

---

## 14.8 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **OpenTelemetry** | Instrument once, export to any backend |
| **Custom metrics** | Business-specific counters and histograms per tenant/provider |
| **Distributed tracing** | Full request lifecycle visibility across services |
| **Structured logging** | JSON logs with tenant/device context for searchability |
| **Dashboards** | Platform overview + per-provider health views |
| **Alerting** | Latency, error rates, quota, and certificate expiry alerts |

### What's Next

**Chapter 15** builds **SRE Practices & Runbooks** — SLOs, error budgets, incident response, and operational runbooks.
