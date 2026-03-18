# Chapter 4: Demo Application Architecture

> **Part II — The Demo Application**

Before measuring anything, we need something worth measuring. This chapter introduces the demo system — an intentionally realistic order management backend — and explains every architectural decision in terms of what it lets us observe.

---

## 4.1 System Overview

```
                    ┌────────────────────────────────────┐
                    │  Browser / curl / k6 load tester  │
                    └───────────────┬────────────────────┘
                                    │ HTTP
                    ┌───────────────▼────────────────────┐
                    │  Frontend (HTML + Vanilla JS)      │
                    │  Static files served by Nginx      │
                    │  port :8080                        │
                    └───────────────┬────────────────────┘
                                    │ HTTP/JSON
                    ┌───────────────▼────────────────────┐
                    │   .NET 8 Minimal API               │
                    │   port :5000                       │
                    │                                    │
                    │   /api/orders     (CRUD)           │
                    │   /api/products   (CRUD)           │
                    │   /api/reports    (heavy query)    │
                    │   /metrics        (Prometheus)     │
                    │   /healthz        (liveness)       │
                    │   /readyz         (readiness)      │
                    └──────────┬─────────────────────────┘
                               │ SQL (Npgsql)
               ┌───────────────┼────────────────────┐
               │               │                    │
               ▼               ▼                    ▼
    ┌──────────────────┐  ┌─────────┐   ┌────────────────────┐
    │  PostgreSQL 16   │  │  Redis  │   │  External HTTP API │
    │  port :5432      │  │ (cache) │   │  (simulated slow)  │
    └──────────────────┘  └─────────┘   └────────────────────┘
              ▲
    ┌─────────┴────────┐
    │ postgres_exporter │   ← Exposes pg_* metrics to Prometheus
    │ port :9187        │
    └──────────────────┘

Observability layer (not in request path):
    Prometheus  :9090   ← Scrapes /metrics, stores time-series
    Jaeger      :16686  ← Receives traces from .NET via OTLP
    Grafana     :3000   ← Queries Prometheus, visualizes everything
```

---

## 4.2 Why This Application?

The demo is designed to exhibit **every class of performance problem** you'll encounter in real systems:

| Problem Class | How the Demo Demonstrates It |
|---|---|
| **Latency spike** | `/api/reports` runs an expensive aggregation query |
| **Throughput ceiling** | DB connection pool is intentionally small (max 10 connections) |
| **Saturation** | Load test fills the connection pool; new requests queue |
| **Cascading failure** | Reports endpoint's pool exhaustion starves the orders endpoint |
| **External dependency latency** | `POST /api/orders` calls a slow external API (simulated) |
| **Memory pressure** | An endpoint has a deliberate memory leak (configurable) |
| **Slow query** | A query that bypasses cache and hits an un-indexed column |

---

## 4.3 API Endpoints

### Orders API

| Method | Route | Description | Characteristics |
|---|---|---|---|
| `GET` | `/api/orders` | List all orders (paginated) | Fast, cached |
| `GET` | `/api/orders/{id}` | Get order by ID | Fast, indexed |
| `POST` | `/api/orders` | Create new order | Calls external API (slow) |
| `PUT` | `/api/orders/{id}` | Update order status | Simple UPDATE |
| `DELETE` | `/api/orders/{id}` | Delete order | Simple DELETE |

### Products API

| Method | Route | Description | Characteristics |
|---|---|---|---|
| `GET` | `/api/products` | List products | Cached, fast |
| `POST` | `/api/products` | Create product | Simple INSERT |

### Reports API

| Method | Route | Description | Characteristics |
|---|---|---|---|
| `GET` | `/api/reports/revenue` | Revenue by day, last 30 days | HEAVY: full table scan + grouping |
| `GET` | `/api/reports/top-products` | Top selling products | HEAVY: nested aggregation |

---

## 4.4 Database Schema

