# Chapter 5: Docker Compose — The Complete Local Stack

> **Part II — The Demo Application**

---

## 5.1 The Full Stack in One File

```yaml
# deploy/docker-compose.yml
version: "3.9"

networks:
  obs-net:
    driver: bridge

volumes:
  postgres_data:
  prometheus_data:
  grafana_data:
  jaeger_data:
  redis_data:

services:

  # ── Application ─────────────────────────────────────────────────────────────

  api:
    build:
      context: ../src/Api
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      ASPNETCORE_URLS:               "http://+:5000"
      ASPNETCORE_ENVIRONMENT:        "Development"
      ConnectionStrings__Postgres:   "Host=postgres;Port=5432;Database=obsdb;Username=obs;Password=obs123"
      ConnectionStrings__Redis:      "redis:6379"
      Observability__JaegerEndpoint: "http://jaeger:4318"
      Observability__ServiceName:    "obs-api"
      ExternalApi__BaseUrl:          "http://slowapi:4999"
      ExternalApi__SlowProbability:  "0.1"    # 10% chance of 2s response
      ExternalApi__TimeoutProbability: "0.01" # 1% chance of timeout
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_started }
      jaeger:   { condition: service_started }
    networks: [obs-net]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/healthz"]
      interval: 10s
      timeout: 5s
      retries: 5

  frontend:
    image: nginx:1.25-alpine
    ports:
      - "8080:80"
    volumes:
      - ../frontend:/usr/share/nginx/html:ro
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks: [obs-net]

  # Simulates a slow/unreliable external API
  slowapi:
    image: mendhak/http-https-echo:31
    ports:
      - "4999:4999"
    environment:
      HTTP_PORT: "4999"
    networks: [obs-net]

  # ── Data Layer ───────────────────────────────────────────────────────────────

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       obsdb
      POSTGRES_USER:     obs
      POSTGRES_PASSWORD: obs123
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/01_schema.sql:ro
      - ./postgres/seed.sql:/docker-entrypoint-initdb.d/02_seed.sql:ro
      - ./postgres/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    ports:
      - "5432:5432"
    networks: [obs-net]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U obs -d obsdb"]
      interval: 5s
      timeout: 3s
      retries: 10
    command: >
      postgres
      -c shared_preload_libraries=pg_stat_statements
      -c pg_stat_statements.track=all
      -c log_min_duration_statement=1000
      -c log_statement=ddl
      -c max_connections=50

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:v0.15.0
    environment:
      DATA_SOURCE_NAME: "postgresql://obs:obs123@postgres:5432/obsdb?sslmode=disable"
    ports:
      - "9187:9187"
    volumes:
      - ./postgres/queries.yaml:/etc/postgres_exporter/queries.yaml:ro
    command:
      - --config.file=/etc/postgres_exporter/queries.yaml
      - --collector.stat_statements
      - --collector.stat_bgwriter
      - --collector.stat_user_tables
      - --collector.locks
    depends_on: [postgres]
    networks: [obs-net]

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks: [obs-net]

  redis-exporter:
    image: oliver006/redis_exporter:v1.59.0
    environment:
      REDIS_ADDR: "redis://redis:6379"
    ports:
      - "9121:9121"
    depends_on: [redis]
    networks: [obs-net]

  # ── Observability Stack ──────────────────────────────────────────────────────

  prometheus:
    image: prom/prometheus:v2.52.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle          # Reload config without restart
      - --web.enable-remote-write-receiver
    networks: [obs-net]

  jaeger:
    image: jaegertracing/all-in-one:1.58
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
      - "14268:14268"   # Jaeger HTTP (legacy)
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
      SPAN_STORAGE_TYPE:      "memory"
      MEMORY_MAX_TRACES:      "100000"
    volumes:
      - jaeger_data:/tmp
    networks: [obs-net]

  grafana:
    image: grafana/grafana:11.0.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER:     admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP:     "false"
      GF_FEATURE_TOGGLES_ENABLE:  "traceqlEditor"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    depends_on: [prometheus, jaeger]
    networks: [obs-net]

  # ── Load Testing ──────────────────────────────────────────────────────────────

  # Run with: docker compose run k6 run /scripts/load-test.js
  k6:
    image: grafana/k6:0.50.0
    volumes:
      - ../scripts:/scripts:ro
    environment:
      K6_API_BASE_URL: "http://api:5000"
    networks: [obs-net]
    profiles: [loadtest]  # Only runs with: docker compose --profile loadtest run k6 ...
```

---

## 5.2 Prometheus Configuration

