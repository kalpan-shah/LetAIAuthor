# Chapter 7: Tenant Management Service

> *"Multi-tenancy is not a feature — it's an architectural decision that pervades every layer of your system."*

---

## 7.1 Introduction

The Tenant Management Service is the **root entity** of the entire platform. Every device, certificate, policy, and audit log belongs to a tenant. Nothing exists without tenant context.

This chapter builds the complete tenant service — API endpoints, business logic, lifecycle state management, and quota enforcement.

> 💡 **For Beginners**: A tenant is like an apartment in a building. Each tenant has their own space (data), locked door (isolation), and lease terms (quotas). The building manager (our platform) handles shared infrastructure (plumbing, electricity) but never enters without permission.

---

## 7.2 Tenant Service API

### Protobuf Definition (for Go gRPC)

```protobuf
// api/proto/v1/tenant.proto
syntax = "proto3";
package iot.platform.v1;

option go_package = "iot-platform/api/proto/v1;pb";

service TenantService {
    rpc CreateTenant(CreateTenantRequest) returns (Tenant);
    rpc GetTenant(GetTenantRequest) returns (Tenant);
    rpc ListTenants(ListTenantsRequest) returns (ListTenantsResponse);
    rpc UpdateTenant(UpdateTenantRequest) returns (Tenant);
    rpc SuspendTenant(SuspendTenantRequest) returns (Tenant);
    rpc ReactivateTenant(ReactivateTenantRequest) returns (Tenant);
    rpc ArchiveTenant(ArchiveTenantRequest) returns (Tenant);
    rpc GetTenantQuota(GetTenantQuotaRequest) returns (TenantQuota);
}

message CreateTenantRequest {
    string name = 1;
    string slug = 2;              // URL-friendly identifier
    string provider = 3;          // "aws", "azure", "mosquitto"
    string cloud_account = 4;     // Provider-specific account ref
    string region = 5;
    TenantQuotaConfig quota = 6;
}

message Tenant {
    string id = 1;
    string name = 2;
    string slug = 3;
    string provider = 4;
    string cloud_account = 5;
    string region = 6;
    string status = 7;            // "active", "suspended", "archived"
    TenantQuotaConfig quota = 8;
    string created_at = 9;
    string updated_at = 10;
}

message TenantQuotaConfig {
    int32 max_devices = 1;
    int32 max_routing_rules = 2;
    int32 max_cert_issuance = 3;
}

message TenantQuota {
    string tenant_id = 1;
    TenantQuotaConfig limits = 2;
    TenantQuotaUsage usage = 3;
}

message TenantQuotaUsage {
    int32 devices = 1;
    int32 routing_rules = 2;
    int32 certs_issued = 3;
}

message GetTenantRequest {
    string tenant_id = 1;
}

message ListTenantsRequest {
    int32 page_size = 1;
    string page_token = 2;
    string status_filter = 3;
}

message ListTenantsResponse {
    repeated Tenant tenants = 1;
    string next_page_token = 2;
    int32 total_count = 3;
}

message UpdateTenantRequest {
    string tenant_id = 1;
    string name = 2;
    TenantQuotaConfig quota = 3;
}

message SuspendTenantRequest { string tenant_id = 1; }
message ReactivateTenantRequest { string tenant_id = 1; }
message ArchiveTenantRequest { string tenant_id = 1; }
message GetTenantQuotaRequest { string tenant_id = 1; }
```

### FastAPI Routes (Python)

