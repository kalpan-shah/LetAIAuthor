# Chapter 1: Platform Overview & System Architecture

> *"The best architectures are discovered, not invented."* — Martin Fowler

---

## 1.1 What Problem Are We Solving?

The Internet of Things has an infrastructure problem. As organizations scale from a handful of connected devices to hundreds of thousands, they encounter a repeating pattern of pain:

- **Provider lock-in** — AWS IoT Core, Azure IoT Hub, and self-hosted brokers each have entirely different APIs, security models, and provisioning workflows.
- **Certificate chaos** — Device identity management becomes a full-time job. Certificates expire, get revoked, and need rotation across fleets.
- **Tenant isolation** — SaaS IoT platforms must guarantee that Tenant A's devices never see Tenant B's data.
- **Operational blindness** — Infrastructure changes happen across cloud boundaries with no unified audit trail.

The **Unified IoT Orchestration Platform** solves these by acting as a **control plane** — a single pane of glass for provisioning, securing, and managing IoT devices across multiple providers.

### What Is a Control Plane?

> 💡 **For Beginners**: Think of a control plane as an air traffic control tower. The planes (devices) fly on their own, but the tower (our platform) decides who can take off, which runway to use, and what altitude to fly at. The tower doesn't carry passengers — it orchestrates.

In networking, a **control plane** handles configuration, routing decisions, and policy enforcement, while a **data plane** handles the actual traffic. Our platform follows this exact model:

| Concern | Who Handles It |
|---------|---------------|
| Device provisioning | ✅ Our platform |
| Certificate generation | ✅ Our platform |
| Topic/policy creation | ✅ Our platform |
| Telemetry data flow | ❌ The IoT provider (AWS/Azure/Mosquitto) |
| Device-to-cloud messaging | ❌ Direct to provider |
| Real-time alerting on data | ❌ Provider's rules engine |

This separation is critical: **we never touch telemetry data**. This simplifies compliance (no PII transit through our systems), reduces latency (no extra hop), and keeps our platform stateless relative to device data.

---

## 1.2 System Architecture — The 30,000-Foot View

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER / OPERATOR                         │
│              (Web Console, CLI, SDK, REST/gRPC API)                 │
└─────────────────┬──────────────────────┬────────────────────────────┘
                  │                      │
                  ▼                      ▼
┌─────────────────────────┐  ┌──────────────────────────┐
│    API Gateway Layer     │  │   Authentication &       │
│  (REST + gRPC)           │  │   Authorization (RBAC)   │
└──────────┬──────────────┘  └───────────┬──────────────┘
           │                             │
           ▼                             ▼
┌──────────────────────────────────────────────────────────────┐
│                  CORE SERVICE MESH                            │
│                                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │ Tenant   │ │ Device   │ │ Cert     │ │ Policy &      │  │
│  │ Service  │ │ Lifecycle│ │ Lifecycle│ │ Namespace Svc │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬────────┘  │
│       │             │            │               │           │
│  ┌────┴─────┐ ┌─────┴────┐ ┌────┴──────┐ ┌─────┴────────┐  │
│  │Provision │ │Telemetry │ │ Audit     │ │ Quota        │  │
│  │Workflow  │ │Routing   │ │ Service   │ │ Service      │  │
│  │Service   │ │Config Svc│ │           │ │              │  │
│  └──────────┘ └──────────┘ └───────────┘ └──────────────┘  │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│              PROVIDER ABSTRACTION LAYER (PAL)                │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ AWS IoT Core │  │ Azure IoT Hub│  │ Eclipse Mosquitto│   │
│  │ Adapter      │  │ Adapter      │  │ Adapter          │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└──────────────────────────────────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ AWS IoT  │ │Azure IoT │ │Mosquitto │
    │ Core     │ │ Hub      │ │ Broker   │
    └──────────┘ └──────────┘ └──────────┘
