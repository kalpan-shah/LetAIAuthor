# Chapter 4: Database Design & State Management

> **Part II — Core Infrastructure**

The database is the source of truth for all platform state: tenants, devices, certificates, routing rules, and the complete immutable audit trail.

---

## 4.1 Schema Design Decisions

- **UUID primary keys everywhere.** Sequential integer IDs are predictable and enumerable by attackers. UUIDs prevent this.
- **`created_at` and `updated_at` maintained by database triggers.** Application code cannot forget to update them.
- **Soft deletes for devices and certificates.** Setting `status = 'deleted'` preserves history for audit purposes.
- **No secrets in the database.** Certificates, private keys, and cloud credentials are stored in Vault. The database stores only the Vault path reference, serial number, and fingerprint.
- **JSONB for extensible metadata.** Devices and routing rules need arbitrary key-value pairs.
- **Partitioned audit log.** Audit logs grow indefinitely. Partitioning by quarter enables cheap archiving of old data.

---

## 4.2 Complete Database Schema

```sql
-- migrations/001_initial_schema.up.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Auto-update updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END; $$;

-- ── TENANTS ───────────────────────────────────────────────────────────────────
CREATE TABLE tenants (
    id               UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    name             VARCHAR(255) NOT NULL,
    slug             VARCHAR(100) NOT NULL UNIQUE
                     CHECK (slug ~ '^[a-z0-9][a-z0-9\-]{0,98}[a-z0-9]$'),
    provider         VARCHAR(50)  NOT NULL
                     CHECK (provider IN ('aws','azure','mosquitto')),
    region           VARCHAR(100),
    status           VARCHAR(50)  NOT NULL DEFAULT 'active'
                     CHECK (status IN ('active','suspended','archived')),
    max_devices          INTEGER NOT NULL DEFAULT 1000,
    max_certs_per_day    INTEGER NOT NULL DEFAULT 500,
    max_routing_rules    INTEGER NOT NULL DEFAULT 50,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_slug   ON tenants(slug);
CREATE INDEX idx_tenants_status ON tenants(status);
CREATE TRIGGER tenants_updated_at BEFORE UPDATE ON tenants
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ── PROVIDER INTEGRATIONS ─────────────────────────────────────────────────────
CREATE TABLE provider_integrations (
    id                UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id         UUID        NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider          VARCHAR(50) NOT NULL CHECK (provider IN ('aws','azure','mosquitto')),
    aws_account_id    VARCHAR(100),
    aws_role_arn      TEXT,
    aws_external_id   VARCHAR(255),  -- Confused deputy protection
    azure_subscription_id VARCHAR(100),
    azure_hub_hostname    TEXT,
    endpoint_url      TEXT,
    vault_secret_path TEXT NOT NULL,  -- Path to credentials in Vault (NOT the creds)
    status            VARCHAR(50) NOT NULL DEFAULT 'active',
    last_validated_at TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, provider)
);

-- ── DEVICES ───────────────────────────────────────────────────────────────────
CREATE TABLE devices (
    id               UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id        UUID        NOT NULL REFERENCES tenants(id),
    external_id      VARCHAR(255) NOT NULL,   -- ID used by the IoT provider
    name             VARCHAR(255) NOT NULL,
    description      TEXT,
    provider         VARCHAR(50)  NOT NULL,
    status           VARCHAR(50)  NOT NULL DEFAULT 'pending'
                     CHECK (status IN ('pending','active','suspended','deleted')),
    current_cert_id  UUID,
    metadata         JSONB       NOT NULL DEFAULT '{}',
    provisioned_at   TIMESTAMPTZ,
    last_seen_at     TIMESTAMPTZ,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, name)
);

CREATE INDEX idx_devices_tenant_id ON devices(tenant_id);
CREATE INDEX idx_devices_status    ON devices(status) WHERE status != 'deleted';
CREATE INDEX idx_devices_metadata  ON devices USING gin(metadata);

-- ── CERTIFICATES ──────────────────────────────────────────────────────────────
CREATE TABLE certificates (
    id               UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    device_id        UUID        NOT NULL REFERENCES devices(id),
    tenant_id        UUID        NOT NULL REFERENCES tenants(id),
    serial_number    VARCHAR(255) NOT NULL UNIQUE,
    vault_path       TEXT        NOT NULL,   -- Path in Vault where cert+key bundle stored
    subject_cn       VARCHAR(255) NOT NULL,
    fingerprint      VARCHAR(128) NOT NULL,  -- SHA-256, used to identify without storing cert
    not_before       TIMESTAMPTZ NOT NULL,
    not_after        TIMESTAMPTZ NOT NULL,
    status           VARCHAR(50)  NOT NULL DEFAULT 'active'
                     CHECK (status IN ('active','expired','revoked')),
    revoked_at       TIMESTAMPTZ,
    revocation_reason VARCHAR(100),
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_certs_not_after ON certificates(not_after) WHERE status = 'active';
CREATE INDEX idx_certs_device_id ON certificates(device_id);

-- Add FK from devices → certificates
ALTER TABLE devices ADD CONSTRAINT fk_devices_current_cert
    FOREIGN KEY (current_cert_id) REFERENCES certificates(id)
    DEFERRABLE INITIALLY DEFERRED;

-- ── ROUTING RULES ─────────────────────────────────────────────────────────────
CREATE TABLE routing_rules (
    id               UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id        UUID         NOT NULL REFERENCES tenants(id),
    name             VARCHAR(255) NOT NULL,
    topic_filter     TEXT         NOT NULL,
    destination_type VARCHAR(100) NOT NULL,
    destination_config JSONB      NOT NULL DEFAULT '{}',
    sql_filter       TEXT,
    enabled          BOOLEAN      NOT NULL DEFAULT true,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE(tenant_id, name)
);

-- ── AUDIT LOGS (Partitioned by quarter) ────────────────────────────────────
CREATE TABLE audit_logs (
    id             UUID         NOT NULL DEFAULT uuid_generate_v4(),
    tenant_id      UUID,
    actor_id       VARCHAR(255) NOT NULL,
    actor_type     VARCHAR(50)  NOT NULL
                   CHECK (actor_type IN ('user','api_key','service','system')),
    action         VARCHAR(100) NOT NULL,
    resource_type  VARCHAR(100) NOT NULL,
    resource_id    UUID,
    outcome        VARCHAR(50)  NOT NULL CHECK (outcome IN ('success','failure')),
    details        JSONB        NOT NULL DEFAULT '{}',
    ip_address     INET,
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create quarterly partitions (add new ones via scheduled job)
CREATE TABLE audit_logs_2025_q1 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
CREATE TABLE audit_logs_2025_q2 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');
CREATE TABLE audit_logs_2025_q3 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-07-01') TO ('2025-10-01');
CREATE TABLE audit_logs_2025_q4 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-10-01') TO ('2026-01-01');

CREATE INDEX idx_audit_tenant  ON audit_logs(tenant_id, created_at DESC);
CREATE INDEX idx_audit_actor   ON audit_logs(actor_id,   created_at DESC);
CREATE INDEX idx_audit_action  ON audit_logs(action, created_at DESC);
```