```python
# app/routers/tenants.py
from fastapi import APIRouter, Depends, HTTPException, Query, Request
from pydantic import BaseModel, Field
from uuid import UUID

router = APIRouter()


class CreateTenantRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    slug: str = Field(..., pattern=r"^[a-z0-9][a-z0-9-]{1,61}[a-z0-9]$")
    provider: str = Field(..., pattern=r"^(aws|azure|mosquitto)$")
    cloud_account: str | None = None
    region: str | None = None
    max_devices: int = Field(default=1000, ge=1, le=100000)
    max_routing_rules: int = Field(default=50, ge=1, le=1000)
    max_cert_issuance: int = Field(default=5000, ge=1, le=100000)


class TenantResponse(BaseModel):
    id: UUID
    name: str
    slug: str
    provider: str
    cloud_account: str | None
    region: str | None
    status: str
    max_devices: int
    max_routing_rules: int
    max_cert_issuance: int
    created_at: str
    updated_at: str


@router.post("/", response_model=TenantResponse, status_code=201)
async def create_tenant(body: CreateTenantRequest, request: Request):
    """Create a new tenant."""
    service = request.app.state.tenant_service
    tenant = await service.create_tenant(body)
    return tenant


@router.get("/{tenant_id}", response_model=TenantResponse)
async def get_tenant(tenant_id: UUID, request: Request):
    """Get a tenant by ID."""
    service = request.app.state.tenant_service
    tenant = await service.get_tenant(str(tenant_id))
    if not tenant:
        raise HTTPException(status_code=404, detail="Tenant not found")
    return tenant


@router.get("/", response_model=list[TenantResponse])
async def list_tenants(
    request: Request,
    status: str | None = Query(None, pattern=r"^(active|suspended|archived)$"),
    limit: int = Query(20, ge=1, le=100),
    offset: int = Query(0, ge=0),
):
    """List tenants with optional status filter."""
    service = request.app.state.tenant_service
    return await service.list_tenants(status=status, limit=limit, offset=offset)


@router.patch("/{tenant_id}", response_model=TenantResponse)
async def update_tenant(tenant_id: UUID, body: dict, request: Request):
    """Update tenant properties."""
    service = request.app.state.tenant_service
    return await service.update_tenant(str(tenant_id), body)


@router.post("/{tenant_id}/suspend", response_model=TenantResponse)
async def suspend_tenant(tenant_id: UUID, request: Request):
    """Suspend a tenant — all devices become inactive."""
    service = request.app.state.tenant_service
    return await service.transition_status(str(tenant_id), "suspended")


@router.post("/{tenant_id}/reactivate", response_model=TenantResponse)
async def reactivate_tenant(tenant_id: UUID, request: Request):
    """Reactivate a suspended tenant."""
    service = request.app.state.tenant_service
    return await service.transition_status(str(tenant_id), "active")


@router.post("/{tenant_id}/archive", response_model=TenantResponse)
async def archive_tenant(tenant_id: UUID, request: Request):
    """Archive a tenant — irreversible, triggers cleanup."""
    service = request.app.state.tenant_service
    return await service.transition_status(str(tenant_id), "archived")
```

---

## 7.3 Tenant Service Implementation

🐍 **Python — Core business logic**:

```python
# app/services/tenant_service.py
import json
from asyncpg import Pool, UniqueViolationError

from app.vault.client import VaultClient
from app.events.publisher import EventPublisher


# Valid state transitions
TENANT_STATE_MACHINE = {
    "active":    {"suspended", "archived"},
    "suspended": {"active", "archived"},
    "archived":  set(),  # Terminal state — no transitions out
}


class TenantService:
    def __init__(self, db: Pool, vault: VaultClient, events: EventPublisher):
        self.db = db
        self.vault = vault
        self.events = events

    async def create_tenant(self, req) -> dict:
        async with self.db.acquire() as conn:
            async with conn.transaction():
                try:
                    row = await conn.fetchrow(
                        """INSERT INTO tenants
                           (name, slug, provider, cloud_account, region,
                            max_devices, max_routing_rules, max_cert_issuance)
                           VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
                           RETURNING *""",
                        req.name, req.slug, req.provider, req.cloud_account,
                        req.region, req.max_devices, req.max_routing_rules,
                        req.max_cert_issuance,
                    )
                except UniqueViolationError:
                    raise TenantAlreadyExistsError(f"Tenant slug '{req.slug}' already exists")

                tenant = dict(row)

        # Emit creation event
        await self.events.publish_audit_event(
            tenant_id=str(tenant["id"]),
            action="tenant.created",
            actor="api",
            resource_type="tenant",
            resource_id=str(tenant["id"]),
            outcome="success",
        )

        return tenant

    async def get_tenant(self, tenant_id: str) -> dict | None:
        async with self.db.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT * FROM tenants WHERE id = $1", tenant_id
            )
            return dict(row) if row else None

    async def list_tenants(
        self, status: str | None = None, limit: int = 20, offset: int = 0
    ) -> list[dict]:
        async with self.db.acquire() as conn:
            if status:
                rows = await conn.fetch(
                    """SELECT * FROM tenants WHERE status = $1::tenant_status
                       ORDER BY created_at DESC LIMIT $2 OFFSET $3""",
                    status, limit, offset,
                )
            else:
                rows = await conn.fetch(
                    "SELECT * FROM tenants ORDER BY created_at DESC LIMIT $1 OFFSET $2",
                    limit, offset,
                )
            return [dict(r) for r in rows]

    async def transition_status(self, tenant_id: str, target_status: str) -> dict:
        """Transition tenant to a new lifecycle state."""
        async with self.db.acquire() as conn:
            async with conn.transaction():
                # Get current state
                row = await conn.fetchrow(
                    "SELECT * FROM tenants WHERE id = $1 FOR UPDATE", tenant_id
                )
                if not row:
                    raise TenantNotFoundError(tenant_id)

                current_status = row["status"]

                # Validate transition
                valid_targets = TENANT_STATE_MACHINE.get(current_status, set())
                if target_status not in valid_targets:
                    raise InvalidStateTransitionError(
                        f"Cannot transition from '{current_status}' to '{target_status}'. "
                        f"Valid targets: {valid_targets}"
                    )

                # Apply transition
                updated = await conn.fetchrow(
                    """UPDATE tenants SET status = $1::tenant_status
                       WHERE id = $2 RETURNING *""",
                    target_status, tenant_id,
                )

                # Side effects for specific transitions
                if target_status == "suspended":
                    await self._suspend_all_devices(conn, tenant_id)
                elif target_status == "archived":
                    await self._schedule_cleanup(conn, tenant_id)

                tenant = dict(updated)

        await self.events.publish_audit_event(
            tenant_id=tenant_id,
            action=f"tenant.{target_status}",
            actor="api",
            resource_type="tenant",
            resource_id=tenant_id,
            outcome="success",
            details={"from_status": current_status},
        )

        return tenant

    async def _suspend_all_devices(self, conn, tenant_id: str):
        """When a tenant is suspended, suspend all their devices."""
        await conn.execute(
            """UPDATE devices SET status = 'suspended'
               WHERE tenant_id = $1 AND status = 'active'""",
            tenant_id,
        )

    async def _schedule_cleanup(self, conn, tenant_id: str):
        """Schedule async cleanup when tenant is archived."""
        # This publishes an event that triggers:
        # 1. Provider-side device cleanup
        # 2. Certificate revocation
        # 3. Vault secret deletion
        # Handled by background workers (Chapter 11)
        pass
```

🔵 **Go — Tenant lifecycle state machine**:

```go
// internal/tenant/state.go
package tenant

import "fmt"

// validTransitions defines the tenant state machine.
var validTransitions = map[string]map[string]bool{
    "active":    {"suspended": true, "archived": true},
    "suspended": {"active": true, "archived": true},
    "archived":  {},  // Terminal state
}

// ValidateTransition checks if a state transition is allowed.
func ValidateTransition(from, to string) error {
    targets, exists := validTransitions[from]
    if !exists {
        return fmt.Errorf("unknown status: %s", from)
    }
    if !targets[to] {
        return fmt.Errorf("cannot transition from '%s' to '%s'", from, to)
    }
    return nil
}
```

