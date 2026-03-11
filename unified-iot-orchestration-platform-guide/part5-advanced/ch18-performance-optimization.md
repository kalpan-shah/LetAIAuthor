# Chapter 18: Performance Optimization

> *"Premature optimization is the root of all evil, but mature optimization is the foundation of scale."*

---

## 18.1 Introduction

An IoT control plane has a unique performance profile: relatively low request volume (compared to a consumer API), but each request often involves multiple external calls to cloud providers. Optimization here is about **reducing provider API latency** and **maximizing throughput** under concurrent tenant operations.

---

## 18.2 Performance Bottleneck Analysis

| Operation | Typical Latency | Bottleneck | Optimization |
|-----------|----------------|-----------|-------------|
| Device provisioning (full) | 2-5 seconds | Provider API calls (3-5 calls) | Parallelize independent calls |
| Tenant CRUD | 5-20 ms | Database | Connection pooling, prepared statements |
| Certificate issuance (Vault) | 50-200 ms | Vault PKI sign | Vault performance tuning |
| Provider endpoint lookup | 200-500 ms | Provider API | Cache with TTL |
| Quota check | 1-5 ms | Redis | Already fast — ensure cache hit rate |
| Audit event publish | < 1 ms | NATS | Async — no impact on request path |
| Policy generation | < 1 ms | CPU | Negligible — compute-only |

---

## 18.3 Key Optimizations

### 1. Parallelize Provider API Calls

During provisioning, some calls are independent and can run concurrently:

🐍 **Python — Parallel provisioning steps**:

```python
import asyncio

async def provision_device_optimized(self, tenant_id, device_id):
    # Step 1: Create device at provider (must be first)
    provider_result = await provider.create_device(req)

    # Steps 2-3: Create cert AND generate policy IN PARALLEL
    cert_task = asyncio.create_task(provider.create_certificate(cert_req))
    policy_task = asyncio.create_task(provider.create_policy(policy_req))
    cert_result, policy_result = await asyncio.gather(cert_task, policy_task)

    # Steps 4-5: Attach cert AND attach policy IN PARALLEL
    await asyncio.gather(
        provider.attach_certificate_to_device(cert_result.provider_cert_id, provider_result.provider_device_id),
        provider.attach_policy_to_certificate(policy_result.provider_policy_id, cert_result.provider_cert_id),
    )

    # Step 6: Database write (single transaction)
    # ...
```

**Impact**: Reduces provisioning from 5 sequential calls (~2.5s) to 3 sequential stages (~1.5s) — a **40% improvement**.

🔵 **Go — errgroup for parallel calls**:

```go
import "golang.org/x/sync/errgroup"

func (s *Service) ProvisionOptimized(ctx context.Context, ...) error {
    // Stage 1: Create device
    deviceResult, err := provider.CreateDevice(ctx, req)
    if err != nil {
        return err
    }

    // Stage 2: Parallel cert + policy creation
    g, gctx := errgroup.WithContext(ctx)
    var certResult *CertificateResult
    var policyResult *PolicyResult

    g.Go(func() error {
        var err error
        certResult, err = provider.CreateCertificate(gctx, certReq)
        return err
    })
    g.Go(func() error {
        var err error
        policyResult, err = provider.CreatePolicy(gctx, policyReq)
        return err
    })

    if err := g.Wait(); err != nil {
        return err
    }

    // Stage 3: Parallel attachments
    // ...
}
```

### 2. Caching Provider Endpoints

Provider endpoints rarely change — cache them:

```python
# app/providers/cache.py
from redis.asyncio import Redis
import json


class ProviderCache:
    def __init__(self, redis: Redis, default_ttl: int = 3600):
        self.redis = redis
        self.ttl = default_ttl

    async def get_endpoint(self, integration_id: str) -> str | None:
        return await self.redis.get(f"endpoint:{integration_id}")

    async def set_endpoint(self, integration_id: str, endpoint: str):
        await self.redis.set(f"endpoint:{integration_id}", endpoint, ex=self.ttl)

    async def get_or_fetch(self, integration_id: str, fetch_fn) -> str:
        cached = await self.get_endpoint(integration_id)
        if cached:
            return cached.decode()

        endpoint = await fetch_fn()
        await self.set_endpoint(integration_id, endpoint)
        return endpoint
```

### 3. Database Query Optimization

```sql
-- Before: N+1 queries for device listing with certs
SELECT * FROM devices WHERE tenant_id = $1;
-- Then for each device:
SELECT * FROM certificates WHERE device_id = $1 AND status = 'active';

-- After: Single join query
SELECT
    d.*,
    c.id as cert_id,
    c.fingerprint,
    c.status as cert_status,
    c.expires_at as cert_expires
FROM devices d
LEFT JOIN certificates c ON d.active_cert_id = c.id
WHERE d.tenant_id = $1
ORDER BY d.created_at DESC
LIMIT $2 OFFSET $3;
```

### 4. Connection Pool Warm-up

```go
// Warm up the connection pool on startup to avoid cold-start latency
func WarmPool(ctx context.Context, pool *pgxpool.Pool, minConns int) {
    conns := make([]*pgxpool.Conn, 0, minConns)
    for i := 0; i < minConns; i++ {
        conn, err := pool.Acquire(ctx)
        if err != nil {
            break
        }
        conns = append(conns, conn)
    }
    // Release all connections back to pool
    for _, c := range conns {
        c.Release()
    }
}
```

### 5. Batch Operations

```python
# Instead of provisioning one device at a time in a loop:
for device in devices:
    await provision(device)  # Sequential — 100 devices = 100 * 2s = 200s

# Use concurrent batch with semaphore:
sem = asyncio.Semaphore(10)
async def bounded_provision(device):
    async with sem:
        return await provision(device)

results = await asyncio.gather(*[bounded_provision(d) for d in devices])
# 100 devices with concurrency 10 = ~20s
```

---

## 18.4 Benchmarking

```go
// benchmark_test.go
func BenchmarkDeviceProvisioning(b *testing.B) {
    // Setup mock provider
    provider := mocks.NewMockProvider()
    service := NewDeviceService(testPool, provider, quotas, events)

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            i++
            _, err := service.ProvisionDevice(context.Background(),
                "test-tenant", fmt.Sprintf("bench-device-%d", i))
            if err != nil {
                b.Fatal(err)
            }
        }
    })
}
```

---

## 18.5 Chapter Summary

| Optimization | Impact | Effort |
|-------------|--------|--------|
| **Parallelize provider calls** | 40% latency reduction | Medium |
| **Cache endpoints** | Eliminate redundant API calls | Low |
| **Optimize queries** | Reduce N+1 patterns | Low |
| **Pool warm-up** | Eliminate cold-start latency | Low |
| **Batch with semaphore** | 10x throughput for batch ops | Medium |

### What's Next

**Chapter 19** covers **Disaster Recovery & High Availability** — ensuring the platform survives failures.
