# Chapter 13: Multi-Provider Integration Patterns

> *"The abstraction is only as good as its handling of the differences that can't be abstracted."*

---

## 13.1 Introduction

The Provider Abstraction Layer (Chapter 5) gave us a unified interface. This chapter tackles the **hard problems** that arise when operating across fundamentally different providers simultaneously — credential management, API rate limiting, provider-specific quirks, feature parity gaps, and cross-provider consistency.

---

## 13.2 Provider Capability Matrix

Not every provider supports every feature identically:

| Capability | AWS IoT Core | Azure IoT Hub | Mosquitto |
|-----------|-------------|---------------|-----------|
| Device creation | ✅ Thing | ✅ Device Identity | ✅ Client ID |
| Certificate generation | ✅ Native | ⚠️ Via DPS | ✅ OpenSSL/CA |
| Policy attachment | ✅ IoT Policy | ✅ Shared Access Policy | ✅ ACL file |
| Device suspension | ⚠️ Detach principals | ✅ Disable device | ✅ Remove ACL |
| Routing rules | ✅ IoT Rules Engine | ✅ Message Routing | ⚠️ Bridge config |
| Device twins/shadows | ✅ Device Shadow | ✅ Device Twin | ❌ N/A |
| Batch operations | ✅ Bulk registration | ✅ Import/Export | ❌ Sequential |
| Rate limits | 15 TPS (us-east-1) | 100 ops/sec (S1) | No limit (self-hosted) |

### Handling Feature Gaps

```python
# app/providers/base.py — Feature capability declaration

class ProviderCapabilities:
    """Declares what a provider supports."""
    supports_batch_creation: bool = False
    supports_device_shadow: bool = False
    supports_routing_rules: bool = False
    supports_native_certs: bool = False
    max_batch_size: int = 1
    rate_limit_per_second: int = 0  # 0 = unlimited


class IoTProvider(ABC):
    @property
    @abstractmethod
    def capabilities(self) -> ProviderCapabilities: ...
```

```python
# app/providers/aws_provider.py
class AwsIoTProvider(IoTProvider):
    @property
    def capabilities(self) -> ProviderCapabilities:
        return ProviderCapabilities(
            supports_batch_creation=True,
            supports_device_shadow=True,
            supports_routing_rules=True,
            supports_native_certs=True,
            max_batch_size=100,
            rate_limit_per_second=15,
        )
```

---

## 13.3 Rate Limiting Provider API Calls

Each provider has different rate limits. We use a per-provider, per-tenant rate limiter:

🐍 **Python — Token bucket rate limiter**:

```python
# app/providers/rate_limiter.py
import asyncio
import time
from redis.asyncio import Redis


class ProviderRateLimiter:
    """Token bucket rate limiter backed by Redis."""

    def __init__(self, redis: Redis):
        self.redis = redis

    async def acquire(self, provider: str, tenant_id: str, max_rate: int):
        """Wait until a token is available."""
        key = f"rate:{provider}:{tenant_id}"

        while True:
            # Lua script for atomic token bucket
            tokens = await self.redis.eval(
                """
                local key = KEYS[1]
                local max_rate = tonumber(ARGV[1])
                local now = tonumber(ARGV[2])

                local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
                local tokens = tonumber(bucket[1]) or max_rate
                local last_refill = tonumber(bucket[2]) or now

                -- Refill tokens based on elapsed time
                local elapsed = now - last_refill
                tokens = math.min(max_rate, tokens + elapsed * max_rate)

                if tokens >= 1 then
                    tokens = tokens - 1
                    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
                    redis.call('EXPIRE', key, 60)
                    return 1
                end

                return 0
                """,
                1, key, max_rate, time.time(),
            )

            if tokens == 1:
                return  # Token acquired
            await asyncio.sleep(0.1)  # Wait and retry
```

🔵 **Go — Rate limiter middleware**:

