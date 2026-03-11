# Chapter 2: Technology Stack Selection

> *"Pick boring technology — excitement should come from the product, not the infrastructure."* — Dan McKinley

---

## 2.1 Introduction

Choosing a technology stack for a multi-tenant IoT orchestration platform isn't about picking the "best" language or database in isolation. It's about selecting components that compose well together, have mature ecosystems for the specific problems we're solving, and can be operated reliably at scale.

This chapter explains **why** we selected each component, what alternatives we considered, and when you might choose differently.

> 💡 **For Beginners**: Don't stress about picking the "perfect" stack. The architecture in Chapter 1 is stack-agnostic by design. If your team is strongest in a different language or database, the patterns still apply. What matters are the *interfaces* between components, not the implementations.

---

## 2.2 The Complete Stack at a Glance

```
┌─────────────────────────────────────────────────────────┐
│                    DEVELOPER INTERFACE                   │
│           Web Console (React/Vue) + CLI + SDK            │
├─────────────────────────────────────────────────────────┤
│                     API LAYER                            │
│        Go (gRPC + grpc-gateway)  │  Python (FastAPI)     │
├─────────────────────────────────────────────────────────┤
│                   DATA LAYER                             │
│     PostgreSQL  │  Redis (Cache)  │  NATS (Events)       │
├─────────────────────────────────────────────────────────┤
│                 SECURITY LAYER                           │
│    HashiCorp Vault  │  CFSSL/step-ca  │  OPA             │
├─────────────────────────────────────────────────────────┤
│               OBSERVABILITY LAYER                        │
│  OpenTelemetry  │  Prometheus  │  Grafana  │  Loki       │
├─────────────────────────────────────────────────────────┤
│               DEPLOYMENT LAYER                           │
│      Docker  │  Kubernetes  │  Helm  │  GitHub Actions    │
└─────────────────────────────────────────────────────────┘
```

| Layer | Technology | Role |
|-------|-----------|------|
| **API (Go)** | Go 1.22+, gRPC, grpc-gateway | Primary API server, high-performance path |
| **API (Python)** | Python 3.12+, FastAPI, uvicorn | Alternative API server, rapid prototyping |
| **Database** | PostgreSQL 16 | Primary data store, ACID compliance |
| **Cache** | Redis 7 | Session cache, rate limiting, idempotency keys |
| **Events** | NATS 2.10 | Audit events, async workflows, internal pub/sub |
| **Secrets** | HashiCorp Vault | Credential storage, dynamic secrets, PKI |
| **PKI** | CFSSL or step-ca | Certificate authority for device certs |
| **Policy** | Open Policy Agent (OPA) | Externalized authorization decisions |
| **Observability** | OpenTelemetry + Prometheus + Grafana + Loki | Metrics, traces, logs |
| **Containers** | Docker + Kubernetes | Packaging and orchestration |
| **CI/CD** | GitHub Actions or GitLab CI | Build, test, deploy pipelines |

---

## 2.3 API Layer: Go + Python (FastAPI)

### Why Two Languages?

This isn't a compromise — it's a pragmatic strategy used by many successful platforms:

| Concern | Go | Python (FastAPI) |
|---------|-----|-----------------|
| **Runtime performance** | ✅ Compiled, goroutine concurrency | ⚡ Fast for Python, but still interpreted |
| **gRPC ecosystem** | ✅ First-class, code-generated clients | ✅ grpcio works, less idiomatic |
| **Rapid prototyping** | Slower iteration cycle | ✅ Interactive, dynamic typing |
| **Type safety** | ✅ Compile-time checks | Runtime validation (Pydantic) |
| **Deployment** | ✅ Single static binary, tiny containers | Needs Python runtime, larger images |
| **Team familiarity** | Systems/infra teams | Data/ML/backend teams |
| **Ecosystem for IoT** | Strong (Kubernetes, Terraform, Consul) | Strong (boto3, azure-sdk, paho-mqtt) |

> 💡 **How to choose**: If your team is building the "production infrastructure" version, use Go. If your team is prototyping, building internal tools, or has strong Python expertise, use FastAPI. Both implement the same interfaces and can be swapped at the service level.

### Go: The High-Performance Path

Go is the language of cloud-native infrastructure. Nearly every tool in the CNCF landscape — Kubernetes, Prometheus, Vault, Terraform, Consul — is written in Go.

For our platform, Go provides:

