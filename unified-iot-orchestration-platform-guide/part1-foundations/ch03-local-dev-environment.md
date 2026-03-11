# Chapter 3: Local Development Environment

> *"If you can't run it locally, you can't debug it effectively."*

---

## 3.1 Introduction

Before writing a single line of platform code, you need a local environment that mirrors production as closely as possible. By the end of this chapter, you'll have the entire platform — PostgreSQL, Redis, NATS, Vault, and an observability stack — running on your laptop with a single command.

> 💡 **For Beginners**: Docker Compose lets you define multi-container applications in a single YAML file. Think of it as a recipe that says "start a PostgreSQL database, a Redis cache, a message broker, and three other services — all pre-configured to talk to each other."

---

## 3.2 Prerequisites

| Tool | Version | Installation |
|------|---------|-------------|
| Docker Desktop | 24.x+ | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop) |
| Docker Compose | v2.20+ | Included with Docker Desktop |
| Go | 1.22+ | [go.dev/dl](https://go.dev/dl/) |
| Python | 3.12+ | [python.org/downloads](https://www.python.org/downloads/) |
| Make | Any | Included on macOS/Linux; use `choco install make` on Windows |
| `grpcurl` | Latest | `go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest` |
| `psql` (optional) | 16 | Comes with PostgreSQL or install via package manager |

### Verify Installations

```bash
docker --version          # Docker version 24.x.x
docker compose version    # Docker Compose version v2.20+
go version                # go1.22.x
python3 --version         # Python 3.12.x
make --version            # GNU Make 4.x
grpcurl --version         # grpcurl v1.8.x
```

---

## 3.3 Project Structure

```
iot-platform/
├── cmd/
│   └── server/
│       └── main.go                 # Go entrypoint
├── internal/
│   ├── tenant/                     # Tenant service module
│   ├── device/                     # Device lifecycle module
│   ├── certificate/                # Certificate lifecycle module
│   ├── policy/                     # Policy & namespace module
│   ├── provisioning/               # Provisioning workflow module
│   ├── routing/                    # Telemetry routing config module
│   ├── audit/                      # Audit service module
│   ├── quota/                      # Quota enforcement module
│   ├── provider/                   # Provider Abstraction Layer
│   │   ├── provider.go             # Interface definition
│   │   ├── aws/                    # AWS adapter
│   │   ├── azure/                  # Azure adapter
│   │   └── mosquitto/              # Mosquitto adapter
│   └── middleware/                 # Interceptors & middleware
├── api/
│   └── proto/
│       └── v1/                     # Protobuf definitions
│           ├── tenant.proto
│           ├── device.proto
│           └── certificate.proto
├── app/                            # Python FastAPI application
│   ├── main.py
│   ├── routers/
│   │   ├── tenants.py
│   │   ├── devices.py
│   │   └── certificates.py
│   ├── services/
│   │   ├── tenant_service.py
│   │   ├── device_service.py
│   │   └── certificate_service.py
│   ├── providers/
│   │   ├── base.py
│   │   ├── aws_provider.py
│   │   ├── azure_provider.py
│   │   └── mosquitto_provider.py
│   ├── middleware/
│   │   ├── tenant.py
│   │   └── audit.py
│   ├── models/
│   │   ├── tenant.py
│   │   └── device.py
│   └── db/
│       ├── connection.py
│       └── migrations/
├── deployments/
│   ├── docker-compose.yml          # Full local stack
│   ├── docker-compose.dev.yml      # Dev overrides
│   └── vault/
│       └── config.hcl              # Vault dev config
├── migrations/
│   ├── 001_create_tenants.sql
│   ├── 002_create_devices.sql
│   └── 003_create_certificates.sql
├── scripts/
│   ├── seed.sh                     # Seed development data
│   └── vault-init.sh               # Initialize Vault secrets
├── Makefile
├── go.mod
├── go.sum
├── requirements.txt
├── requirements-dev.txt
└── .env.example
```

> ☕ *Java developers*: This is analogous to a Maven/Gradle multi-module project, but without the XML ceremony. The `internal/` directory in Go is a language-enforced visibility boundary — nothing outside this package can import from it. In Spring terms, it's as if your `@Service` classes were physically impossible to use from outside their module. Python's `app/` directory follows a standard FastAPI project layout similar to Django's app structure.

---

## 3.4 Docker Compose: The Complete Local Stack

```yaml
# deployments/docker-compose.yml
version: "3.9"

services:
  # ─────────────────────────────────────────────────
  #  DATA STORES
  # ─────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: iot_platform
      POSTGRES_USER: platform
      POSTGRES_PASSWORD: localdev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U platform -d iot_platform"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  nats:
    image: nats:2.10-alpine
    ports:
      - "4222:4222"   # Client connections
      - "8222:8222"   # HTTP monitoring
    command: >
      --jetstream
      --store_dir /data
      --http_port 8222
    volumes:
      - nats_data:/data
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:8222/healthz"]
      interval: 5s
      timeout: 3s
      retries: 5

  # ─────────────────────────────────────────────────
  #  SECURITY
  # ─────────────────────────────────────────────────
  vault:
    image: hashicorp/vault:1.15
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "dev-root-token"
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    cap_add:
      - IPC_LOCK
    healthcheck:
      test: ["CMD", "vault", "status"]
      interval: 5s
      timeout: 3s
      retries: 5

  # ─────────────────────────────────────────────────
  #  OBSERVABILITY
  # ─────────────────────────────────────────────────
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.92.0
    ports:
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP HTTP receiver
    volumes:
      - ./deployments/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
    depends_on:
      - prometheus
      - loki

  prometheus:
    image: prom/prometheus:v2.49.0
    ports:
      - "9090:9090"
    volumes:
      - ./deployments/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:10.3.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: localdev
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
    volumes:
      - ./deployments/grafana/provisioning:/etc/grafana/provisioning
      - grafana_data:/var/lib/grafana

  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki

  # ─────────────────────────────────────────────────
  #  LOCAL IOT BROKER (Mosquitto — simulates a provider)
  # ─────────────────────────────────────────────────
  mosquitto:
    image: eclipse-mosquitto:2.0
    ports:
      - "1883:1883"   # MQTT
      - "9001:9001"   # WebSocket
    volumes:
      - ./deployments/mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - mosquitto_data:/mosquitto/data
      - mosquitto_logs:/mosquitto/log

volumes:
  postgres_data:
  nats_data:
  prometheus_data:
  grafana_data:
  loki_data:
  mosquitto_data:
  mosquitto_logs:
```

### Dev Override File

```yaml
# deployments/docker-compose.dev.yml
version: "3.9"

services:
  postgres:
    ports:
      - "5432:5432"
    environment:
      POSTGRES_LOG_STATEMENT: "all"   # Log all SQL for debugging

  redis:
    command: redis-server --maxmemory 64mb --maxmemory-policy allkeys-lru --loglevel verbose
```

---

## 3.5 Makefile: Developer Commands

```makefile
# Makefile — Developer workflow commands

.PHONY: help up down logs db-shell redis-shell nats-monitor
.PHONY: migrate seed vault-init test test-go test-python
.PHONY: run-go run-python proto lint

# ─────────────────────────────────────────────────
#  INFRASTRUCTURE
# ─────────────────────────────────────────────────

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

up: ## Start all infrastructure services
	docker compose -f deployments/docker-compose.yml up -d
	@echo "Waiting for services to be healthy..."
	@sleep 5
	@$(MAKE) vault-init
	@echo "✅ All services running"
	@echo "  PostgreSQL: localhost:5432"
	@echo "  Redis:      localhost:6379"
	@echo "  NATS:       localhost:4222 (monitor: localhost:8222)"
	@echo "  Vault:      http://localhost:8200 (token: dev-root-token)"
	@echo "  Grafana:    http://localhost:3000 (admin/localdev)"
	@echo "  Mosquitto:  localhost:1883"

down: ## Stop all services
	docker compose -f deployments/docker-compose.yml down

down-clean: ## Stop all services and remove volumes
	docker compose -f deployments/docker-compose.yml down -v

logs: ## Tail logs from all services
	docker compose -f deployments/docker-compose.yml logs -f

# ─────────────────────────────────────────────────
#  DATABASE
# ─────────────────────────────────────────────────

migrate: ## Run database migrations
	@for f in migrations/*.sql; do \
		echo "Applying $$f..."; \
		PGPASSWORD=localdev psql -h localhost -U platform -d iot_platform -f $$f; \
	done

seed: ## Seed development data
	bash scripts/seed.sh

db-shell: ## Open PostgreSQL shell
	PGPASSWORD=localdev psql -h localhost -U platform -d iot_platform

# ─────────────────────────────────────────────────
#  APPLICATION
# ─────────────────────────────────────────────────

run-go: ## Run Go server
	go run ./cmd/server/main.go

run-python: ## Run Python FastAPI server
	uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

proto: ## Regenerate protobuf code
	buf generate

# ─────────────────────────────────────────────────
#  TESTING
# ─────────────────────────────────────────────────

test: test-go test-python ## Run all tests

test-go: ## Run Go tests
	go test ./... -race -count=1 -coverprofile=coverage.out
	go tool cover -func=coverage.out

test-python: ## Run Python tests
	pytest app/ --cov=app --cov-report=term-missing -v

lint: ## Run linters
	golangci-lint run ./...
	ruff check app/
	ruff format --check app/

# ─────────────────────────────────────────────────
#  UTILITIES
# ─────────────────────────────────────────────────

vault-init: ## Initialize Vault with development secrets
	bash scripts/vault-init.sh

redis-shell: ## Open Redis CLI
	docker exec -it $$(docker compose -f deployments/docker-compose.yml ps -q redis) redis-cli

nats-monitor: ## Open NATS monitoring
	@echo "NATS monitoring: http://localhost:8222"
	@echo "Subscriptions:   http://localhost:8222/subsz"
	@echo "Connections:     http://localhost:8222/connz"
```

---

## 3.6 Initial Database Migrations

```sql
-- migrations/001_create_tenants.sql

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TYPE tenant_status AS ENUM ('active', 'suspended', 'archived');

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(63) NOT NULL UNIQUE,
    provider        VARCHAR(50) NOT NULL,       -- 'aws', 'azure', 'mosquitto'
    cloud_account   VARCHAR(255),               -- Provider-specific account reference
    region          VARCHAR(50),
    status          tenant_status NOT NULL DEFAULT 'active',

    -- Quota limits
    max_devices         INTEGER NOT NULL DEFAULT 1000,
    max_routing_rules   INTEGER NOT NULL DEFAULT 50,
    max_cert_issuance   INTEGER NOT NULL DEFAULT 5000,

    -- Timestamps
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Row-Level Security
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;

-- Index for slug lookups
CREATE INDEX idx_tenants_slug ON tenants (slug);
CREATE INDEX idx_tenants_status ON tenants (status);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tenants_updated_at
    BEFORE UPDATE ON tenants
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

```sql
-- migrations/002_create_devices.sql

CREATE TYPE device_status AS ENUM ('active', 'suspended', 'pending', 'deleted');

CREATE TABLE devices (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    provider_device_id  VARCHAR(255),       -- Provider's reference (e.g., AWS Thing ARN)
    status              device_status NOT NULL DEFAULT 'pending',

    -- Certificate reference
    active_cert_id      UUID,               -- Set after provisioning

    -- Metadata
    metadata            JSONB DEFAULT '{}',

    -- Timestamps
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Uniqueness constraint: device name unique per tenant
    UNIQUE(tenant_id, name)
);

-- Row-Level Security
ALTER TABLE devices ENABLE ROW LEVEL SECURITY;

CREATE POLICY devices_tenant_isolation ON devices
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Indexes
CREATE INDEX idx_devices_tenant ON devices (tenant_id);
CREATE INDEX idx_devices_status ON devices (tenant_id, status);
CREATE INDEX idx_devices_name ON devices (tenant_id, name);

CREATE TRIGGER devices_updated_at
    BEFORE UPDATE ON devices
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

```sql
-- migrations/003_create_certificates.sql

CREATE TYPE cert_status AS ENUM ('active', 'pending', 'revoked', 'expired');

CREATE TABLE certificates (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    device_id           UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    tenant_id           UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider_cert_id    VARCHAR(255),       -- Provider's certificate reference
    fingerprint         VARCHAR(64),        -- SHA-256 fingerprint of the cert
    status              cert_status NOT NULL DEFAULT 'pending',

    -- We store the PEM but NOT the private key
    cert_pem            TEXT,

    -- Lifecycle
    issued_at           TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    revoked_at          TIMESTAMPTZ,

    -- Timestamps
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Row-Level Security
ALTER TABLE certificates ENABLE ROW LEVEL SECURITY;

CREATE POLICY certs_tenant_isolation ON certificates
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Indexes
CREATE INDEX idx_certs_device ON certificates (device_id);
CREATE INDEX idx_certs_tenant ON certificates (tenant_id);
CREATE INDEX idx_certs_status ON certificates (tenant_id, status);
CREATE INDEX idx_certs_expiry ON certificates (expires_at) WHERE status = 'active';

CREATE TRIGGER certificates_updated_at
    BEFORE UPDATE ON certificates
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

---

## 3.7 Vault Initialization Script

```bash
#!/bin/bash
# scripts/vault-init.sh — Initialize Vault for local development

set -euo pipefail

export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="dev-root-token"

echo "🔐 Initializing Vault for local development..."

# Enable KV v2 secrets engine for cloud credentials
vault secrets enable -path=iot-platform/credentials kv-v2 2>/dev/null || true

# Store example cloud credentials (for development only!)
vault kv put iot-platform/credentials/tenant-demo-aws \
    provider=aws \
    role_arn="arn:aws:iam::123456789012:role/iot-platform-dev" \
    external_id="dev-external-id" \
    region="us-east-1"

vault kv put iot-platform/credentials/tenant-demo-azure \
    provider=azure \
    tenant_id="00000000-0000-0000-0000-000000000000" \
    client_id="demo-client-id" \
    client_secret="demo-client-secret" \
    subscription_id="00000000-0000-0000-0000-000000000000"

vault kv put iot-platform/credentials/tenant-demo-mosquitto \
    provider=mosquitto \
    broker_url="tcp://mosquitto:1883" \
    admin_username="admin" \
    admin_password="localdev"

# Enable PKI secrets engine for certificate management
vault secrets enable -path=iot-platform/pki pki 2>/dev/null || true
vault secrets tune -max-lease-ttl=87600h iot-platform/pki

# Generate root certificate
vault write iot-platform/pki/root/generate/internal \
    common_name="IoT Platform Dev CA" \
    ttl=87600h

# Configure issuing URL
vault write iot-platform/pki/config/urls \
    issuing_certificates="http://vault:8200/v1/iot-platform/pki/ca" \
    crl_distribution_points="http://vault:8200/v1/iot-platform/pki/crl"

# Create a role for issuing device certificates
vault write iot-platform/pki/roles/device-cert \
    allowed_domains="iot.local" \
    allow_subdomains=true \
    max_ttl=8760h \
    key_type=ec \
    key_bits=256

# Create policies for platform services
vault policy write iot-platform-service - <<EOF
# Read cloud credentials
path "iot-platform/credentials/*" {
  capabilities = ["read", "list"]
}

# Issue device certificates
path "iot-platform/pki/issue/device-cert" {
  capabilities = ["create", "update"]
}

# Revoke certificates
path "iot-platform/pki/revoke" {
  capabilities = ["create", "update"]
}
EOF

echo "✅ Vault initialized successfully"
echo "   Root token:  dev-root-token"
echo "   UI:          http://localhost:8200/ui"
```

---

## 3.8 Seed Script

```bash
#!/bin/bash
# scripts/seed.sh — Seed development data

set -euo pipefail

export PGPASSWORD=localdev
PSQL="psql -h localhost -U platform -d iot_platform -q"

echo "🌱 Seeding development data..."

$PSQL <<EOF
-- Insert demo tenants
INSERT INTO tenants (name, slug, provider, cloud_account, region, status, max_devices)
VALUES
    ('Acme Corp', 'acme', 'aws', '123456789012', 'us-east-1', 'active', 5000),
    ('Beta Industries', 'beta', 'azure', 'beta-subscription', 'eastus', 'active', 2000),
    ('Gamma Labs', 'gamma', 'mosquitto', NULL, NULL, 'active', 500)
ON CONFLICT (slug) DO NOTHING;

-- Insert demo devices for Acme
INSERT INTO devices (tenant_id, name, status, metadata)
SELECT t.id, d.name, d.status::device_status, d.metadata::jsonb
FROM tenants t
CROSS JOIN (VALUES
    ('temperature-sensor-01', 'active', '{"type": "temperature", "location": "warehouse-a"}'),
    ('humidity-sensor-01', 'active', '{"type": "humidity", "location": "warehouse-a"}'),
    ('motion-detector-01', 'pending', '{"type": "motion", "location": "entrance"}')
) AS d(name, status, metadata)
WHERE t.slug = 'acme'
ON CONFLICT (tenant_id, name) DO NOTHING;
EOF

echo "✅ Seed data inserted"
echo "   Tenants: Acme (AWS), Beta (Azure), Gamma (Mosquitto)"
echo "   Devices: 3 devices for Acme"
```

---

## 3.9 Environment Configuration

```bash
# .env.example — Copy to .env and customize

# ─── Database ───
DATABASE_URL=postgres://platform:localdev@localhost:5432/iot_platform?sslmode=disable

# ─── Redis ───
REDIS_URL=redis://localhost:6379/0

# ─── NATS ───
NATS_URL=nats://localhost:4222

# ─── Vault ───
VAULT_ADDR=http://localhost:8200
VAULT_TOKEN=dev-root-token

# ─── Server ───
GO_PORT=9090
GO_HTTP_PORT=8080
PYTHON_PORT=8000

# ─── Observability ───
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=iot-platform

# ─── Development ───
LOG_LEVEL=debug
LOG_FORMAT=text    # 'text' for dev, 'json' for production
```

---

## 3.10 Running the Stack

### Quick Start

```bash
# 1. Clone and enter the project
git clone <your-repo-url> iot-platform
cd iot-platform

# 2. Copy environment config
cp .env.example .env

# 3. Start all infrastructure
make up

# 4. Run database migrations
make migrate

# 5. Seed development data
make seed

# 6. Start the application (choose one)
make run-go       # Go server on :9090 (gRPC) + :8080 (REST)
# OR
make run-python   # Python server on :8000 (REST)
```

### Verifying Everything Works

```bash
# Check infrastructure health
docker compose -f deployments/docker-compose.yml ps

# Test PostgreSQL
make db-shell
# => \dt   (should list tenants, devices, certificates tables)

# Test Redis
make redis-shell
# => PING   (should return PONG)

# Test NATS
curl http://localhost:8222/varz | jq .

# Test Vault
export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=dev-root-token
vault kv list iot-platform/credentials/

# Test API (once application is running)
# Go server:
grpcurl -plaintext localhost:9090 list

# Python server:
curl http://localhost:8000/docs    # Opens Swagger UI
```

🐍 **Python — Quick API test**:

```python
# test_local.py — Quick smoke test
import httpx
import asyncio


async def smoke_test():
    async with httpx.AsyncClient(base_url="http://localhost:8000") as client:
        # Health check
        r = await client.get("/health")
        print(f"Health: {r.status_code}")

        # List tenants
        r = await client.get(
            "/api/v1/tenants",
            headers={"X-Tenant-ID": "system"},  # System-level call
        )
        print(f"Tenants: {r.json()}")

        # Create a device
        r = await client.post(
            "/api/v1/devices",
            headers={"X-Tenant-ID": "acme"},
            json={"name": "test-sensor-01", "metadata": {"type": "test"}},
        )
        print(f"Created device: {r.json()}")


asyncio.run(smoke_test())
```

🔵 **Go — Quick API test via grpcurl**:

```bash
# List available services
grpcurl -plaintext localhost:9090 list

# Create a tenant
grpcurl -plaintext -d '{
    "name": "Test Corp",
    "slug": "test-corp",
    "provider": "mosquitto"
}' localhost:9090 iot.platform.v1.TenantService/CreateTenant

# List devices for a tenant
grpcurl -plaintext \
    -H "X-Tenant-ID: acme" \
    localhost:9090 iot.platform.v1.DeviceService/ListDevices
```

---

## 3.11 Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `port 5432 already in use` | Local PostgreSQL running | Stop local PG or change port in compose |
| `vault: permission denied` | Wrong token | `export VAULT_TOKEN=dev-root-token` |
| NATS `connection refused` | JetStream not enabled | Check `--jetstream` flag in compose |
| `grpcurl: connection refused` | Go server not running | Run `make run-go` first |
| Python `ModuleNotFoundError` | Dependencies not installed | `pip install -r requirements.txt` |
| Docker `no space left on device` | Docker disk full | `docker system prune -a` |

---

## 3.12 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Docker Compose** | One command starts the entire platform infrastructure |
| **Makefile** | Developer-friendly commands for common workflows |
| **Migrations** | Schema-first approach with tenant isolation built in |
| **Vault dev mode** | Pre-configured with PKI and sample credentials |
| **Seed data** | Three tenants (one per provider) with sample devices |

### What's Next

In **Chapter 4**, we'll deep-dive into database design — the full schema, index strategies, migration tooling, and how row-level security enforces tenant isolation at the PostgreSQL level.