```go
// internal/provider/ratelimit.go
package provider

import (
    "context"
    "time"

    "golang.org/x/time/rate"
)

type RateLimitedProvider struct {
    inner   Provider
    limiter *rate.Limiter
}

func NewRateLimitedProvider(inner Provider, rps int) *RateLimitedProvider {
    return &RateLimitedProvider{
        inner:   inner,
        limiter: rate.NewLimiter(rate.Limit(rps), rps), // Allow burst = rps
    }
}

func (p *RateLimitedProvider) CreateDevice(ctx context.Context, req CreateDeviceRequest) (*DeviceResult, error) {
    if err := p.limiter.Wait(ctx); err != nil {
        return nil, fmt.Errorf("rate limit: %w", err)
    }
    return p.inner.CreateDevice(ctx, req)
}

// ... wrap every Provider method similarly
```

---

## 13.4 Credential Lifecycle Per Provider

| Provider | Auth Method | Credential Lifetime | Rotation |
|----------|-----------|-------------------|----------|
| **AWS** | STS AssumeRole | 1 hour (session) | Automatic — re-assume on expiry |
| **Azure** | Service Principal | Client secret: 2 years | Manual rotation via Vault |
| **Mosquitto** | Username/password | Unlimited | Manual rotation via Vault |

```python
# app/providers/aws_provider.py (enhanced)

class AwsIoTProvider(IoTProvider):
    def __init__(self, role_arn, external_id, region):
        self._role_arn = role_arn
        self._external_id = external_id
        self._region = region
        self._client = None
        self._credentials_expiry = None

    def _ensure_client(self):
        """Re-assume role if credentials are expired or about to expire."""
        if self._client and self._credentials_expiry:
            if datetime.utcnow() < self._credentials_expiry - timedelta(minutes=5):
                return  # Still valid

        sts = boto3.client("sts")
        result = sts.assume_role(
            RoleArn=self._role_arn,
            ExternalId=self._external_id,
            RoleSessionName="iot-platform",
            DurationSeconds=3600,
        )

        creds = result["Credentials"]
        self._credentials_expiry = creds["Expiration"].replace(tzinfo=None)

        self._client = boto3.client(
            "iot",
            region_name=self._region,
            aws_access_key_id=creds["AccessKeyId"],
            aws_secret_access_key=creds["SecretAccessKey"],
            aws_session_token=creds["SessionToken"],
        )
```

---

## 13.5 Cross-Provider Consistency Checks

A background reconciliation job ensures platform state matches provider state:

```python
# app/workers/reconciler.py

class ProviderReconciler:
    """Detect drift between platform database and provider state."""

    async def reconcile_tenant(self, tenant_id: str):
        async with self.db.acquire() as conn:
            devices = await conn.fetch(
                "SELECT * FROM devices WHERE tenant_id = $1 AND status = 'active'",
                tenant_id,
            )

        provider = await self.providers.get_provider(
            await self._get_integration(tenant_id)
        )

        for device in devices:
            try:
                # Check if device exists at provider
                exists = await provider.device_exists(device["provider_device_id"])
                if not exists:
                    await self._handle_drift(
                        tenant_id, device,
                        drift_type="device_missing_at_provider"
                    )
            except CredentialsExpiredError:
                await self._handle_drift(
                    tenant_id, device,
                    drift_type="credentials_expired"
                )
                break  # All devices for this tenant will fail

    async def _handle_drift(self, tenant_id, device, drift_type):
        """Log drift and optionally auto-remediate."""
        await self.events.publish_audit_event(
            tenant_id=tenant_id,
            action="reconciliation.drift_detected",
            actor="system",
            resource_type="device",
            resource_id=str(device["id"]),
            outcome="warning",
            details={"drift_type": drift_type},
        )
```

---

## 13.6 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Capability matrix** | Not all providers support all features — declare capabilities |
| **Rate limiting** | Per-provider, per-tenant token bucket prevents API throttling |
| **Credential lifecycle** | AWS = auto-refreshing sessions; Azure/Mosquitto = Vault rotation |
| **Reconciliation** | Background job detects drift between platform and provider state |

### What's Next

**Chapter 14** covers **Observability & Monitoring** — metrics, traces, logs, and dashboards for operating the platform.
