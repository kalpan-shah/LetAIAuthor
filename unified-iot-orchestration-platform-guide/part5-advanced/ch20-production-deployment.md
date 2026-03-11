# Chapter 20: Production Deployment Strategies

> *"In theory, there is no difference between theory and practice. In practice, there is."* — Yogi Berra

---

## 20.1 Introduction

This final chapter brings everything together — from the Kubernetes manifests and Helm charts that package the platform, to the production readiness checklist that ensures nothing is forgotten before going live.

---

## 20.2 Helm Chart Structure

```
helm/iot-platform/
├── Chart.yaml
├── values.yaml
├── values-staging.yaml
├── values-production.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml              # Horizontal Pod Autoscaler
│   ├── pdb.yaml               # Pod Disruption Budget
│   ├── networkpolicy.yaml
│   ├── serviceaccount.yaml
│   └── ingress.yaml
└── charts/                     # Subcharts for dependencies
```

### Core Values

```yaml
# values-production.yaml
replicaCount: 3

image:
  repository: ghcr.io/org/iot-platform
  pullPolicy: IfNotPresent
  tag: ""  # Set by CI/CD

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

podDisruptionBudget:
  minAvailable: 2

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2

env:
  DATABASE_URL: "postgres://platform:$(DB_PASSWORD)@postgres:5432/iot_platform?sslmode=require"
  REDIS_URL: "redis://redis:6379/0"
  NATS_URL: "nats://nats:4222"
  VAULT_ADDR: "https://vault:8200"
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
```

---

## 20.3 Health Check Endpoints

🐍 **Python**:

```python
# app/routers/health.py
from fastapi import APIRouter, Response
import asyncio

router = APIRouter()


@router.get("/healthz")
async def liveness(request):
    """Liveness: Is the process alive? Returns 200 if yes."""
    return {"status": "alive"}


@router.get("/readyz")
async def readiness(request):
    """Readiness: Can this instance handle traffic?"""
    checks = {}

    # Database connectivity
    try:
        async with request.app.state.db_pool.acquire() as conn:
            await conn.fetchval("SELECT 1")
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"

    # Redis connectivity
    try:
        await request.app.state.redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"

    # NATS connectivity
    try:
        if request.app.state.nats.is_connected:
            checks["nats"] = "ok"
        else:
            checks["nats"] = "disconnected"
    except Exception as e:
        checks["nats"] = f"error: {e}"

    all_ok = all(v == "ok" for v in checks.values())
    status_code = 200 if all_ok else 503

    return Response(
        content=json.dumps({"status": "ready" if all_ok else "not_ready", "checks": checks}),
        status_code=status_code,
        media_type="application/json",
    )
```

🔵 **Go**:

```go
// internal/health/handler.go
package health

import (
    "context"
    "encoding/json"
    "net/http"
    "time"
)

type Checker struct {
    DB    *pgxpool.Pool
    Redis *redis.Client
    NATS  *nats.Conn
}

func (c *Checker) Liveness(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "alive"})
}

func (c *Checker) Readiness(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    checks := map[string]string{}
    allOk := true

    // Check database
    if err := c.DB.Ping(ctx); err != nil {
        checks["database"] = "error: " + err.Error()
        allOk = false
    } else {
        checks["database"] = "ok"
    }

    // Check Redis
    if err := c.Redis.Ping(ctx).Err(); err != nil {
        checks["redis"] = "error: " + err.Error()
        allOk = false
    } else {
        checks["redis"] = "ok"
    }

    // Check NATS
    if c.NATS.IsConnected() {
        checks["nats"] = "ok"
    } else {
        checks["nats"] = "disconnected"
        allOk = false
    }

    status := http.StatusOK
    if !allOk {
        status = http.StatusServiceUnavailable
    }

    w.WriteHeader(status)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "status": map[bool]string{true: "ready", false: "not_ready"}[allOk],
        "checks": checks,
    })
}
```

---

## 20.4 Graceful Shutdown

🔵 **Go**:

```go
// cmd/server/main.go (enhanced)
func main() {
    // ... setup ...

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        <-quit
        slog.Info("shutting down gracefully...")

        // 1. Stop accepting new connections
        grpcServer.GracefulStop()

        // 2. Wait for in-flight requests (with timeout)
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()

        // 3. Close database pool
        dbPool.Close()

        // 4. Drain NATS connections
        natsConn.Drain()

        // 5. Close Redis
        redisClient.Close()

        slog.Info("shutdown complete")
    }()

    slog.Info("server started", "grpc", ":9090", "http", ":8080")
    select {} // Block forever until signal
}
```

