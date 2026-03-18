# Chapter 27: Production Readiness Checklist

> **Part VIII — DevOps**

Use this checklist before every production deployment. Items marked **[CRITICAL]** are hard blockers — do not go live until they are complete.

---

## 27.1 Security Checklist

| # | Item | Status |
|---|---|---|
| 1 | **[CRITICAL]** Vault is NOT running in dev mode — storage backend configured (Consul, Raft, or S3) and auto-unseal configured (AWS KMS, Azure Key Vault, or GCP KMS) | ☐ |
| 2 | **[CRITICAL]** No `VAULT_TOKEN=root` or any hardcoded token in production configs, Kubernetes Secrets, or ConfigMaps | ☐ |
| 3 | **[CRITICAL]** All secrets delivered via Vault Agent Injector — no secrets in env vars in Kubernetes Deployments | ☐ |
| 4 | **[CRITICAL]** TLS 1.2+ required for all connections: device→broker, client→API, platform→DB, platform→Vault | ☐ |
| 5 | **[CRITICAL]** Kubernetes NetworkPolicy deployed — default-deny, explicit allow rules only | ☐ |
| 6 | **[CRITICAL]** All pods run as non-root (`runAsNonRoot: true`, `runAsUser: 65534`) | ☐ |
| 7 | JWT_SECRET is a cryptographically random 256-bit value (`openssl rand -hex 32`) — not a human-readable string | ☐ |
| 8 | Mosquitto plaintext port 1883 disabled in production — TLS-only on port 8883 | ☐ |
| 9 | GitLeaks secret scanning runs on every PR and pre-commit | ☐ |
| 10 | Trivy vulnerability scanning in CI — build fails on CRITICAL/HIGH CVEs | ☐ |
| 11 | Audit log table: platform DB user granted no `UPDATE` or `DELETE` permissions | ☐ |
| 12 | IAM roles for AWS providers use `ExternalID` = tenant_id in all trust policies | ☐ |
| 13 | Security headers middleware active: HSTS, X-Frame-Options, CSP, X-Content-Type-Options | ☐ |
| 14 | API rate limiting active: 1,000 req/min per IP, 10 provisioning req/min per tenant | ☐ |

---

## 27.2 Reliability Checklist

| # | Item | Status |
|---|---|---|
| 1 | **[CRITICAL]** API Deployment: `replicas: 3` minimum, `maxUnavailable: 0` in rolling update strategy | ☐ |
| 2 | **[CRITICAL]** PodDisruptionBudget deployed: `minAvailable: 2` for iot-api | ☐ |
| 3 | **[CRITICAL]** Liveness and readiness probes configured on ALL pods — not just iot-api | ☐ |
| 4 | **[CRITICAL]** HPA deployed with appropriate min/max replicas | ☐ |
| 5 | **[CRITICAL]** Certificate ExpiryMonitor running, connected to NATS, Prometheus metric visible | ☐ |
| 6 | **[CRITICAL]** PostgreSQL automated backups configured and tested — restored successfully in last 30 days | ☐ |
| 7 | **[CRITICAL]** Prometheus alerting rules deployed and alerts tested end-to-end (actually received in PagerDuty/Opsgenie) | ☐ |
| 8 | Vault HA mode: 3+ nodes with Raft storage, or DR replication to a secondary cluster | ☐ |
| 9 | NATS JetStream stream configured with `replicas: 3` (requires 3-node NATS cluster) | ☐ |
| 10 | PostgreSQL read replica configured for audit log queries (avoids load on primary) | ☐ |
| 11 | `topologySpreadConstraints` spreads iot-api pods across availability zones | ☐ |
| 12 | Rollback procedure documented and tested: `kubectl rollout undo deployment/iot-api` | ☐ |
| 13 | Graceful shutdown implemented: API catches SIGTERM, stops accepting new requests, drains in-flight (30s timeout) | ☐ |
| 14 | Database migration runs as an init container or pre-deploy job — never inside the running API | ☐ |

---

## 27.3 Observability Checklist

| # | Item | Status |
|---|---|---|
| 1 | Grafana platform dashboard deployed, RED metrics and business metrics visible | ☐ |
| 2 | Loki log aggregation running — able to search API logs by request_id and trace_id in Grafana Explore | ☐ |
| 3 | Jaeger / Tempo traces visible for end-to-end provisioning request traces | ☐ |
| 4 | PagerDuty or Opsgenie connected to Prometheus Alertmanager | ☐ |
| 5 | On-call rotation defined, tested (someone received a test page) | ☐ |
| 6 | Runbooks written for top 5 alerts and linked in alert `annotations.runbook` field | ☐ |
| 7 | SLO error budget dashboard accessible to the whole engineering team | ☐ |
| 8 | Log retention policy configured in Loki (recommend 30d hot, 1y cold object storage) | ☐ |
| 9 | Audit logs archive policy configured (quarterly partition → immutable S3 Object Lock) | ☐ |

