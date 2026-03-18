# Chapter 3: Local Development Environment

> **Part I — Foundations**

The fastest path to productive development is a complete local environment that mirrors production. This chapter sets up everything — database, cache, event bus, secrets manager, MQTT broker, observability stack, and the platform API — using a single Docker Compose file. A cloud account is not required.

---

## 3.1 Prerequisites

| Tool | Install |
|---|---|
| **Docker Desktop** (Mac/Win) or **Docker Engine** (Linux) | https://docs.docker.com/get-docker/ — version 24+ |
| **Docker Compose v2** | Bundled with Docker Desktop; `apt install docker-compose-plugin` on Linux |
| **Go 1.22+** | https://go.dev/dl/ |
| **Node.js 20 LTS** | https://nodejs.org — for web console and CLI |
| **kubectl** | https://kubernetes.io/docs/tasks/tools/ |
| **Terraform / OpenTofu** | https://opentofu.org/docs/intro/install/ |
| **jq** | `brew install jq` / `apt install jq` |
| **make** | Usually pre-installed |
| **pre-commit** | `pip install pre-commit` |
| **Vault CLI** (optional) | https://developer.hashicorp.com/vault/install |
| **mosquitto-clients** (optional) | `brew install mosquitto` / `apt install mosquitto-clients` |

---

## 3.2 Repository Structure

```
iot-platform/
├── cmd/                          # Go binary entrypoints
│   ├── api/main.go               # HTTP API server
│   ├── worker/main.go            # Background workers (NATS consumers)
│   └── migrate/main.go           # Database migration runner
├── internal/                     # Private application packages
│   ├── audit/                    # Audit logging service
│   ├── auth/                     # JWT authentication & RBAC
│   ├── certificate/              # Certificate lifecycle service
│   ├── db/
│   │   ├── migrations/           # SQL migration files (golang-migrate)
│   │   ├── queries/              # SQL query files (sqlc input)
│   │   └── generated/            # sqlc-generated Go code (DO NOT EDIT)
│   ├── device/                   # Device lifecycle service
│   ├── events/                   # NATS event publisher/subscriber
│   ├── handler/                  # HTTP handlers
│   ├── metrics/                  # Prometheus metric definitions
│   ├── middleware/               # HTTP middleware (auth, logging, metrics)
│   ├── policy/                   # Topic namespace & policy engine
│   ├── providers/                # IoT provider abstraction layer
│   │   ├── interface.go          # IoTProvider interface
│   │   ├── aws/                  # AWS IoT Core adapter
│   │   ├── azure/                # Azure IoT Hub adapter
│   │   └── mosquitto/            # Eclipse Mosquitto adapter
│   ├── provisioning/             # Provisioning workflow orchestrator
│   ├── routing/                  # Telemetry routing config service
│   ├── tenant/                   # Tenant management service
│   └── vault/                    # Vault client wrapper
├── api/
│   ├── openapi.yaml              # OpenAPI 3.1 specification
│   └── codegen.yaml              # oapi-codegen configuration
├── web/                          # React + TypeScript console
├── cli/                          # Cobra CLI tool
├── deploy/
│   ├── docker/                   # Docker Compose & configs
│   ├── k8s/                      # Kubernetes manifests (Kustomize)
│   └── terraform/                # Infrastructure as Code
├── scripts/
├── Makefile
├── go.mod
└── go.sum
```

---

## 3.3 Docker Compose: The Complete Local Stack

```yaml
# deploy/docker/docker-compose.yml
version: '3.9'

networks:
  iot-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  postgres_data:
  redis_data:
  vault_data:
  nats_data:
  mosquitto_data:
  mosquitto_log:
  prometheus_data:
  grafana_data:

services:

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       iotplatform
      POSTGRES_USER:     iotplatform
      POSTGRES_PASSWORD: devpassword123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports: ['5432:5432']
    networks: [iot-net]
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U iotplatform']
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass devpassword123 --save 60 1
    volumes: [redis_data:/data]
    ports: ['6379:6379']
    networks: [iot-net]

  nats:
    image: nats:2.10-alpine
    command: -js -sd /data -m 8222 -n iot-platform
    volumes: [nats_data:/data]
    ports:
      - '4222:4222'
      - '8222:8222'
    networks: [iot-net]

  vault:
    image: hashicorp/vault:1.17
    environment:
      VAULT_DEV_ROOT_TOKEN_ID:   root
      VAULT_DEV_LISTEN_ADDRESS:  0.0.0.0:8200
    ports: ['8200:8200']
    cap_add: [IPC_LOCK]
    networks: [iot-net]

  mosquitto:
    image: eclipse-mosquitto:2.0
    volumes:
      - ./mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf:ro
      - mosquitto_data:/mosquitto/data
    ports:
      - '1883:1883'
      - '8883:8883'
      - '9001:9001'
    networks: [iot-net]

  api:
    build:
      context: ../../
      dockerfile: deploy/docker/Dockerfile.dev
    env_file: ../../.env
    volumes: [../../:/app]
    ports: ['8080:8080']
    depends_on:
      postgres: { condition: service_healthy }
      vault:    { condition: service_healthy }
    networks: [iot-net]

  prometheus:
    image: prom/prometheus:v2.52.0
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports: ['9090:9090']
    networks: [iot-net]

  grafana:
    image: grafana/grafana:11.0.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
    ports: ['3000:3000']
    networks: [iot-net]

  jaeger:
    image: jaegertracing/all-in-one:1.58
    environment:
      COLLECTOR_OTLP_ENABLED: 'true'
    ports:
      - '16686:16686'
      - '4317:4317'
      - '4318:4318'
    networks: [iot-net]

  loki:
    image: grafana/loki:3.0.0
    ports: ['3100:3100']
    networks: [iot-net]
```

