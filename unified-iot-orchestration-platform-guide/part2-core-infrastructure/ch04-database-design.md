# Chapter 4: Database Design & State Management

> *"The database is the heart of your system. Get the schema wrong, and every layer above it suffers."*

---

## 4.1 Introduction

The IoT Orchestration Platform is a **stateful control plane**. Every tenant, device, certificate, policy, and audit record is persisted, queried, and updated through PostgreSQL. This chapter covers the complete database design — from entity relationships and indexing strategies to row-level security, migration tooling, and connection management.

> 💡 **For Beginners**: A database schema is like a blueprint for data. Just as an architect draws floor plans before building a house, we design our tables, columns, and relationships before writing application code. This prevents costly restructuring later.

---

## 4.2 Entity Relationship Model

```
┌──────────────────┐       ┌──────────────────────┐
│     tenants      │       │   cloud_integrations  │
│──────────────────│       │──────────────────────│
│ id (PK)          │──┐    │ id (PK)              │
│ name             │  │    │ tenant_id (FK)────────│──┐
│ slug (UNIQUE)    │  │    │ provider              │  │
│ status           │  │    │ vault_secret_path     │  │
│ max_devices      │  │    │ endpoint              │  │
│ created_at       │  │    │ validated_at          │  │
│ updated_at       │  │    │ status                │  │
└─────────┬────────┘  │    └──────────────────────┘  │
          │           │                               │
          │           │    ┌──────────────────────┐   │
          │           └───▶│      devices          │   │
          │                │──────────────────────│   │
          │                │ id (PK)              │   │
          └───────────────▶│ tenant_id (FK)       │◀──┘
                           │ integration_id (FK)  │
                           │ name                 │
                           │ provider_device_id   │
                           │ status               │
                           │ active_cert_id       │
                           │ metadata (JSONB)     │
                           │ created_at           │
                           │ updated_at           │
                           └─────────┬────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
                    ▼                ▼                 ▼
          ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
          │ certificates │  │  policies     │  │  audit_logs      │
          │──────────────│  │──────────────│  │──────────────────│
          │ id (PK)      │  │ id (PK)      │  │ id (PK)          │
          │ device_id(FK)│  │ device_id(FK)│  │ tenant_id (FK)   │
          │ tenant_id(FK)│  │ tenant_id(FK)│  │ action           │
          │ fingerprint  │  │ policy_doc   │  │ actor            │
          │ cert_pem     │  │ provider_id  │  │ resource_type    │
          │ status       │  │ attached_at  │  │ resource_id      │
          │ expires_at   │  │ created_at   │  │ outcome          │
          │ revoked_at   │  │              │  │ details (JSONB)  │
          └──────────────┘  └──────────────┘  │ created_at       │
                                              └──────────────────┘
```

### Core Entities

| Entity | Purpose | Key Relationships |
|--------|---------|-------------------|
| **tenants** | Customer/org unit registration | Root entity — everything belongs to a tenant |
| **cloud_integrations** | Provider connection config per tenant | References Vault path, no secrets stored |
| **devices** | IoT device registration and state | Belongs to tenant, linked to integration |
| **certificates** | X.509 cert lifecycle tracking | Belongs to device +tenant |
| **policies** | Access control policies for devices | Attached to devices, provider-specific |
| **audit_logs** | Immutable event log | Append-only, references tenant |

---

## 4.3 Complete Schema

### Cloud Integrations Table

This table was introduced in Chapter 3's migrations as part of the tenant model. Here's the full definition:

```sql
-- migrations/004_create_cloud_integrations.sql

CREATE TYPE integration_status AS ENUM ('active', 'validating', 'failed', 'revoked');

CREATE TABLE cloud_integrations (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider            VARCHAR(50) NOT NULL,       -- 'aws', 'azure', 'mosquitto'
    vault_secret_path   VARCHAR(500) NOT NULL,      -- Path in Vault to credentials

    -- Cached from provider (refreshed periodically)
    endpoint            VARCHAR(500),
    endpoint_updated_at TIMESTAMPTZ,

    -- Status tracking
    status              integration_status NOT NULL DEFAULT 'validating',
    last_validated_at   TIMESTAMPTZ,
    last_error          TEXT,

    -- Timestamps
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- One integration per provider per tenant
    UNIQUE(tenant_id, provider)
);

ALTER TABLE cloud_integrations ENABLE ROW LEVEL SECURITY;

CREATE POLICY ci_tenant_isolation ON cloud_integrations
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

CREATE INDEX idx_ci_tenant ON cloud_integrations (tenant_id);
CREATE INDEX idx_ci_status ON cloud_integrations (status);

CREATE TRIGGER cloud_integrations_updated_at
    BEFORE UPDATE ON cloud_integrations
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

> ⚠️ **Critical**: The `vault_secret_path` is a *reference* to where the actual credentials live in Vault — never the credentials themselves. This means even a full database dump exposes zero cloud credentials.

### Policies Table

```sql
-- migrations/005_create_policies.sql