```yaml
# deploy/prometheus/prometheus.yml
global:
  scrape_interval:     15s     # Scrape every 15 seconds
  evaluation_interval: 15s     # Evaluate rules every 15 seconds
  scrape_timeout:      10s

# Load alert + recording rules
rule_files:
  - /etc/prometheus/rules/recording_rules.yml
  - /etc/prometheus/rules/alerts.yml

# Alert routing
alerting:
  alertmanagers:
    - static_configs:
        - targets: []   # Add alertmanager:9093 when needed

scrape_configs:

  # .NET API — exposes /metrics via prometheus-net
  - job_name: obs-api
    static_configs:
      - targets: [api:5000]
    metrics_path: /metrics
    scrape_interval: 5s    # More frequent for the application
    metric_relabel_configs:
      # Drop high-cardinality debug metrics
      - source_labels: [__name__]
        regex: "go_.*"
        action: drop

  # PostgreSQL
  - job_name: postgres
    static_configs:
      - targets: [postgres-exporter:9187]
    scrape_interval: 15s

  # Redis
  - job_name: redis
    static_configs:
      - targets: [redis-exporter:9121]
    scrape_interval: 15s

  # Prometheus itself
  - job_name: prometheus
    static_configs:
      - targets: [localhost:9090]
```

---

## 5.3 PostgreSQL Custom Queries for postgres_exporter

```yaml
# deploy/postgres/queries.yaml
# These extend postgres_exporter with application-relevant metrics

pg_database_size:
  query: |
    SELECT datname, pg_database_size(datname) AS bytes
    FROM pg_database
    WHERE datname NOT IN ('template0', 'template1')
  metrics:
    - datname:
        usage: LABEL
        description: Database name
    - bytes:
        usage: GAUGE
        description: Database size in bytes

pg_slow_queries:
  query: |
    SELECT
      calls,
      total_exec_time / calls AS avg_ms,
      stddev_exec_time AS stddev_ms,
      rows / calls AS avg_rows,
      LEFT(query, 100) AS query_snippet
    FROM pg_stat_statements
    WHERE calls > 100
    ORDER BY total_exec_time DESC
    LIMIT 20
  metrics:
    - calls:
        usage: GAUGE
        description: Number of times this query has been called
    - avg_ms:
        usage: GAUGE
        description: Average execution time in milliseconds

pg_table_bloat:
  query: |
    SELECT
      schemaname,
      tablename,
      pg_total_relation_size(schemaname||'.'||tablename) AS total_bytes,
      n_live_tup AS live_rows,
      n_dead_tup AS dead_rows,
      CASE WHEN n_live_tup > 0
           THEN n_dead_tup::float / n_live_tup::float
           ELSE 0 END AS bloat_ratio
    FROM pg_stat_user_tables
    WHERE schemaname = 'public'
  metrics:
    - schemaname: { usage: LABEL }
    - tablename:  { usage: LABEL }
    - total_bytes:
        usage: GAUGE
        description: Total table size including indexes
    - live_rows:
        usage: GAUGE
        description: Live rows
    - dead_rows:
        usage: GAUGE
        description: Dead rows (not yet vacuumed)
    - bloat_ratio:
        usage: GAUGE
        description: Ratio of dead to live rows

pg_lock_waits:
  query: |
    SELECT
      count(*) FILTER (WHERE NOT granted) AS waiting,
      count(*) FILTER (WHERE granted) AS held
    FROM pg_locks
    WHERE locktype = 'relation'
  metrics:
    - waiting:
        usage: GAUGE
        description: Number of lock wait events currently happening
    - held:
        usage: GAUGE
        description: Number of locks currently held

pg_connection_states:
  query: |
    SELECT
      state,
      count(*) AS count
    FROM pg_stat_activity
    WHERE datname = 'obsdb'
    GROUP BY state
  metrics:
    - state: { usage: LABEL }
    - count:
        usage: GAUGE
        description: Number of connections in each state
```

---

## 5.4 Grafana Auto-Provisioning

```yaml
# deploy/grafana/provisioning/datasources/datasources.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: jaeger   # Links metric exemplars to Jaeger traces

  - name: Jaeger
    type: jaeger
    uid: jaeger
    access: proxy
    url: http://jaeger:16686
    jsonData:
      tracesToMetrics:
        datasourceUid: prometheus    # Links traces back to metrics
        tags:
          - key: service.name
            value: service
```