---

## 7.4 Tenant Lifecycle State Machine

```
                ┌──────────┐
                │          │
    ┌──────────▶│  active  │◀──────────┐
    │           │          │           │
    │           └────┬─────┘           │
    │                │                 │
    │         suspend│          reactivate
    │                │                 │
    │                ▼                 │
    │           ┌──────────┐           │
    │           │          │───────────┘
    │           │suspended │
    │           │          │
    │           └────┬─────┘
    │                │
    │                │ archive
    │                │
    │                ▼
    │           ┌──────────┐
    │  archive  │          │
    └──────────▶│ archived │  (Terminal — no way out)
                │          │
                └──────────┘
```

| Transition | Side Effects |
|-----------|-------------|
| active → suspended | All devices suspended, certs deactivated |
| suspended → active | Devices reactivated, certs re-enabled |
| * → archived | Cleanup job: revoke certs, delete provider resources, remove Vault secrets |

---

## 7.5 Quota Enforcement

```python
# app/services/quota_service.py

class QuotaService:
    def __init__(self, db: Pool, redis):
        self.db = db
        self.redis = redis

    async def check_quota(self, tenant_id: str, resource_type: str) -> bool:
        """Check if tenant has capacity for a new resource."""
        quota = await self.get_quota(tenant_id)

        limits = {
            "device": ("max_devices", "device_count"),
            "routing_rule": ("max_routing_rules", "routing_rule_count"),
            "certificate": ("max_cert_issuance", "cert_count"),
        }

        limit_key, usage_key = limits.get(resource_type, (None, None))
        if not limit_key:
            raise ValueError(f"Unknown resource type: {resource_type}")

        return quota["usage"][usage_key] < quota["limits"][limit_key]

    async def get_quota(self, tenant_id: str) -> dict:
        """Get current quota limits and usage for a tenant."""
        # Try cache first (1-minute TTL)
        cached = await self.redis.get(f"quota:{tenant_id}")
        if cached:
            return json.loads(cached)

        async with self.db.acquire() as conn:
            tenant = await conn.fetchrow(
                "SELECT max_devices, max_routing_rules, max_cert_issuance FROM tenants WHERE id = $1",
                tenant_id,
            )

            device_count = await conn.fetchval(
                "SELECT COUNT(*) FROM devices WHERE tenant_id = $1 AND status != 'deleted'",
                tenant_id,
            )
            rule_count = await conn.fetchval(
                "SELECT COUNT(*) FROM routing_rules WHERE tenant_id = $1",
                tenant_id,
            )
            cert_count = await conn.fetchval(
                "SELECT COUNT(*) FROM certificates WHERE tenant_id = $1",
                tenant_id,
            )

        quota = {
            "tenant_id": tenant_id,
            "limits": {
                "max_devices": tenant["max_devices"],
                "max_routing_rules": tenant["max_routing_rules"],
                "max_cert_issuance": tenant["max_cert_issuance"],
            },
            "usage": {
                "device_count": device_count,
                "routing_rule_count": rule_count,
                "cert_count": cert_count,
            },
        }

        # Cache for 1 minute
        await self.redis.set(f"quota:{tenant_id}", json.dumps(quota), ex=60)
        return quota
```

---

## 7.6 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Tenant as root entity** | Everything in the platform is scoped to a tenant |
| **State machine** | Explicit valid transitions prevent invalid lifecycle states |
| **Side effects** | Suspension cascades to devices; archival triggers cleanup |
| **Quota enforcement** | Redis-cached usage counts, checked before resource creation |
| **Slug-based identity** | URL-friendly, unique identifiers for API paths |

### What's Next

**Chapter 8** builds the **Device Lifecycle Service** — creating, suspending, deleting, and reassigning IoT devices across providers.