CREATE TABLE policies (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    device_id           UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,

    -- Policy content
    policy_name         VARCHAR(255) NOT NULL,
    policy_document     JSONB NOT NULL,             -- Provider-agnostic policy
    provider_policy_id  VARCHAR(255),               -- Provider's reference after attachment

    -- Provider translation
    provider_format     JSONB,                      -- Provider-specific translated policy

    -- Status
    attached_at         TIMESTAMPTZ,
    detached_at         TIMESTAMPTZ,

    -- Timestamps
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE policies ENABLE ROW LEVEL SECURITY;

CREATE POLICY policies_tenant_isolation ON policies
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

CREATE INDEX idx_policies_device ON policies (device_id);
CREATE INDEX idx_policies_tenant ON policies (tenant_id);

CREATE TRIGGER policies_updated_at
    BEFORE UPDATE ON policies
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

### Audit Logs Table

```sql
-- migrations/006_create_audit_logs.sql

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    action          VARCHAR(100) NOT NULL,      -- 'device.created', 'cert.rotated', etc.
    actor           VARCHAR(255) NOT NULL,      -- API key ID, user email, or 'system'
    resource_type   VARCHAR(50) NOT NULL,       -- 'device', 'certificate', 'tenant', etc.
    resource_id     UUID,
    outcome         VARCHAR(20) NOT NULL,       -- 'success', 'failure', 'partial'
    details         JSONB DEFAULT '{}',         -- Additional context
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- NO RLS on audit_logs — platform operators need cross-tenant audit access
-- Access controlled at application layer via RBAC

-- Partitioning by month for efficient retention management
CREATE INDEX idx_audit_tenant ON audit_logs (tenant_id, created_at DESC);
CREATE INDEX idx_audit_action ON audit_logs (action, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_logs (resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_logs (created_at DESC);

-- For full-text search on audit details
CREATE INDEX idx_audit_details ON audit_logs USING GIN (details);
```

> 💡 **Production tip**: For high-volume audit logs, use [PostgreSQL table partitioning](https://www.postgresql.org/docs/16/ddl-partitioning.html) by month. This allows efficient archival and retention management without affecting query performance on recent data.

### Routing Rules Table

```sql
-- migrations/007_create_routing_rules.sql

CREATE TYPE routing_status AS ENUM ('active', 'disabled', 'draft');

CREATE TABLE routing_rules (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,

    -- Rule definition
    source_topic    VARCHAR(500) NOT NULL,       -- MQTT topic pattern to match
    destination     JSONB NOT NULL,              -- {"type": "kinesis", "stream": "..."}
    filter_sql      TEXT,                        -- Optional SQL-like filter expression
    transform       JSONB,                       -- Optional field transformations

    -- Status
    status          routing_status NOT NULL DEFAULT 'draft',
    provider_rule_id VARCHAR(255),               -- Provider's rule reference after deployment

    -- Timestamps
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(tenant_id, name)
);

ALTER TABLE routing_rules ENABLE ROW LEVEL SECURITY;

CREATE POLICY routing_tenant_isolation ON routing_rules
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

CREATE INDEX idx_routing_tenant ON routing_rules (tenant_id);
CREATE INDEX idx_routing_status ON routing_rules (tenant_id, status);

CREATE TRIGGER routing_rules_updated_at
    BEFORE UPDATE ON routing_rules
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

---

## 4.4 Row-Level Security Deep Dive

RLS is our database-level safety net for tenant isolation. Here's how it works end-to-end:

### Setting the Tenant Context

Before any query, the application sets a session variable that RLS policies reference:

🔵 **Go — Setting tenant context per-request**:

```go
// internal/db/tenant_context.go
package db

import (
    "context"
    "fmt"

    "github.com/jackc/pgx/v5/pgxpool"
)

// WithTenantContext wraps a database operation with tenant isolation.
func WithTenantContext(ctx context.Context, pool *pgxpool.Pool, tenantID string, fn func(ctx context.Context, conn *pgxpool.Conn) error) error {
    conn, err := pool.Acquire(ctx)
    if err != nil {
        return fmt.Errorf("acquire connection: %w", err)
    }
    defer conn.Release()

    // Set the tenant context for RLS
    _, err = conn.Exec(ctx,
        "SET LOCAL app.current_tenant_id = $1", tenantID)
    if err != nil {
        return fmt.Errorf("set tenant context: %w", err)
    }

    return fn(ctx, conn)
}

// Usage:
// db.WithTenantContext(ctx, pool, tenantID, func(ctx context.Context, conn *pgxpool.Conn) error {
//     rows, err := conn.Query(ctx, "SELECT * FROM devices")  // Auto-filtered by tenant!
//     ...
// })
```

🐍 **Python — Setting tenant context per-request**:

```python
# db/tenant_context.py
from contextlib import asynccontextmanager
from asyncpg import Connection, Pool


@asynccontextmanager
async def tenant_context(pool: Pool, tenant_id: str):
    """Acquire a connection with tenant context set for RLS."""
    async with pool.acquire() as conn:  # type: Connection
        # Set tenant context — RLS policies will auto-filter
        await conn.execute(
            "SET LOCAL app.current_tenant_id = $1", tenant_id
        )
        yield conn


# Usage:
# async with tenant_context(pool, tenant_id) as conn:
#     devices = await conn.fetch("SELECT * FROM devices")  # Auto-filtered!
```

### What Happens Under the Hood

```sql
-- Without RLS, this returns ALL devices across ALL tenants:
SELECT * FROM devices;

-- With RLS enabled and app.current_tenant_id = 'acme-uuid':
-- PostgreSQL automatically rewrites the query to:
SELECT * FROM devices WHERE tenant_id = 'acme-uuid';
```

> ☕ *For Java developers using Hibernate*: RLS replaces the common pattern of adding `@Filter(name = "tenantFilter")` on every entity and `@FilterDef` with parameter binding. RLS is superior because it's impossible to forget — even raw SQL queries through `@NativeQuery` are filtered. In C# / EF Core, this replaces global query filters (`modelBuilder.Entity<Device>().HasQueryFilter(d => d.TenantId == currentTenantId)`).

### RLS Bypass for Platform Operations

Some operations (like cross-tenant reporting) need to bypass RLS:

```sql
-- Create a superuser role that bypasses RLS
CREATE ROLE platform_admin BYPASSRLS;

-- Application uses this role ONLY for cross-tenant admin operations
-- Normal request handling ALWAYS uses the RLS-enforced role
```

---

## 4.5 Indexing Strategy

### Principles

1. **Every foreign key gets an index** — PostgreSQL doesn't auto-index FKs.
2. **Composite indexes for common query patterns** — `(tenant_id, status)` not separate indexes.
3. **Partial indexes for active-only queries** — `WHERE status = 'active'` to reduce index size.
4. **GIN indexes for JSONB** — Only where we actually query JSON fields.

### Index Inventory

| Table | Index | Columns | Purpose |
|-------|-------|---------|---------|
| tenants | `idx_tenants_slug` | `slug` | Lookup by URL slug |
| tenants | `idx_tenants_status` | `status` | Filter by lifecycle state |
| devices | `idx_devices_tenant` | `tenant_id` | FK lookup |
| devices | `idx_devices_status` | `tenant_id, status` | "Show me all active devices" |
| devices | `idx_devices_name` | `tenant_id, name` | "Find device by name" |
| certificates | `idx_certs_device` | `device_id` | FK lookup |
| certificates | `idx_certs_expiry` | `expires_at WHERE active` | Certificate rotation scheduler |
| audit_logs | `idx_audit_tenant` | `tenant_id, created_at DESC` | "Show tenant audit trail" |
| audit_logs | `idx_audit_details` | `details` (GIN) | Full-text search in audit details |

### Monitoring Index Health

```sql
-- Find unused indexes (candidates for removal)
SELECT
    schemaname, tablename, indexname,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find missing indexes (sequential scans on large tables)
SELECT
    relname AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    n_live_tup AS estimated_rows
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
  AND n_live_tup > 10000
ORDER BY seq_tup_read DESC;
```

---

## 4.6 Migration Tooling

For production, we recommend **golang-migrate** (Go) or **Alembic** (Python) instead of raw SQL files:

🔵 **Go — Using golang-migrate**:

```go
// internal/db/migrate.go
package db

import (
    "fmt"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(databaseURL string) error {
    m, err := migrate.New("file://migrations", databaseURL)
    if err != nil {
        return fmt.Errorf("create migrator: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return fmt.Errorf("run migrations: %w", err)
    }

    version, dirty, _ := m.Version()
    fmt.Printf("Database at version %d (dirty: %v)\n", version, dirty)
    return nil
}
```

🐍 **Python — Using Alembic**:

```python
# alembic/env.py (simplified)
from alembic import context
from sqlalchemy import create_engine
from app.models import Base  # SQLAlchemy metadata
import os


def run_migrations():
    engine = create_engine(os.environ["DATABASE_URL"])

    with engine.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=Base.metadata,
            compare_type=True,
        )

        with context.begin_transaction():
            context.run_migrations()
```

---

## 4.7 Connection Pool Management

In production, connection pool sizing is critical:

🔵 **Go — pgxpool configuration**:

```go
// internal/db/pool.go
package db

import (
    "context"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
    config, err := pgxpool.ParseConfig(databaseURL)
    if err != nil {
        return nil, err
    }

    // Pool sizing
    config.MaxConns = 25              // Match (available_connections / num_instances)
    config.MinConns = 5
    config.MaxConnLifetime = 30 * time.Minute
    config.MaxConnIdleTime = 5 * time.Minute

    // Health checks
    config.HealthCheckPeriod = 30 * time.Second

    return pgxpool.NewWithConfig(ctx, config)
}
```

🐍 **Python — asyncpg pool**:

```python
# db/connection.py
import asyncpg


async def create_pool(database_url: str) -> asyncpg.Pool:
    return await asyncpg.create_pool(
        database_url,
        min_size=5,
        max_size=25,
        max_inactive_connection_lifetime=300,  # 5 minutes
        command_timeout=30,
    )
```

### Pool Sizing Formula

```
max_connections_per_instance = (max_pg_connections - reserved) / num_app_instances

Example:
  PostgreSQL max_connections = 100
  Reserved for admin/monitoring = 10
  App instances = 3
  Max per instance = (100 - 10) / 3 ≈ 30
```

> ☕ *Java context*: This replaces HikariCP configuration. The same sizing principles apply — PostgreSQL has a hard limit on connections, and each app instance needs its fair share. In .NET, this maps to Npgsql connection pool settings in the connection string.

---

## 4.8 Transaction Patterns

### Device Provisioning Transaction

Device creation involves multiple tables and must be atomic:

🔵 **Go — Transaction with rollback**:

```go
// internal/device/service.go
func (s *Service) CreateDevice(ctx context.Context, req CreateDeviceRequest) (*Device, error) {
    var device Device

    err := s.pool.BeginTxFunc(ctx, pgx.TxOptions{}, func(tx pgx.Tx) error {
        // 1. Check quota
        var count int
        err := tx.QueryRow(ctx,
            "SELECT COUNT(*) FROM devices WHERE tenant_id = $1",
            req.TenantID,
        ).Scan(&count)
        if err != nil {
            return fmt.Errorf("check quota: %w", err)
        }

        var maxDevices int
        err = tx.QueryRow(ctx,
            "SELECT max_devices FROM tenants WHERE id = $1",
            req.TenantID,
        ).Scan(&maxDevices)
        if err != nil {
            return fmt.Errorf("get quota limit: %w", err)
        }

        if count >= maxDevices {
            return ErrQuotaExceeded
        }

        // 2. Create device record
        err = tx.QueryRow(ctx,
            `INSERT INTO devices (tenant_id, name, metadata)
             VALUES ($1, $2, $3)
             RETURNING id, tenant_id, name, status, created_at`,
            req.TenantID, req.Name, req.Metadata,
        ).Scan(&device.ID, &device.TenantID, &device.Name, &device.Status, &device.CreatedAt)
        if err != nil {
            return fmt.Errorf("insert device: %w", err)
        }

        return nil
    })

    return &device, err
}
```

🐍 **Python — Transaction with rollback**:

```python
# services/device_service.py
from asyncpg import Pool, UniqueViolationError


class DeviceService:
    def __init__(self, pool: Pool):
        self.pool = pool

    async def create_device(
        self, tenant_id: str, name: str, metadata: dict | None = None
    ) -> dict:
        async with self.pool.acquire() as conn:
            async with conn.transaction():
                # 1. Check quota
                count = await conn.fetchval(
                    "SELECT COUNT(*) FROM devices WHERE tenant_id = $1",
                    tenant_id,
                )
                max_devices = await conn.fetchval(
                    "SELECT max_devices FROM tenants WHERE id = $1",
                    tenant_id,
                )

                if count >= max_devices:
                    raise QuotaExceededException(
                        f"Tenant {tenant_id} has reached device limit ({max_devices})"
                    )

                # 2. Create device (transaction auto-rolls back on exception)
                try:
                    device = await conn.fetchrow(
                        """INSERT INTO devices (tenant_id, name, metadata)
                           VALUES ($1, $2, $3::jsonb)
                           RETURNING id, tenant_id, name, status, created_at""",
                        tenant_id, name, json.dumps(metadata or {}),
                    )
                except UniqueViolationError:
                    raise DeviceAlreadyExistsError(f"Device '{name}' already exists")

                return dict(device)
```

---

## 4.9 JSONB for Flexible Metadata

PostgreSQL's `jsonb` type lets us store semi-structured data without schema changes:

```sql
-- Query devices by metadata
SELECT * FROM devices
WHERE tenant_id = $1
  AND metadata @> '{"type": "temperature"}'::jsonb;

-- Query devices by nested metadata
SELECT * FROM devices
WHERE tenant_id = $1
  AND metadata->'location'->>'building' = 'warehouse-a';

-- Index for fast metadata queries
CREATE INDEX idx_devices_metadata_type
ON devices USING GIN ((metadata -> 'type'));
```

> 💡 **When to use JSONB vs. columns**: Use columns for data you filter/sort on frequently (status, tenant_id). Use JSONB for data that varies between devices (custom tags, location metadata, device-specific config).

---

## 4.10 Soft Deletes & Lifecycle States

We use **status enums** instead of hard deletes for devices and tenants:

```sql
-- Soft delete = status change
UPDATE devices SET status = 'deleted', updated_at = NOW()
WHERE id = $1 AND tenant_id = $2;

-- Exclude soft-deleted records in normal queries
-- (This is done automatically via a view or a default scope)
CREATE VIEW active_devices AS
SELECT * FROM devices WHERE status != 'deleted';
```

**Why not hard deletes?**

| Concern | Hard Delete | Soft Delete (Status) |
|---------|------------|---------------------|
| Audit trail | Lost | Preserved |
| Undo capability | Impossible | Change status back |
| FK integrity | CASCADE risk | Safe |
| Provider cleanup | Race condition | Deferred cleanup job |
| Historical queries | Data gone | Data retained |

---

## 4.11 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Entity model** | 6 core tables, all linked through `tenant_id` |
| **RLS** | Database-enforced tenant isolation via session variables |
| **Indexing** | Composite indexes for common queries, partial for active records |
| **Migrations** | golang-migrate (Go) or Alembic (Python) for version control |
| **Connection pools** | Size based on PostgreSQL limits and instance count |
| **Transactions** | Atomic quota check + creation in a single transaction |
| **JSONB** | Flexible metadata without schema migrations |
| **Soft deletes** | Status enums preserve audit trail and enable undo |

### What's Next

In **Chapter 5**, we build the **Provider Abstraction Layer** — the central decoupling mechanism that allows every platform service to work with AWS, Azure, and Mosquitto through a single interface.
