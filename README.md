# 📚 LetAIAuthor

**AI-generated engineering handbooks — built from structured prompts, not copy-paste.**

> [!IMPORTANT]
> Every book in this repository is **AI-generated**, written by **Claude Sonnet** and **Claude Opus**.
>
> This is an experiment in making AI write full-length technical books from user-provided requirements and specifications. The content aggregates openly available engineering knowledge into structured, cohesive guides that don't exist elsewhere as a single resource.
>
> **Caveats:**
> - The AI does not include source references or citations — readers can't easily trace claims back to original material.
> - This is a starting effort by a novice engineer; the books may not be actively maintained but I'll try to update them when I can.
> - Treat these as learning companions, not authoritative references.

---

## 📖 Books

| # | Book | Chapters | Focus |
|---|------|----------|-------|
| 1 | [**IoT Platform Engineering Handbook**](./IoT_Platform_Engineering_Handbook/) | 27 | Design, build, and operate a production-grade, provider-agnostic IoT device management platform |
| 2 | [**Observability Engineering**](./Observability_Engineering_Book/) | 25 | Measure latency, throughput, saturation & availability with hands-on .NET, Prometheus, Jaeger & Grafana |
| 3 | [**Unified IoT Platform Guide**](./unified-iot-orchestration-platform-guide/) | 20 | Architecture, core infrastructure, platform services, operations & advanced deployment |

---

### 1. IoT Platform Engineering Handbook

A complete Developer, SRE & Security Engineering handbook covering the full stack of a Unified IoT Orchestration Platform.

- **Foundations** — Platform architecture, technology stack rationale, local Docker Compose environment
- **Core Infrastructure** — PostgreSQL schema, HashiCorp Vault PKI, NATS JetStream event bus
- **Provider Abstraction** — `IoTProvider` interface with Mosquitto, AWS IoT Core, and Azure IoT Hub adapters
- **Platform Services** — Tenant management, device lifecycle, certificate service, policy engine, routing & audit
- **Developer Interfaces** — REST API, React web console, Cobra CLI
- **SRE & Security** — Prometheus, OpenTelemetry, STRIDE threat model, JWT/RBAC, NetworkPolicy
- **DevOps** — GitHub Actions CI/CD, Terraform, Kubernetes deployment, production readiness checklist

---

### 2. Observability Engineering

A hands-on guide to understanding what your systems are *actually doing* — not just whether they're running, but **how well**.

- **Foundations** — The four signals, RED/USE methods, why percentiles beat averages
- **Demo Application** — .NET 8 Minimal API + PostgreSQL, fully instrumented with OpenTelemetry
- **Metrics** — Prometheus, PromQL, histograms, `postgres_exporter`, alerting rules
- **Traces** — Jaeger spans, context propagation, trace analysis, metric correlation
- **Dashboards** — Grafana RED dashboard, USE dashboard, availability & SLO panels
- **Deep Dives** — Latency math, throughput & backpressure, saturation signals, availability engineering
- **Production Patterns** — Alert design, incident response, runbooks

---

### 3. Unified IoT Platform Guide

The original 20-chapter guide that kicked off this project — covering architecture through production deployment with dual Go (gRPC) and Python (FastAPI) code examples.

---

## 🛠️ How These Books Were Made

1. A **detailed requirements document** describing the system was provided as input.
2. The AI was prompted to generate a comprehensive, multi-part book covering development, operations, security, and DevOps.
3. Each chapter was generated as a standalone Markdown file, organized by part.

---

## 📄 License

This project is provided as-is for educational purposes.