```go
// cmd/server/main.go — Go server startup
package main

import (
    "context"
    "log/slog"
    "net"
    "net/http"

    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    pb "iot-platform/api/proto/v1"
    "iot-platform/internal/tenant"
    "iot-platform/internal/device"
    "iot-platform/internal/middleware"
)

func main() {
    // gRPC server
    lis, _ := net.Listen("tcp", ":9090")
    grpcServer := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            middleware.TenantInterceptor(),
            middleware.AuditInterceptor(),
            middleware.QuotaInterceptor(),
        ),
    )

    // Register services
    pb.RegisterTenantServiceServer(grpcServer, tenant.NewService())
    pb.RegisterDeviceServiceServer(grpcServer, device.NewService())

    // REST gateway (translates HTTP/JSON to gRPC)
    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
    pb.RegisterTenantServiceHandlerFromEndpoint(context.Background(), mux, ":9090", opts)
    pb.RegisterDeviceServiceHandlerFromEndpoint(context.Background(), mux, ":9090", opts)

    slog.Info("starting platform", "grpc", ":9090", "http", ":8080")

    go grpcServer.Serve(lis)
    http.ListenAndServe(":8080", mux)
}
```

**Why this matters**: With `grpc-gateway`, you define your API **once** in protobuf and get both gRPC and REST automatically. No maintaining two codebases.

### Python: The Rapid Development Path

FastAPI gives you automatic OpenAPI documentation, Pydantic validation, and async-first design:

```python
# main.py — FastAPI server startup
from contextlib import asynccontextmanager
from fastapi import FastAPI

from app.middleware.tenant import TenantMiddleware
from app.middleware.audit import AuditMiddleware
from app.routers import tenants, devices, certificates


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown hooks."""
    # Startup: connect to DB, Vault, NATS
    await app.state.db_pool.connect()
    await app.state.vault_client.authenticate()
    await app.state.nats_client.connect()
    yield
    # Shutdown: clean up
    await app.state.db_pool.disconnect()
    await app.state.nats_client.close()


app = FastAPI(
    title="Unified IoT Orchestration Platform",
    version="1.0.0",
    description="Control plane for multi-provider IoT device management",
    lifespan=lifespan,
)

# Middleware stack (applied in reverse order)
app.add_middleware(AuditMiddleware)
app.add_middleware(TenantMiddleware)

# Route registration
app.include_router(tenants.router, prefix="/api/v1/tenants", tags=["Tenants"])
app.include_router(devices.router, prefix="/api/v1/devices", tags=["Devices"])
app.include_router(certificates.router, prefix="/api/v1/certificates", tags=["Certificates"])
```

> ☕ *Java/C# note*: The FastAPI approach is similar to Spring Boot's `@SpringBootApplication` with `@Bean` configuration, but with significantly less boilerplate. The `lifespan` context manager replaces `@PostConstruct` / `@PreDestroy`. ASP.NET Core developers will see parallels with `Program.cs` and `builder.Services`.

---

## 2.4 Database: PostgreSQL

### Why PostgreSQL Over Everything Else?

| Requirement | PostgreSQL | MongoDB | DynamoDB | CockroachDB |
|------------|-----------|---------|----------|-------------|
| ACID transactions | ✅ Full | ⚠️ Single-doc only* | ⚠️ 25-item limit | ✅ Full |
| JSON support | ✅ `jsonb` | ✅ Native | ✅ Native | ✅ Via GIN |
| Row-level security | ✅ Built-in | ❌ | ❌ | ✅ |
| Multi-tenant isolation | ✅ RLS + schema | Manual | Manual | ✅ |
| Provider agnostic | ✅ Self-hosted or cloud | ✅ | ❌ AWS only | ✅ |
| Operational maturity | 30+ years | 15+ years | 12+ years | 8+ years |
| Cost at scale | Low | Medium | High (per-request) | Medium |

*MongoDB 4.0+ supports multi-document transactions, but they carry significant performance overhead.

**The key differentiator is Row-Level Security (RLS)**. PostgreSQL can enforce tenant isolation at the database level:

```sql
-- Tenant isolation via Row-Level Security
ALTER TABLE devices ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_policy ON devices
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Now this query automatically filters by tenant:
-- SELECT * FROM devices;
-- Becomes: SELECT * FROM devices WHERE tenant_id = 'current-tenant';
```

> ⚠️ **Important**: RLS is a safety net, not a replacement for application-level tenant filtering. We enforce tenant isolation at **both** the application middleware layer and the database layer. Defense in depth.