---

## 27.4 Load Testing Checklist

Run load tests against staging before every major release:

```bash
# Install k6 (https://k6.io)
brew install k6

# Run provisioning load test
k6 run scripts/load-test-provision.js \
  -e BASE_URL=https://staging-api.iotplatform.io \
  -e TENANT_ID=t-staging-test \
  -e TOKEN=$(cat /tmp/staging-token)
```

```javascript
// scripts/load-test-provision.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m',  target: 10  },  // Ramp up
    { duration: '5m',  target: 50  },  // Sustain
    { duration: '2m',  target: 100 },  // Peak
    { duration: '1m',  target: 0   },  // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],  // 95% of requests < 2s
    http_req_failed:   ['rate<0.01'],   // Error rate < 1%
  },
};

export default function () {
  const res = http.post(
    `${__ENV.BASE_URL}/api/v1/tenants/${__ENV.TENANT_ID}/devices`,
    JSON.stringify({ name: `load-test-${Date.now()}` }),
    {
      headers: {
        'Content-Type': 'application/json',
        Authorization:  `Bearer ${__ENV.TOKEN}`,
      },
    }
  );
  check(res, {
    'status is 201': r => r.status === 201,
    'has device id': r => r.json('device.id') !== undefined,
    'has private key': r => r.json('certificate.private_key_pem') !== undefined,
  });
  sleep(1);
}
```

---

## 27.5 Day-2 Operations Quick Reference

```bash
# ── Deployments ──────────────────────────────────────────────────────────────
kubectl rollout status deployment/iot-api -n iot-platform
kubectl rollout undo  deployment/iot-api -n iot-platform    # Rollback
kubectl scale deployment iot-api --replicas=10 -n iot-platform

# ── Logs ─────────────────────────────────────────────────────────────────────
kubectl logs -l app=iot-api -n iot-platform --tail=100 | jq .
kubectl logs -l app=iot-api -n iot-platform --since=30m | \
  jq 'select(.level=="ERROR") | {msg, error, trace_id}'

# ── Vault ─────────────────────────────────────────────────────────────────────
vault status
vault read pki/cert/ca
vault list pki/certs
vault audit list

# ── NATS ─────────────────────────────────────────────────────────────────────
nats stream ls
nats stream info IOT_PLATFORM_EVENTS
nats consumer ls IOT_PLATFORM_EVENTS

# ── Database ─────────────────────────────────────────────────────────────────
psql $DATABASE_URL -c "SELECT count(*) FROM devices WHERE status='active';"
psql $DATABASE_URL -c \
  "SELECT action, outcome, count(*) FROM audit_logs \
   WHERE created_at > NOW() - INTERVAL '1 hour' \
   GROUP BY action, outcome ORDER BY count DESC;"

# ── Certificate Operations ────────────────────────────────────────────────────
# Force rotation for a specific device
curl -X POST https://api.iotplatform.io/api/v1/tenants/{tid}/devices/{did}/certificates/rotate \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# List certificates expiring in next 30 days
psql $DATABASE_URL -c \
  "SELECT d.name, c.not_after, c.status \
   FROM certificates c JOIN devices d ON d.current_cert_id = c.id \
   WHERE c.not_after < NOW() + INTERVAL '30 days' \
   AND c.status = 'active' ORDER BY c.not_after;"
```

---

## 27.6 What Comes Next

This handbook covers the complete platform foundation. From here, teams commonly extend the platform with:

| Extension | Approach |
|---|---|
| **Device Fleet Management** | Group devices by tags; API for bulk operations (mass suspend, mass cert rotation) |
| **OTA Firmware Updates** | Add `tenant/{t}/device/{d}/ota` topic; platform manages update state machine in PostgreSQL |
| **Real-time Telemetry Dashboards** | Connect IoT provider outputs to TimescaleDB or InfluxDB; Grafana as the frontend |
| **Anomaly Detection** | Subscribe to telemetry via IoT Rules Engine; route to ML pipeline (SageMaker, Azure ML) |
| **Multi-Region Active-Active** | Deploy platform to multiple Kubernetes clusters; Vault Enterprise for geo-replication |
| **Device Simulator** | Build a Go-based simulator that provisions and sends telemetry for load testing |

The provider-agnostic architecture ensures all extensions can be added without rewriting the foundation.