```

### Architectural Layers Explained

**1. API Gateway Layer**
All developer and operator interactions enter through a unified API layer. The platform exposes both REST (for web consoles, simple integrations) and gRPC (for high-performance service-to-service calls, CLI tools, and SDKs).

> ☕ *If you're coming from Spring Boot*: This is analogous to your `@RestController` layer, but with the addition of gRPC endpoints via protocol buffer definitions. In ASP.NET terms, think of it as your Controller + gRPC service layer combined.

**2. Core Service Mesh**
This is the heart of the platform. Each domain concern is a discrete service:

| Service | Responsibility |
|---------|---------------|
| **Tenant Service** | CRUD for tenants, lifecycle states, quota enforcement |
| **Device Lifecycle Service** | Create, suspend, delete, reassign devices |
| **Certificate Lifecycle Service** | Generate, attach, rotate, revoke certificates |
| **Policy & Namespace Service** | Topic structure generation, ACL rule creation |
| **Provisioning Workflow Service** | Multi-step device onboarding orchestration |
| **Telemetry Routing Config Service** | Define routing rules (not execute them) |
| **Audit Service** | Log every infrastructure mutation |
| **Quota Service** | Enforce per-tenant resource limits |

**3. Provider Abstraction Layer (PAL)**
The PAL is the critical decoupling point. Every service above talks to the PAL's unified interface — never directly to AWS/Azure/Mosquitto. Adding a new provider means implementing one adapter, not rewriting every service.

---

## 1.3 Key Architectural Decisions

### Decision 1: Microservices vs. Modular Monolith

We recommend starting with a **modular monolith** and extracting services as scale demands.

**Why not microservices from day one?**

| Factor | Microservices | Modular Monolith |
|--------|--------------|-----------------|
| Team size < 10 | Overhead kills velocity | ✅ Ship fast |
| Distributed transactions | Need saga patterns | In-process calls |
| Debugging | Distributed tracing required | Stack traces work |
| Deployment complexity | Kubernetes + service mesh | Single binary |
| Service extraction | Already done | Extract when needed |

> 💡 **Principal Engineer Note**: The modular monolith approach follows the "monolith-first" strategy popularized by Martin Fowler and Sam Newman. The key is enforcing module boundaries *as if* they were service boundaries (separate packages, defined interfaces, no shared mutable state). This makes future extraction mechanical rather than archaeological.

> ☕ *Java teams*: This is similar to a well-structured Spring Boot application with clear `@Service` boundaries and dependency injection, but without the overhead of Spring Cloud microservices. C# teams can think of it as a single ASP.NET Core application with strict project/assembly boundaries.

**The Module Boundary Rule**: Modules communicate only through defined interfaces. No module reaches into another module's database tables, internal types, or private functions.

### Decision 2: Event-Driven Where It Matters

Not everything needs to be event-driven. We use a pragmatic mix:

```
Synchronous (Request/Response):
  - Tenant CRUD
  - Device creation
  - Certificate generation
  - API queries

Asynchronous (Event-Driven):
  - Audit log writes
  - Quota recalculation
  - Multi-step provisioning workflows
  - Cross-provider state synchronization
```

Events are published to **NATS** (lightweight, cloud-native) rather than Kafka (which would be over-engineered for a control plane with modest throughput).

### Decision 3: Provider Abstraction via the Strategy Pattern

The PAL uses the **Strategy pattern** — each provider implements a common interface, and the runtime selects the correct strategy based on tenant configuration.

🔵 **Go**:

```go
// provider.go — The Provider interface every adapter must implement
type Provider interface {
    // Device operations
    CreateDevice(ctx context.Context, req CreateDeviceRequest) (*Device, error)
    DeleteDevice(ctx context.Context, deviceID string) error
    SuspendDevice(ctx context.Context, deviceID string) error

    // Certificate operations
    CreateCertificate(ctx context.Context, deviceID string) (*Certificate, error)
    RevokeCertificate(ctx context.Context, certID string) error
    RotateCertificate(ctx context.Context, certID string) (*Certificate, error)

    // Policy operations
    AttachPolicy(ctx context.Context, certID string, policy Policy) error
    DetachPolicy(ctx context.Context, certID string, policyName string) error

    // Connectivity
    GetEndpoint(ctx context.Context) (string, error)
    ValidateCredentials(ctx context.Context) error
}
```

🐍 **Python**:

```python
# provider.py — Abstract base class for all provider adapters
from abc import ABC, abstractmethod
from dataclasses import dataclass