### Schema Design Philosophy

We use a **shared schema** model (all tenants in the same tables, isolated by `tenant_id`) rather than schema-per-tenant or database-per-tenant:

| Approach | Pros | Cons | When to Use |
|----------|------|------|-------------|
| **Shared schema** (our choice) | Simple operations, easy queries | Noisy neighbor risk | < 10,000 tenants |
| Schema-per-tenant | Good isolation | Migration complexity | 100–10,000 tenants |
| Database-per-tenant | Maximum isolation | Operational nightmare | Regulated industries |

> Detailed schema design is covered in **Chapter 4**.

---

## 2.5 Cache & Messaging: Redis + NATS

### Redis: More Than a Cache

Redis serves four roles in our platform:

| Role | Usage | Example |
|------|-------|---------|
| **Cache** | Provider endpoint caching | Avoid re-querying AWS for IoT endpoint every request |
| **Rate limiter** | Per-tenant API rate limiting | Sliding window counter |
| **Idempotency store** | Prevent duplicate provisioning | Store request hash for 24h |
| **Distributed lock** | Certificate rotation coordination | Prevent concurrent rotation for same device |

🔵 **Go — Redis-based idempotency**:

```go
// internal/middleware/idempotency.go
func IdempotencyCheck(rdb *redis.Client) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
        key := computeIdempotencyKey(ctx, req)
        
        // Try to set the key with NX (only if not exists)
        set, err := rdb.SetNX(ctx, "idempotency:"+key, "processing", 24*time.Hour).Result()
        if err != nil {
            return nil, status.Error(codes.Internal, "idempotency check failed")
        }
        
        if !set {
            // Request already processed or in-progress
            cached, _ := rdb.Get(ctx, "idempotency:result:"+key).Bytes()
            if cached != nil {
                return unmarshalCachedResponse(info.FullMethod, cached), nil
            }
            return nil, status.Error(codes.AlreadyExists, "duplicate request in progress")
        }
        
        resp, err := handler(ctx, req)
        if err == nil {
            // Cache successful result
            data, _ := marshalResponse(resp)
            rdb.Set(ctx, "idempotency:result:"+key, data, 24*time.Hour)
        }
        return resp, err
    }
}
```

🐍 **Python — Redis-based idempotency**:

```python
# middleware/idempotency.py
import hashlib
import json
from fastapi import Request, HTTPException
from redis.asyncio import Redis


class IdempotencyMiddleware:
    def __init__(self, redis: Redis, ttl_seconds: int = 86400):
        self.redis = redis
        self.ttl = ttl_seconds

    async def check_and_set(self, request: Request) -> dict | None:
        """Returns cached response if duplicate, None if new request."""
        body = await request.body()
        key = hashlib.sha256(
            f"{request.method}:{request.url.path}:{body}".encode()
        ).hexdigest()

        # Try to claim this request
        was_set = await self.redis.set(
            f"idempotency:{key}", "processing", nx=True, ex=self.ttl
        )

        if not was_set:
            # Check for cached result
            cached = await self.redis.get(f"idempotency:result:{key}")
            if cached:
                return json.loads(cached)
            raise HTTPException(status_code=409, detail="Duplicate request in progress")

        return None  # Proceed with request

    async def cache_result(self, request: Request, response: dict):
        body = await request.body()
        key = hashlib.sha256(
            f"{request.method}:{request.url.path}:{body}".encode()
        ).hexdigest()
        await self.redis.set(
            f"idempotency:result:{key}", json.dumps(response), ex=self.ttl
        )
```

### NATS: Lightweight Event Backbone

We chose **NATS** over Kafka for internal messaging because:

| Concern | NATS | Kafka |
|---------|------|-------|
| **Operational complexity** | Single binary, zero config | ZooKeeper/KRaft, topic management |
| **Latency** | Sub-millisecond | Low-millisecond |
| **Use case fit** | Control plane events (low volume, high value) | Data plane streaming (high volume) |
| **JetStream persistence** | ✅ Built-in | ✅ Native |
| **Memory footprint** | ~20 MB | ~1 GB+ |
| **Learning curve** | Simple pub/sub | Partitions, consumer groups, offsets |

> 💡 **When to use Kafka instead**: If your platform eventually needs to process millions of telemetry events per second, Kafka is the right tool. But for a control plane that handles hundreds to thousands of infrastructure operations per minute, NATS is dramatically simpler to operate.