```sql
-- migrations/001_initial.sql

CREATE TABLE IF NOT EXISTS products (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    category    VARCHAR(100),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS orders (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_email VARCHAR(255) NOT NULL,
    status        VARCHAR(50)  NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','processing','shipped','delivered','cancelled')),
    total_amount  NUMERIC(10,2) NOT NULL DEFAULT 0,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS order_items (
    id         UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id   UUID         NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID         NOT NULL REFERENCES products(id),
    quantity   INTEGER      NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10,2) NOT NULL
);

-- Indexes for fast lookups
CREATE INDEX idx_orders_status     ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_order_items_order ON order_items(order_id);

-- Intentionally missing: idx_orders_customer_email
-- This makes customer-filtered queries do full scans, demonstrating the
-- latency difference observable in traces when this index is added/removed.

CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE orders
    SET total_amount = (
        SELECT COALESCE(SUM(quantity * unit_price), 0)
        FROM order_items WHERE order_id = NEW.order_id
    ), updated_at = NOW()
    WHERE id = NEW.order_id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER recalculate_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW EXECUTE FUNCTION update_order_total();

-- Seed data (1000 products, 10000 orders)
INSERT INTO products (name, price, category)
SELECT
    'Product ' || i,
    (random() * 100 + 1)::numeric(10,2),
    (ARRAY['electronics','clothing','food','books','sports'])[ceil(random()*5)::int]
FROM generate_series(1, 1000) AS i;

-- (Seeding orders would be done by seed script)
```

---

## 4.5 Project Structure

```
obs-demo/
├── src/
│   ├── Api/
│   │   ├── Api.csproj
│   │   ├── Program.cs              ← Minimal API entry point + all wiring
│   │   ├── Endpoints/
│   │   │   ├── OrderEndpoints.cs   ← Route handlers
│   │   │   ├── ProductEndpoints.cs
│   │   │   └── ReportEndpoints.cs
│   │   ├── Data/
│   │   │   ├── AppDbContext.cs     ← Npgsql / Dapper setup
│   │   │   └── Migrations/
│   │   ├── Services/
│   │   │   ├── OrderService.cs
│   │   │   └── ExternalApiClient.cs ← Simulated slow external call
│   │   ├── Observability/
│   │   │   ├── Metrics.cs           ← Custom metric definitions
│   │   │   └── Tracing.cs           ← Tracer configuration
│   │   └── appsettings.json
├── frontend/
│   └── index.html                   ← Single-page dashboard
├── deploy/
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── rules/
│   │       ├── alerts.yml
│   │       └── recording_rules.yml
│   ├── grafana/
│   │   ├── provisioning/
│   │   │   ├── datasources/
│   │   │   └── dashboards/
│   │   └── dashboards/
│   │       ├── red-dashboard.json
│   │       ├── use-dashboard.json
│   │       └── slo-dashboard.json
│   ├── postgres/
│   │   └── queries.yaml             ← postgres_exporter custom queries
│   └── jaeger/
│       └── config.yaml
├── scripts/
│   ├── seed.sql
│   └── load-test.js                 ← k6 load test script
└── README.md
```

---

## 4.6 Key Design Decisions

### Why .NET Minimal API?

Minimal API in .NET 8 has almost no magic — every route, every middleware, every dependency injection registration is visible in code. This makes observability easier to understand: you can see exactly *where* the instrumentation is added and *why*.

### Why PostgreSQL?

PostgreSQL has the richest observability built in of any relational database. The `pg_stat_*` family of views exposes everything: query statistics, lock waits, connection states, buffer hit rates, and more. `postgres_exporter` turns all of this into Prometheus metrics with a single Docker container.

### Why Vanilla JS Frontend?

The frontend is intentionally simple — no React, no build step. This keeps the focus on the backend observability. The frontend is just a thin wrapper that makes HTTP calls and shows the results, which is all we need to demonstrate the signals.

### Why Redis?

Redis adds a caching layer that creates an interesting observability question: "Is the slowness coming from the cache or from the database?" Traces will answer this clearly, and we'll build a panel showing cache hit rate.

### What "Slow External API" Simulates

The `POST /api/orders` endpoint calls an `ExternalApiClient` that adds configurable latency (default: 200ms, with 10% chance of 2000ms, 1% chance of timeout). This simulates the real-world pattern of third-party API dependency — perhaps a payment processor or shipping service. Traces will show this dependency clearly.
