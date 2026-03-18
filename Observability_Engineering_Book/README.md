# Observability Engineering
## Measuring Latency, Throughput, Saturation & Availability
### A Hands-On Guide with .NET, Prometheus, Jaeger & Grafana

---

### What This Book Is

A ground-up, practical guide to **understanding what your systems are actually doing** — not just whether they're running, but *how well* they're running. You will build a complete observability stack from scratch, instrument a real .NET 8 Minimal API backed by PostgreSQL, collect metrics with Prometheus, trace requests with Jaeger, and build Grafana dashboards that answer the questions that matter when things go wrong at 2 AM.

Every chapter includes runnable code. Every concept is demonstrated with the actual tool you'll use in production.

---

### The Demo System

```
Browser / curl
    │  HTTP
    ▼
Frontend (HTML + vanilla JS)
    │  HTTP / fetch
    ▼
.NET 8 Minimal API  ←── Prometheus scrapes /metrics
    │  SQL (Npgsql)     OpenTelemetry → Jaeger (traces)
    ▼
PostgreSQL 16
    ▲
postgres_exporter ──► Prometheus

All visualised in Grafana.
Everything runs on a single laptop with Docker Compose.
```

---

### Tech Stack

| Layer | Tool | Version |
|---|---|---|
| Application | .NET Minimal API | .NET 8 |
| Database | PostgreSQL | 16 |
| Metrics collection | Prometheus | 2.52 |
| Tracing | Jaeger | 1.58 |
| Visualization | Grafana | 11.0 |
| Instrumentation | OpenTelemetry .NET | 1.8 |
| DB metrics | postgres_exporter | 0.15 |
| Load generation | k6 | 0.50 |
| Container runtime | Docker Compose | v2 |

---

### Book Structure

| Part | Chapters | What You Learn |
|---|---|---|
| **I — Foundations** | 1–3 | The four signals, RED/USE methods, why percentiles beat averages |
| **II — Demo Application** | 4–6 | Architecture, Docker Compose, OpenTelemetry wiring |
| **III — Metrics: Prometheus** | 7–10 | PromQL, histograms, postgres_exporter, alerting rules |
| **IV — Traces: Jaeger** | 11–14 | Spans, context propagation, trace analysis, correlating with metrics |
| **V — Grafana Dashboards** | 15–18 | RED dashboard, USE dashboard, availability & SLO panels |
| **VI — Deep Dives** | 19–22 | Latency math, throughput & backpressure, saturation signals, availability engineering |
| **VII — Production Patterns** | 23–25 | Alert design, incident response, runbooks |

---

### Converting to PDF

```bash
# Single chapter
pandoc Chapter_01.md -o Chapter_01.pdf \
  --pdf-engine=xelatex -V geometry:margin=1in \
  --highlight-style=tango --toc

# Full book
pandoc $(find . -name "Chapter_*.md" | sort) \
  -o Observability_Engineering.pdf \
  --pdf-engine=xelatex --toc --toc-depth=2 \
  -V geometry:margin=1in -V fontsize=11pt \
  --highlight-style=tango
```

---

### Quick Start

```bash
git clone https://github.com/your-org/obs-demo
cd obs-demo
docker compose up -d
# API:        http://localhost:5000
# Prometheus: http://localhost:9090
# Jaeger:     http://localhost:16686
# Grafana:    http://localhost:3000  (admin/admin)
```