---

## 4.3 sqlc Configuration

`sqlc` reads your SQL schema and query files and generates idiomatic, type-safe Go code:

```yaml
# sqlc.yaml
version: '2'
sql:
  - engine:  'postgresql'
    queries:  'internal/db/queries/'
    schema:   'internal/db/migrations/'
    gen:
      go:
        package:                    'db'
        out:                        'internal/db/generated'
        emit_json_tags:             true
        emit_pointers_for_null_types: true
        overrides:
          - db_type: 'uuid'
            go_type:  'github.com/google/uuid.UUID'
          - db_type: 'timestamptz'
            go_type:  'time.Time'
          - db_type: 'jsonb'
            go_type:
              import: 'encoding/json'
              type:   'RawMessage'
```

---

## 4.4 Example SQL Queries

```sql
-- internal/db/queries/devices.sql

-- name: CreateDevice :one
INSERT INTO devices (tenant_id, external_id, name, description, provider, metadata)
VALUES ($1, $2, $3, $4, $5, $6)
RETURNING *;

-- name: GetDeviceByID :one
SELECT d.*, c.serial_number AS cert_serial, c.not_after AS cert_expires_at
FROM  devices d
LEFT JOIN certificates c ON c.id = d.current_cert_id
WHERE d.id = $1
  AND d.tenant_id = $2        -- Always scope to tenant!
  AND d.status != 'deleted'
LIMIT 1;

-- name: ListDevicesByTenant :many
SELECT * FROM devices
WHERE tenant_id = $1
  AND status != 'deleted'
  AND ($2::varchar IS NULL OR status = $2)
ORDER BY created_at DESC
LIMIT $3 OFFSET $4;

-- name: GetCertificatesExpiringSoon :many
SELECT c.id AS cert_id, c.device_id, c.tenant_id, c.serial_number,
       c.not_after, d.name AS device_name, d.external_id
FROM  certificates c
JOIN  devices d ON d.current_cert_id = c.id
WHERE c.status = 'active'
  AND c.not_after < NOW() + ($1 || ' days')::interval
ORDER BY c.not_after ASC;
```