🐍 **Python** (via FastAPI lifespan):

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.db_pool = await create_pool(os.environ["DATABASE_URL"])
    app.state.redis = Redis.from_url(os.environ["REDIS_URL"])
    app.state.nats = await nats.connect(os.environ["NATS_URL"])

    yield  # Application runs

    # Shutdown (graceful)
    await app.state.nats.drain()
    await app.state.redis.close()
    await app.state.db_pool.close()
```

---

## 20.5 Production Readiness Checklist

### Infrastructure

- [ ] PostgreSQL: HA cluster (primary + replica), backups verified
- [ ] Redis: Sentinel or Cluster mode, maxmemory configured
- [ ] NATS: 3-node JetStream cluster, persistence enabled
- [ ] Vault: 3-node Raft cluster, auto-unseal configured
- [ ] Kubernetes: Multi-AZ node spread, PDB configured

### Application

- [ ] Health endpoints (`/healthz`, `/readyz`) implemented
- [ ] Graceful shutdown handles in-flight requests
- [ ] Structured JSON logging enabled
- [ ] OpenTelemetry traces, metrics, and logs exported
- [ ] All environment variables documented and set
- [ ] Database migrations run and verified
- [ ] Connection pool sizes tuned for instance count

### Security

- [ ] TLS on all endpoints (external and internal)
- [ ] JWT authentication with short TTL
- [ ] RBAC enforced on all API endpoints
- [ ] Row-Level Security enabled on all tenant tables
- [ ] Vault policies follow least privilege
- [ ] Container images scanned and non-root
- [ ] NetworkPolicies applied
- [ ] Secrets not in env vars or config files
- [ ] API rate limiting per tenant enabled

### Observability

- [ ] Grafana dashboards deployed (platform overview, provider health)
- [ ] Alerting rules configured (latency, errors, certificates, quotas)
- [ ] On-call rotation established
- [ ] Runbooks written for top 5 failure scenarios
- [ ] SLOs defined and tracked

### Operations

- [ ] CI/CD pipeline with security scanning gates
- [ ] Canary deployment strategy configured
- [ ] Backup and restore procedure tested
- [ ] Disaster recovery plan documented
- [ ] Chaos testing schedule established
- [ ] Error budget policy agreed upon

---

## 20.6 First Day in Production

```
Hour 0: Deploy
  └─ helm install → staging verification → production canary (5%)

Hour 1: Validate
  └─ Check dashboards, verify SLIs tracking
  └─ Run smoke tests against production API

Hours 2-24: Monitor
  └─ Watch error rates, latency, provider API calls
  └─ Verify audit log completeness
  └─ Check certificate expiry scanner running

Day 2+: Operate
  └─ Onboard first real tenant
  └─ Provision devices, verify end-to-end flow
  └─ Monitor quota utilization
  └─ Review first audit log entries
```

---

## 20.7 Conclusion

You've built a complete IoT orchestration platform — from architecture vision to production deployment. Here's what we've covered across 20 chapters:

| Part | Chapters | What You Built |
|------|----------|---------------|
| **I: Foundations** | 1-3 | Architecture, tech stack, dev environment |
| **II: Core Infrastructure** | 4-6 | Database, provider abstraction, secrets management |
| **III: Platform Services** | 7-12 | Tenants, devices, certs, policies, provisioning, routing |
| **IV: Operations** | 13-17 | Multi-provider, observability, SRE, security, DevOps |
| **V: Advanced** | 18-20 | Performance, DR/HA, production deployment |

The platform you've built is:

- **Provider-agnostic** — switch between AWS, Azure, and Mosquitto through a unified interface
- **Multi-tenant** — isolated at API, application, database, and topic levels
- **Secure** — defense-in-depth from container to network to data layer
- **Observable** — metrics, traces, and logs with production alerting
- **Production-ready** — HA, DR, graceful shutdown, and canary deployments

> 💡 **What comes next?** Here are natural extensions:
> - **Web management console** (React/Vue dashboard)
> - **CLI tool** (cobra-based Go CLI or Python click)
> - **SDK** (Go and Python client libraries)
> - **Device fleet management** (OTA updates, fleet-wide commands)
> - **Multi-region active-active** (CockroachDB or Spanner for global state)
> - **AI/ML integration** (anomaly detection on device provisioning patterns)

Thank you for reading. May your devices provision cleanly, your certificates never expire unnoticed, and your tenants always stay in their lane. 🚀
