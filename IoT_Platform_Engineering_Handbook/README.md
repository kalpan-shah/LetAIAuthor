# Building the Unified IoT Orchestration Platform
## A Complete Developer, SRE & Security Engineering Handbook

---

### Overview

This handbook covers everything needed to design, build, and operate a **production-grade, provider-agnostic IoT device management platform** — from local development on a single laptop to multi-cloud Kubernetes deployments managing 50,000+ devices.

**Tech Stack:** Eclipse Mosquitto · AWS IoT Core · Azure IoT Hub · Go 1.22 · PostgreSQL 16 · HashiCorp Vault · NATS JetStream · Kubernetes 1.29+ · Prometheus · Grafana · OpenTelemetry · ArgoCD · Terraform/OpenTofu

---

### Book Structure

| Part | Chapters | Coverage |
|---|---|---|
| **Part I — Foundations** | Ch 1–3 | Platform architecture, tech stack rationale, local Docker Compose dev environment |
| **Part II — Core Infrastructure** | Ch 4–6 | PostgreSQL schema, Vault PKI, NATS JetStream event bus |
| **Part III — Provider Abstraction** | Ch 7–10 | IoTProvider interface, Mosquitto + AWS IoT Core + Azure IoT Hub adapters |
| **Part IV — Platform Services** | Ch 11–15 | Tenant service, Device lifecycle, Certificate service, Policy engine, Routing & Audit |
| **Part V — Developer Interfaces** | Ch 16–17 | REST API, React web console, Cobra CLI |
| **Part VI — SRE & Observability** | Ch 18–19 | Prometheus, structured logging, OpenTelemetry tracing, SLOs, incident runbooks |
| **Part VII — Security (SecOps)** | Ch 20–23 | STRIDE threat model, JWT/RBAC, NetworkPolicy, secrets governance |
| **Part VIII — DevOps** | Ch 24–27 | GitHub Actions CI/CD, Terraform, Kubernetes deployment, production checklist |

---

### Converting Chapters to PDF

Each `.md` file can be converted individually or by part:

```bash
# Single chapter — using pandoc (best quality)
pandoc Chapter_01_Platform_Overview_and_Architecture.md \
  -o Chapter_01.pdf \
  --pdf-engine=xelatex \
  -V geometry:margin=1in \
  -V fontsize=11pt \
  -V monofont="Courier New" \
  --highlight-style=tango \
  --toc

# Entire Part at once
pandoc Part_I_Foundations/*.md \
  -o Part_I_Foundations.pdf \
  --pdf-engine=xelatex \
  -V geometry:margin=1in \
  --highlight-style=tango \
  --toc

# Full handbook (all chapters)
pandoc $(find . -name "*.md" | sort) \
  -o IoT_Platform_Handbook.pdf \
  --pdf-engine=xelatex \
  --toc --toc-depth=2 \
  -V geometry:margin=1in \
  -V fontsize=11pt \
  --highlight-style=tango

# Using grip (GitHub-style rendering, then print to PDF from browser)
pip install grip
grip Chapter_01_Platform_Overview_and_Architecture.md
# Open http://localhost:6419 → Ctrl+P → Save as PDF

# VS Code: Install "Markdown PDF" extension
# Right-click any .md file → "Markdown PDF: Export (pdf)"
```

---

### Quick Start

```bash
git clone https://github.com/your-org/iot-platform
cd iot-platform
cp .env.example .env
make up          # Start all Docker services
make vault-init  # Configure Vault PKI (first time only)
make migrate     # Apply database schema
make seed        # Load sample tenant + device data
```

**Services after `make up`:**

| Service | URL | Credentials |
|---|---|---|
| Platform API | http://localhost:8080 | JWT from `/api/v1/auth/login` |
| Grafana | http://localhost:3000 | admin / admin |
| Prometheus | http://localhost:9090 | — |
| Jaeger (traces) | http://localhost:16686 | — |
| Vault | http://localhost:8200 | Token: `root` |
| NATS Monitor | http://localhost:8222 | — |
| Mosquitto MQTT | localhost:1883 (plain) / 8883 (TLS) | — |