@dataclass
class CreateDeviceRequest:
    device_name: str
    tenant_id: str
    metadata: dict | None = None


@dataclass
class Device:
    device_id: str
    provider_device_id: str
    tenant_id: str
    status: str


@dataclass
class Certificate:
    cert_id: str
    device_id: str
    pem: str
    private_key: str
    expiry: str


class IoTProvider(ABC):
    """Every provider adapter implements this interface."""

    @abstractmethod
    async def create_device(self, req: CreateDeviceRequest) -> Device: ...

    @abstractmethod
    async def delete_device(self, device_id: str) -> None: ...

    @abstractmethod
    async def suspend_device(self, device_id: str) -> None: ...

    @abstractmethod
    async def create_certificate(self, device_id: str) -> Certificate: ...

    @abstractmethod
    async def revoke_certificate(self, cert_id: str) -> None: ...

    @abstractmethod
    async def rotate_certificate(self, cert_id: str) -> Certificate: ...

    @abstractmethod
    async def attach_policy(self, cert_id: str, policy: dict) -> None: ...

    @abstractmethod
    async def get_endpoint(self) -> str: ...

    @abstractmethod
    async def validate_credentials(self) -> bool: ...
```

> ☕ *Legacy context*: Java developers will recognize this as the classic `interface + implementation` pattern — identical to defining a Spring `@Service` interface with `@Qualifier`-selected implementations. C# developers: this maps directly to an `IIoTProvider` interface resolved via `IServiceCollection.AddScoped<IIoTProvider, AwsIoTProvider>()`.

---

## 1.4 Data Flow Walkthrough: Provisioning a Device

Let's trace a complete device provisioning request through the system:

```
Developer                  Platform                           AWS IoT Core
   │                          │                                    │
   │  POST /devices           │                                    │
   │  {tenant: "acme",        │                                    │
   │   name: "sensor-42"}     │                                    │
   │─────────────────────────▶│                                    │
   │                          │                                    │
   │                     ┌────┴────┐                               │
   │                     │ Validate │                              │
   │                     │ Tenant   │                              │
   │                     │ & Quota  │                              │
   │                     └────┬────┘                               │
   │                          │                                    │
   │                     ┌────┴────┐                               │
   │                     │ Create  │   CreateThing("sensor-42")    │
   │                     │ Device  │──────────────────────────────▶│
   │                     │ (PAL)   │◀──────────────────────────────│
   │                     └────┬────┘   {thingArn: "arn:..."}      │
   │                          │                                    │
   │                     ┌────┴────┐                               │
   │                     │ Create  │   CreateKeysAndCertificate()   │
   │                     │ Cert    │──────────────────────────────▶│
   │                     │ (PAL)   │◀──────────────────────────────│
   │                     └────┬────┘   {certPem, privateKey}      │
   │                          │                                    │
   │                     ┌────┴────┐                               │
   │                     │ Generate│   AttachPolicy("acme-sensor-42") │
   │                     │ & Attach│──────────────────────────────▶│
   │                     │ Policy  │◀──────────────────────────────│
   │                     └────┬────┘                               │
   │                          │                                    │
   │                     ┌────┴────┐                               │
   │                     │ Save to │                               │
   │                     │ Database│                               │
   │                     └────┬────┘                               │
   │                          │                                    │
   │                     ┌────┴────┐                               │
   │                     │  Emit   │                               │
   │                     │  Audit  │                               │
   │                     │  Event  │                               │
   │                     └────┬────┘                               │
   │                          │                                    │
   │  201 Created             │                                    │
   │  {device_id, cert_pem,   │                                    │
   │   private_key, endpoint} │                                    │
   │◀─────────────────────────│                                    │