---

## 3.4 Mosquitto Configuration

```conf
# deploy/docker/mosquitto/mosquitto.conf
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log

# Plaintext listener (DEVELOPMENT ONLY — remove in production)
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
acl_file      /mosquitto/config/acl

# TLS listener (required in production)
listener 8883
cafile   /mosquitto/config/certs/ca.crt
certfile /mosquitto/config/certs/server.crt
keyfile  /mosquitto/config/certs/server.key
require_certificate true
use_identity_as_username true
acl_file /mosquitto/config/acl
```

---

## 3.5 Vault Initialisation Script

```bash
#!/usr/bin/env bash
# scripts/init-vault.sh — Run once after 'docker compose up'
set -euo pipefail

export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=root

# Enable PKI secrets engine
vault secrets enable pki 2>/dev/null || echo 'PKI already enabled'
vault secrets tune -max-lease-ttl=87600h pki

# Generate root CA
vault write -format=json pki/root/generate/internal \
    common_name='IoT Platform Root CA' \
    ttl='87600h' \
    key_type='ec' \
    key_bits=384

# Configure PKI URLs
vault write pki/config/urls \
    issuing_certificates='http://vault:8200/v1/pki/ca' \
    crl_distribution_points='http://vault:8200/v1/pki/crl'

# Create device certificate role
vault write pki/roles/device-cert \
    allowed_domains='devices.iotplatform.local' \
    allow_subdomains=true \
    max_ttl='8760h' \
    key_type='ec' \
    key_bits=256 \
    client_flag=true

# Enable AppRole auth
vault auth enable approle 2>/dev/null || echo 'AppRole already enabled'
vault secrets enable -path=secret kv-v2 2>/dev/null || echo 'KV already enabled'

# Create platform policy
vault policy write iot-platform - <<'EOF'
path "pki/issue/device-cert" { capabilities = ["create", "update"] }
path "pki/revoke"            { capabilities = ["create", "update"] }
path "secret/data/providers/*" { capabilities = ["create","read","update","delete"] }
path "secret/data/certs/*"     { capabilities = ["create","read","delete"] }
EOF

# Create AppRole
vault write auth/approle/role/iot-platform \
    token_policies='iot-platform' \
    token_ttl=1h \
    secret_id_ttl=720h

ROLE_ID=$(vault read -field=role_id auth/approle/role/iot-platform/role-id)
SECRET_ID=$(vault write -field=secret_id -f auth/approle/role/iot-platform/secret-id)

echo "VAULT_ROLE_ID=$ROLE_ID"
echo "VAULT_SECRET_ID=$SECRET_ID"
```

---

## 3.6 Environment File (.env)

```dotenv
# .env — LOCAL DEVELOPMENT ONLY. Never commit this file.

DATABASE_URL=postgres://iotplatform:devpassword123@localhost:5432/iotplatform?sslmode=disable
REDIS_URL=redis://:devpassword123@localhost:6379/0
NATS_URL=nats://localhost:4222
NATS_STREAM_NAME=IOT_PLATFORM_EVENTS

VAULT_ADDR=http://localhost:8200
VAULT_TOKEN=root  # Dev mode only. Production: use VAULT_ROLE_ID + VAULT_SECRET_ID

PORT=8080
LOG_LEVEL=debug
ENV=development

# Generate with: openssl rand -hex 32
JWT_SECRET=dev-secret-change-in-production-00000000000000000000000000000000
JWT_ACCESS_EXPIRY=24h

DEFAULT_PROVIDER=mosquitto
MOSQUITTO_HOST=localhost
MOSQUITTO_PORT_PLAIN=1883
MOSQUITTO_PORT_TLS=8883

OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_SERVICE_NAME=iot-platform-api
```

---

## 3.7 Makefile

```makefile
.PHONY: up down logs build test lint migrate vault-init seed

COMPOSE = cd deploy/docker && docker compose

up:           ## Start the complete local stack
	$(COMPOSE) up -d

down:         ## Stop all services
	$(COMPOSE) down

logs:         ## Tail logs. Usage: make logs svc=api
	$(COMPOSE) logs -f $(svc)

build:        ## Build all Go binaries
	go build ./...

test:         ## Run all tests
	go test ./... -race -count=1 -timeout=120s

lint:         ## Run golangci-lint
	golangci-lint run ./...

generate:     ## Regenerate sqlc + OpenAPI types
	sqlc generate
	oapi-codegen -config api/codegen.yaml api/openapi.yaml

migrate:      ## Apply DB migrations
	go run ./cmd/migrate up

vault-init:   ## Configure Vault (run once after 'make up')
	bash scripts/init-vault.sh

seed:         ## Seed dev data
	go run ./scripts/seed
```

---

## 3.8 Quick Start

```bash
# Clone and start
git clone https://github.com/your-org/iot-platform
cd iot-platform
cp .env.example .env

make up          # Start all Docker services
make vault-init  # Configure Vault PKI (first time only)
make migrate     # Apply database schema
make seed        # Load sample data

# Services now available:
# API:        http://localhost:8080
# Grafana:    http://localhost:3000  (admin / admin)
# Prometheus: http://localhost:9090
# Jaeger:     http://localhost:16686
# Vault:      http://localhost:8200  (token: root)
# NATS:       http://localhost:8222  (monitoring UI)
```