> ☕ *Java context*: If your team is already running Kafka and has operational expertise, using Kafka for the control plane events is perfectly valid. The event schemas and patterns remain the same — only the transport changes.

🐍 **Python — Publishing audit events to NATS**:

```python
# events/publisher.py
import nats
import json
from datetime import datetime, timezone


class EventPublisher:
    def __init__(self, nats_client: nats.NATS):
        self.nc = nats_client

    async def publish_audit_event(
        self,
        tenant_id: str,
        action: str,
        actor: str,
        resource_type: str,
        resource_id: str,
        outcome: str,
        details: dict | None = None,
    ):
        event = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "tenant_id": tenant_id,
            "action": action,
            "actor": actor,
            "resource_type": resource_type,
            "resource_id": resource_id,
            "outcome": outcome,
            "details": details or {},
        }

        subject = f"audit.{tenant_id}.{resource_type}.{action}"
        await self.nc.publish(subject, json.dumps(event).encode())
```

🔵 **Go — Publishing audit events to NATS**:

```go
// internal/events/publisher.go
package events

import (
    "encoding/json"
    "fmt"
    "time"

    "github.com/nats-io/nats.go"
)

type AuditEvent struct {
    Timestamp    time.Time         `json:"timestamp"`
    TenantID     string            `json:"tenant_id"`
    Action       string            `json:"action"`
    Actor        string            `json:"actor"`
    ResourceType string            `json:"resource_type"`
    ResourceID   string            `json:"resource_id"`
    Outcome      string            `json:"outcome"`
    Details      map[string]any    `json:"details,omitempty"`
}

type Publisher struct {
    nc *nats.Conn
}

func NewPublisher(nc *nats.Conn) *Publisher {
    return &Publisher{nc: nc}
}

func (p *Publisher) PublishAuditEvent(event AuditEvent) error {
    event.Timestamp = time.Now().UTC()
    data, err := json.Marshal(event)
    if err != nil {
        return fmt.Errorf("marshal audit event: %w", err)
    }

    subject := fmt.Sprintf("audit.%s.%s.%s", event.TenantID, event.ResourceType, event.Action)
    return p.nc.Publish(subject, data)
}
```

---

## 2.6 Secrets Management: HashiCorp Vault

### Why Not Just Environment Variables?

Environment variables and Kubernetes Secrets have two critical problems for an IoT platform:

1. **No rotation without restart** — Changing a cloud credential means redeploying.
2. **No audit trail** — You can't answer "who accessed the AWS key at 3 AM?"

Vault solves both while keeping us provider-agnostic:

| Capability | Env Vars | K8s Secrets | Vault |
|-----------|----------|-------------|-------|
| Dynamic secrets | ❌ | ❌ | ✅ |
| Auto-rotation | ❌ | ❌ | ✅ |
| Audit logging | ❌ | API audit | ✅ Full |
| Encryption at rest | OS-dependent | ✅ (etcd) | ✅ (Shamir/Auto-unseal) |
| Multi-tenant isolation | ❌ | Namespace-based | ✅ Namespaces + policies |
| PKI engine | ❌ | ❌ | ✅ Built-in CA |

> Vault's PKI engine is particularly powerful for our use case — it can act as a certificate authority for device certificates, with automatic expiration and revocation. Detailed coverage in **Chapter 6**.

---

## 2.7 Observability: OpenTelemetry + Prometheus + Grafana

### The Three Pillars

```
                    Observability
                   ╱      │      ╲
              Metrics   Traces    Logs
                │         │        │
           Prometheus  Jaeger    Loki
                ╲         │       ╱
                 ╲        │      ╱
                  ╲       │     ╱
                   ───────┼──────
                        Grafana
                     (Unified UI)
```

**OpenTelemetry** is the instrumentation layer — a vendor-neutral SDK that collects metrics, traces, and logs from your application and exports them to whatever backend you choose.

| Pillar | Tool | What It Answers |
|--------|------|----------------|
| **Metrics** | Prometheus | "How many devices were provisioned in the last hour?" |
| **Traces** | Jaeger (via OTLP) | "Why did this provisioning request take 12 seconds?" |
| **Logs** | Loki | "What happened during the failed certificate rotation?" |
| **Dashboards** | Grafana | Unified view across all three pillars |

🐍 **Python — OpenTelemetry integration with FastAPI**:

```python
# observability/setup.py
from opentelemetry import trace, metrics
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor


def setup_observability(app, service_name: str = "iot-platform"):
    resource = Resource.create({"service.name": service_name})

    # Traces
    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317"))
    )
    trace.set_tracer_provider(tracer_provider)

    # Metrics
    meter_provider = MeterProvider(
        resource=resource,
        metric_readers=[PeriodicExportingMetricReader(
            OTLPMetricExporter(endpoint="otel-collector:4317")
        )],
    )
    metrics.set_meter_provider(meter_provider)

    # Auto-instrument FastAPI
    FastAPIInstrumentor.instrument_app(app)
```

> Full observability setup, custom dashboards, and SRE alerting rules are covered in **Chapters 14 and 15**.

---

## 2.8 Deployment: Docker + Kubernetes

### Why Kubernetes?

For a platform that manages infrastructure across cloud providers, Kubernetes provides:

1. **Provider neutrality** — Runs on AWS EKS, Azure AKS, GCP GKE, or bare metal.
2. **Service orchestration** — Automatic restarts, scaling, rolling updates.
3. **Secret mounting** — First-class Vault integration via CSI driver.
4. **Network policies** — Service-to-service access control.

> ☕ *If you're in a Java/.NET shop*: Kubernetes replaces your application server (Tomcat, IIS) and deployment orchestration (Octopus, Azure DevOps release pipelines). The mental model shifts from "deploy a WAR/DLL" to "deploy a container image."

### Docker Images: Small and Secure

🔵 **Go — Multi-stage Dockerfile**:

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /iot-platform ./cmd/server

# Runtime stage — distroless for security
FROM gcr.io/distroless/static-debian12
COPY --from=builder /iot-platform /iot-platform
EXPOSE 8080 9090
ENTRYPOINT ["/iot-platform"]
```

Final image size: **~12 MB**.

🐍 **Python — Multi-stage Dockerfile**:

```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Runtime stage
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Final image size: **~150 MB** (Python runtime overhead).

> 💡 **Key insight**: The Go image is 12x smaller than the Python image. In a Kubernetes environment with frequent rolling updates and auto-scaling, this translates to faster pod startup and lower container registry costs.

---

## 2.9 CI/CD: GitHub Actions

We use GitHub Actions as the reference CI/CD system, but the pipeline logic is portable to GitLab CI, Jenkins, or any other system.

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: iot_platform_test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
      redis:
        image: redis:7
        ports: ["6379:6379"]
      nats:
        image: nats:2.10
        ports: ["4222:4222"]

    steps:
      - uses: actions/checkout@v4

      # Go tests
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./... -race -coverprofile=coverage.out

      # Python tests
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: |
          pip install -r requirements-dev.txt
          pytest --cov=app --cov-report=xml

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: golangci/golangci-lint-action@v4
      - run: |
          pip install ruff
          ruff check .

  build:
    needs: [test, lint]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## 2.10 Decision Matrix Summary

| Decision | Our Choice | Runner-Up | Why Not Runner-Up |
|----------|-----------|-----------|-------------------|
| API Language | Go + Python | TypeScript (NestJS) | Weaker gRPC story, runtime overhead |
| Database | PostgreSQL | CockroachDB | Operational complexity for control plane scale |
| Cache | Redis | Memcached | Need data structures (sorted sets, streams) |
| Events | NATS | Kafka | Over-engineered for control plane volume |
| Secrets | Vault | AWS Secrets Manager | Provider lock-in |
| PKI | CFSSL/step-ca | Vault PKI | Depends on use case (Ch. 6 covers both) |
| Auth policy | OPA | Casbin | OPA is more widely adopted, Rego is powerful |
| Containers | Kubernetes | Docker Compose | Need multi-node, auto-healing in production |
| CI/CD | GitHub Actions | GitLab CI | Both excellent; GH more widely accessible |

---

## 2.11 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Dual language** | Go for performance infrastructure, Python for rapid development |
| **PostgreSQL** | RLS provides database-level tenant isolation |
| **Redis** | Four roles: cache, rate limiter, idempotency, distributed locks |
| **NATS** | Lightweight event backbone — Kafka is overkill for control plane |
| **Vault** | Dynamic secrets, audit trail, and built-in PKI |
| **OpenTelemetry** | Vendor-neutral observability — change backends without re-instrumenting |
| **Kubernetes** | Provider-neutral deployment — matches our multi-cloud philosophy |

### What's Next

In **Chapter 3**, we'll set up a complete local development environment with Docker Compose, so you can run the entire platform on your laptop.