```

**Key observations:**

1. **The developer never calls AWS directly.** All provider interactions go through the PAL.
2. **Quota checks happen before any cloud API call** — fail fast, don't waste cloud API quota.
3. **The audit event is asynchronous** — it's published to NATS, not written synchronously. A failed audit write should never block provisioning.
4. **Certificate material is returned once** — the platform stores a reference, not the private key (the private key is generated by the provider and returned to the caller, then discarded).

---

## 1.5 Tenant Isolation Model

Multi-tenancy is the hardest architectural concern to bolt on later. We bake it in from day one.

### Isolation Layers

```
┌───────────────────────────────────────────────────┐
│ Layer 4: Topic Namespace Isolation                │
│  tenant/{id}/device/{id}/telemetry                │
│  Enforced by provider ACL / IoT policies          │
├───────────────────────────────────────────────────┤
│ Layer 3: Certificate Isolation                    │
│  Each device cert only permits access to          │
│  its tenant's topic namespace                     │
├───────────────────────────────────────────────────┤
│ Layer 2: Database Row-Level Isolation             │
│  Every table has tenant_id column                 │
│  All queries include WHERE tenant_id = ?          │
├───────────────────────────────────────────────────┤
│ Layer 1: API Authentication & RBAC                │
│  API keys / JWT tokens scoped to tenant           │
│  RBAC enforces tenant boundaries                  │
└───────────────────────────────────────────────────┘
```

> ⚠️ **Critical Design Rule**: The `tenant_id` is **never optional**. Every database query, every provider API call, every audit log entry includes the tenant context. This is enforced at the middleware/interceptor level, not left to individual service implementations.

🔵 **Go — Tenant Middleware**:

```go
// middleware/tenant.go
func TenantInterceptor() grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        tenantID, err := extractTenantID(ctx)
        if err != nil {
            return nil, status.Error(codes.Unauthenticated, "missing tenant context")
        }

        // Inject tenant ID into context — all downstream code reads from here
        ctx = context.WithValue(ctx, tenantKey, tenantID)
        return handler(ctx, req)
    }
}
```

🐍 **Python — Tenant Middleware**:

```python
# middleware/tenant.py
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware


class TenantMiddleware(BaseHTTPMiddleware):
    """Extract and enforce tenant context on every request."""

    async def dispatch(self, request: Request, call_next):
        tenant_id = request.headers.get("X-Tenant-ID")

        if not tenant_id:
            raise HTTPException(status_code=401, detail="Missing tenant context")

        # Store in request state — all downstream handlers read from here
        request.state.tenant_id = tenant_id
        response = await call_next(request)
        return response
```

---

## 1.6 Security Posture Overview

Security isn't a chapter — it's a cross-cutting concern woven through every layer. Here's the security surface at a glance:

| Surface | Threat | Mitigation |
|---------|--------|-----------|
| API endpoints | Unauthorized access | JWT + RBAC, rate limiting |
| Cloud credentials | Credential theft | Vault with short-lived leases |
| Certificates | Unauthorized device access | Auto-rotation, revocation lists |
| Database | Data leakage across tenants | Row-level isolation, tenant middleware |
| Audit logs | Tampering | Append-only storage, integrity hashing |
| Provider APIs | Overprovisioning | Quota enforcement, rate limiting |

> Detailed threat modeling and security architecture are covered in **Chapter 16**.

---

## 1.7 What This Platform Is *Not*

To prevent scope creep, let's be explicit:

| This Platform... | Does NOT... |
|-----------------|-------------|
| Provisions devices | Process telemetry data |
| Manages certificates | Run as a message broker |
| Defines topic namespaces | Route messages at runtime |
| Configures routing rules | Execute routing rules |
| Generates policies | Serve as an identity provider |
| Audits infrastructure actions | Monitor device health metrics |

The platform is a **control plane**, not a data plane. Devices communicate directly with their IoT provider after provisioning. Our platform manages the *infrastructure* that makes that communication possible.

---

## 1.8 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Control Plane** | We manage infrastructure, not telemetry data |
| **Provider Abstraction** | Strategy pattern decouples services from cloud providers |
| **Modular Monolith** | Start unified, extract services when needed |
| **Tenant Isolation** | Four-layer isolation baked in from day one |
| **Event-Driven (selective)** | Sync for CRUD, async for auditing and workflows |
| **Dual API** | REST for humans, gRPC for machines |

### What's Next

In **Chapter 2**, we'll deep-dive into the technology stack selection — why Go + Python, PostgreSQL, NATS, Vault, and Kubernetes form the ideal foundation for this platform, and how they compare to alternatives you might be considering.