```yaml
# deploy/grafana/provisioning/dashboards/dashboard.yaml
apiVersion: 1

providers:
  - name: obs-demo
    type: file
    disableDeletion: true
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

---

## 5.5 Quick Start Commands

```bash
# Start everything
docker compose up -d

# Wait for all services to be healthy
docker compose ps

# Seed test data (if not auto-seeded)
docker compose exec postgres psql -U obs -d obsdb -f /docker-entrypoint-initdb.d/02_seed.sql

# Watch logs from the API
docker compose logs -f api

# Run the load test (1 minute, 20 virtual users)
docker compose --profile loadtest run k6 run \
  --vus 20 --duration 60s \
  /scripts/load-test.js

# Stop everything
docker compose down

# Stop and remove volumes (clean slate)
docker compose down -v

# Access URLs:
# API:        http://localhost:5000/api/orders
# Frontend:   http://localhost:8080
# Prometheus: http://localhost:9090
# Jaeger:     http://localhost:16686
# Grafana:    http://localhost:3000  (admin/admin)
```

---

## 5.6 Load Test Script

```javascript
// scripts/load-test.js
// Run with: k6 run --vus 20 --duration 2m scripts/load-test.js

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const BASE = __ENV.K6_API_BASE_URL || 'http://localhost:5000';

// Custom k6 metrics
const provisionErrors = new Rate('provision_errors');
const reportLatency   = new Trend('report_latency_ms');

export const options = {
  stages: [
    { duration: '30s', target: 10  }, // Ramp up to 10 VUs
    { duration: '60s', target: 50  }, // Hold at 50 VUs (normal load)
    { duration: '30s', target: 100 }, // Spike to 100 VUs
    { duration: '30s', target: 10  }, // Back down
    { duration: '30s', target: 0   }, // Wind down
  ],
  thresholds: {
    // Our SLO targets
    http_req_duration:   ['p(95)<500'],   // P95 < 500ms
    http_req_failed:     ['rate<0.01'],   // Error rate < 1%
    report_latency_ms:   ['p(95)<5000'],  // Reports may be slower
  },
};

// Helper: random item from array
const pick = arr => arr[Math.floor(Math.random() * arr.length)];

const statuses = ['pending', 'processing', 'shipped', 'delivered'];

export default function () {
  const scenario = Math.random();

  if (scenario < 0.5) {
    // 50%: List orders (read-heavy, tests pagination)
    const res = http.get(`${BASE}/api/orders?page=1&pageSize=20`, {
      tags: { route: '/api/orders' },
    });
    check(res, { 'orders list 200': r => r.status === 200 });

  } else if (scenario < 0.7) {
    // 20%: Create an order (write path, calls external API)
    const payload = JSON.stringify({
      customerEmail: `user${Math.floor(Math.random()*1000)}@test.com`,
      items: [
        { productId: null, quantity: Math.ceil(Math.random()*5) }
      ]
    });
    const res = http.post(`${BASE}/api/orders`, payload, {
      headers: { 'Content-Type': 'application/json' },
      tags: { route: 'POST /api/orders' },
    });
    provisionErrors.add(res.status >= 500);
    check(res, { 'order created 201': r => r.status === 201 });

  } else if (scenario < 0.9) {
    // 20%: Get single order (read by ID, tests index)
    const id = Math.floor(Math.random() * 10000) + 1;
    const res = http.get(`${BASE}/api/orders/${id}`, {
      tags: { route: '/api/orders/{id}' },
    });
    // 404 is expected for non-existent IDs — not an error
    check(res, { 'status not 5xx': r => r.status < 500 });

  } else {
    // 10%: Revenue report (heavy query, tests saturation)
    const start = Date.now();
    const res = http.get(`${BASE}/api/reports/revenue`, {
      timeout: '10s',
      tags: { route: '/api/reports/revenue' },
    });
    reportLatency.add(Date.now() - start);
    check(res, { 'report 200': r => r.status === 200 });
  }

  sleep(Math.random() * 0.5 + 0.1); // 0.1–0.6s think time
}

export function handleSummary(data) {
  console.log('\n=== Test Summary ===');
  console.log('P50 latency:', data.metrics.http_req_duration.values['p(50)'].toFixed(2), 'ms');
  console.log('P95 latency:', data.metrics.http_req_duration.values['p(95)'].toFixed(2), 'ms');
  console.log('P99 latency:', data.metrics.http_req_duration.values['p(99)'].toFixed(2), 'ms');
  console.log('Error rate:', (data.metrics.http_req_failed.values.rate * 100).toFixed(3), '%');
  console.log('RPS:', data.metrics.http_reqs.values.rate.toFixed(1));
}
```
