# Chapter 11: Provisioning Workflows

> *"Device onboarding is where security meets usability. Make it too hard and devices ship with default passwords. Make it too easy and you've got an open door."*

---

## 11.1 Introduction

Not every device is provisioned the same way. A factory-floor sensor gets a certificate burned into firmware before shipping. A field-deployed gateway claims its identity on first boot. An engineer manually pairs a prototype from their laptop.

This chapter covers four provisioning strategies, all orchestrated through the platform.

---

## 11.2 Provisioning Strategies

| Strategy | Use Case | Security Level | Automation |
|----------|----------|---------------|-----------|
| **Factory Provisioning** | Mass production | ★★★★★ | Full |
| **Claim-Based** | Field deployment with pre-shared tokens | ★★★★ | Semi |
| **Manual Pairing** | Prototyping, one-off devices | ★★★ | Manual |
| **OTA Credential Provisioning** | Brownfield devices, migration | ★★★★ | Full |

---

## 11.3 Factory Provisioning

```
 Manufacturing Line          Platform               Cloud Provider
       │                        │                        │
       │  Request batch (100)   │                        │
       │───────────────────────▶│                        │
       │                        │  Create 100 things     │
       │                        │───────────────────────▶│
       │                        │  100 ARNs              │
       │                        │◀───────────────────────│
       │                        │  Create 100 certs      │
       │                        │───────────────────────▶│
       │                        │  100 cert+key pairs    │
       │                        │◀───────────────────────│
       │                        │                        │
       │  Batch response:       │                        │
       │  [{device_id, cert,    │                        │
       │    key, endpoint}...]  │                        │
       │◀───────────────────────│                        │
       │                        │                        │
   Flash certs to firmware      │                        │
```

🐍 **Python — Batch provisioning endpoint**:

```python
# app/routers/provisioning.py
from fastapi import APIRouter, Request
from pydantic import BaseModel

router = APIRouter()


class BatchProvisionRequest(BaseModel):
    tenant_id: str
    device_prefix: str          # e.g., "sensor" → sensor-001, sensor-002
    count: int                  # Number of devices
    metadata: dict = {}


class DeviceCredentials(BaseModel):
    device_id: str
    device_name: str
    cert_pem: str
    private_key_pem: str
    ca_pem: str | None
    endpoint: str


@router.post("/batch", response_model=list[DeviceCredentials])
async def batch_provision(body: BatchProvisionRequest, request: Request):
    """Factory provisioning: create and provision multiple devices."""
    service = request.app.state.provisioning_service
    return await service.batch_provision(
        tenant_id=body.tenant_id,
        prefix=body.device_prefix,
        count=body.count,
        metadata=body.metadata,
    )
```

```python
# app/services/provisioning_service.py
import asyncio


class ProvisioningService:
    def __init__(self, device_service, cert_service, events):
        self.devices = device_service
        self.certs = cert_service
        self.events = events

    async def batch_provision(
        self, tenant_id: str, prefix: str, count: int, metadata: dict
    ) -> list[dict]:
        """Create and provision N devices concurrently (with concurrency limit)."""
        semaphore = asyncio.Semaphore(10)  # Max 10 concurrent provider calls

        async def provision_one(index: int):
            name = f"{prefix}-{index:04d}"
            async with semaphore:
                device = await self.devices.create_device(tenant_id, name, metadata)
                result = await self.devices.provision_device(tenant_id, str(device["id"]))
                return {
                    "device_id": str(device["id"]),
                    "device_name": name,
                    "cert_pem": result["certificate"]["cert_pem"],
                    "private_key_pem": result["certificate"]["private_key_pem"],
                    "ca_pem": result["certificate"].get("ca_pem"),
                    "endpoint": result["endpoint"],
                }

        tasks = [provision_one(i) for i in range(1, count + 1)]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Separate successes and failures
        successes = [r for r in results if isinstance(r, dict)]
        failures = [r for r in results if isinstance(r, Exception)]

        if failures:
            await self.events.publish_audit_event(
                tenant_id=tenant_id,
                action="batch_provision.partial_failure",
                actor="api",
                resource_type="batch",
                resource_id=prefix,
                outcome="partial",
                details={
                    "requested": count,
                    "succeeded": len(successes),
                    "failed": len(failures),
                    "errors": [str(e) for e in failures[:5]],
                },
            )

        return successes
```

---

## 11.4 Claim-Based Provisioning

Devices ship with a **claim token** embedded in firmware. On first boot, the device presents the token to claim its identity:

```
Device (first boot)          Platform                    Provider
       │                        │                           │
       │  POST /claim           │                           │
       │  {claim_token: "xyz"}  │                           │
       │───────────────────────▶│                           │
       │                        │                           │
       │                   Validate token                   │
       │                   (Redis lookup)                   │
       │                        │                           │
       │                   Create device                    │
       │                        │──────────────────────────▶│
       │                   Create cert                      │
       │                        │──────────────────────────▶│
       │                   Attach policy                    │
       │                        │──────────────────────────▶│
       │                        │                           │
       │  {cert, key, endpoint} │                           │
       │◀───────────────────────│                           │
       │                        │                           │
    Store cert in flash         │                           │
    Connect to provider ────────────────────────────────────▶
```

```python
# app/services/claim_service.py
import secrets
import json
from redis.asyncio import Redis


class ClaimService:
    def __init__(self, redis: Redis, provisioning_service, events):
        self.redis = redis
        self.provisioning = provisioning_service
        self.events = events

    async def create_claim_tokens(
        self, tenant_id: str, device_prefix: str, count: int, ttl_hours: int = 720
    ) -> list[dict]:
        """Generate claim tokens for a batch of devices."""
        tokens = []
        for i in range(1, count + 1):
            token = secrets.token_urlsafe(32)
            device_name = f"{device_prefix}-{i:04d}"

            await self.redis.set(
                f"claim:{token}",
                json.dumps({
                    "tenant_id": tenant_id,
                    "device_name": device_name,
                    "claimed": False,
                }),
                ex=ttl_hours * 3600,
            )

            tokens.append({"device_name": device_name, "claim_token": token})

        return tokens

    async def claim_device(self, claim_token: str) -> dict:
        """Exchange a claim token for device credentials."""
        # 1. Validate token
        claim_data = await self.redis.get(f"claim:{claim_token}")
        if not claim_data:
            raise InvalidClaimTokenError("Token not found or expired")

        claim = json.loads(claim_data)
        if claim["claimed"]:
            raise InvalidClaimTokenError("Token already claimed")

        # 2. Mark as claimed (atomically)
        claim["claimed"] = True
        await self.redis.set(f"claim:{claim_token}", json.dumps(claim), keepttl=True)

        # 3. Provision the device
        device = await self.provisioning.devices.create_device(
            claim["tenant_id"], claim["device_name"]
        )
        result = await self.provisioning.devices.provision_device(
            claim["tenant_id"], str(device["id"])
        )

        # 4. Clean up token
        await self.redis.delete(f"claim:{claim_token}")

        return {
            "device_id": str(device["id"]),
            "cert_pem": result["certificate"]["cert_pem"],
            "private_key_pem": result["certificate"]["private_key_pem"],
            "endpoint": result["endpoint"],
        }
```

---

## 11.5 Manual Pairing

For development and prototyping — an engineer creates a device via API or CLI:

```bash
# Using the CLI
iot-platform device create \
    --tenant acme \
    --name "prototype-sensor-01" \
    --metadata '{"type": "temperature", "location": "lab-bench"}'

# Then provision it
iot-platform device provision \
    --tenant acme \
    --device-id "uuid-of-device" \
    --output-certs ./certs/
```

---

## 11.6 Chapter Summary

| Strategy | Best For | Automation |
|----------|---------|-----------|
| **Factory** | Mass production, batch creates (100s-1000s) | Full |
| **Claim-based** | Field deployment, token-per-device | Semi |
| **Manual** | Prototyping, development | Manual |
| **OTA** | Brownfield migration, credential refreshing | Full |

### What's Next

**Chapter 12** covers **Telemetry Routing Configuration** — defining how device data flows from the broker to analytics, databases, and alerting systems.

---

# Chapter 12: Telemetry Routing Configuration

> *"We don't touch the data — we just tell the provider where to send it."*

---

## 12.1 Introduction

While our platform never processes telemetry data directly, it **configures** how telemetry flows at the provider level. Routing rules define: "When a message arrives on topic X, send it to destination Y."

This is the difference between a data plane (handling the data) and a control plane (configuring where data goes).

---

## 12.2 Routing Rule Model

```python
# app/models/routing_rule.py
from pydantic import BaseModel, Field
from enum import Enum


class DestinationType(str, Enum):
    KINESIS = "kinesis"          # AWS Kinesis Data Stream
    EVENTHUB = "eventhub"        # Azure Event Hub
    S3 = "s3"                    # AWS S3 Bucket
    BLOB = "blob"                # Azure Blob Storage
    WEBHOOK = "webhook"          # HTTP POST to endpoint
    LAMBDA = "lambda"            # AWS Lambda function
    FUNCTION = "function"        # Azure Function
    DATABASE = "database"        # Database insert
    SNS = "sns"                  # AWS SNS Topic


class RoutingDestination(BaseModel):
    type: DestinationType
    config: dict = Field(..., description="Provider-specific destination config")
    # Example config for Kinesis: {"stream_name": "telemetry-stream", "partition_key": "device_id"}
    # Example config for webhook: {"url": "https://hooks.example.com/telemetry", "headers": {...}}


class CreateRoutingRuleRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=255)
    description: str | None = None
    source_topic: str = Field(..., description="MQTT topic pattern, e.g., tenant/*/device/*/telemetry")
    destination: RoutingDestination
    filter_sql: str | None = Field(None, description="SQL-like filter, e.g., SELECT * WHERE temperature > 50")
    transform: dict | None = Field(None, description="Field transformations")
    enabled: bool = True
```

---

## 12.3 Routing Service

```python
# app/services/routing_service.py
from asyncpg import Pool
from app.providers.factory import ProviderFactory
from app.services.quota_service import QuotaService


class RoutingService:
    def __init__(self, db: Pool, providers: ProviderFactory, quotas: QuotaService, events):
        self.db = db
        self.providers = providers
        self.quotas = quotas
        self.events = events

    async def create_rule(self, tenant_id: str, rule: dict) -> dict:
        """Create a routing rule — persists locally and deploys to provider."""
        # Check quota
        if not await self.quotas.check_quota(tenant_id, "routing_rule"):
            raise QuotaExceededException("Routing rule limit reached")

        # Resolve the topic pattern to include tenant scope
        scoped_topic = self._scope_topic(tenant_id, rule["source_topic"])

        # Persist in database
        async with self.db.acquire() as conn:
            row = await conn.fetchrow(
                """INSERT INTO routing_rules
                   (tenant_id, name, description, source_topic,
                    destination, filter_sql, transform, status)
                   VALUES ($1, $2, $3, $4, $5, $6, $7, 'draft')
                   RETURNING *""",
                tenant_id, rule["name"], rule.get("description"),
                scoped_topic, json.dumps(rule["destination"]),
                rule.get("filter_sql"), json.dumps(rule.get("transform")),
            )

        return dict(row)

    async def deploy_rule(self, tenant_id: str, rule_id: str) -> dict:
        """Deploy a draft rule to the provider's routing engine."""
        async with self.db.acquire() as conn:
            rule = await conn.fetchrow(
                "SELECT * FROM routing_rules WHERE id = $1 AND tenant_id = $2",
                rule_id, tenant_id,
            )
            if not rule:
                raise RuleNotFoundError(rule_id)

            integration = await conn.fetchrow(
                "SELECT * FROM cloud_integrations WHERE tenant_id = $1 AND status = 'active'",
                tenant_id,
            )

        provider = await self.providers.get_provider(dict(integration))

        # Deploy to provider (AWS IoT Rule, Azure route, etc.)
        try:
            from app.providers.base import CreateRoutingRuleRequest as ProviderReq
            result = await provider.create_routing_rule(
                ProviderReq(
                    rule_name=f"{tenant_id}-{rule['name']}",
                    source_topic=rule["source_topic"],
                    destination=json.loads(rule["destination"]),
                    filter_sql=rule["filter_sql"],
                )
            )
        except NotImplementedError:
            # Provider doesn't support programmatic routing (e.g., Mosquitto)
            result = type("Obj", (), {"provider_rule_id": "manual-config-required"})()

        # Update status
        async with self.db.acquire() as conn:
            updated = await conn.fetchrow(
                """UPDATE routing_rules
                   SET status = 'active', provider_rule_id = $1
                   WHERE id = $2 RETURNING *""",
                result.provider_rule_id, rule_id,
            )

        return dict(updated)

    def _scope_topic(self, tenant_id: str, topic_pattern: str) -> str:
        """Ensure topic pattern is scoped to tenant namespace."""
        if not topic_pattern.startswith(f"tenant/{tenant_id}"):
            return f"tenant/{tenant_id}/{topic_pattern.lstrip('/')}"
        return topic_pattern

    async def list_rules(self, tenant_id: str) -> list[dict]:
        async with self.db.acquire() as conn:
            rows = await conn.fetch(
                "SELECT * FROM routing_rules WHERE tenant_id = $1 ORDER BY created_at DESC",
                tenant_id,
            )
            return [dict(r) for r in rows]

    async def delete_rule(self, tenant_id: str, rule_id: str):
        async with self.db.acquire() as conn:
            rule = await conn.fetchrow(
                "SELECT * FROM routing_rules WHERE id = $1 AND tenant_id = $2", rule_id, tenant_id
            )
            if not rule:
                raise RuleNotFoundError(rule_id)

        # Delete at provider
        if rule["provider_rule_id"] and rule["provider_rule_id"] != "manual-config-required":
            integration = await self._get_integration(tenant_id)
            provider = await self.providers.get_provider(integration)
            await provider.delete_routing_rule(rule["provider_rule_id"])

        # Delete locally
        async with self.db.acquire() as conn:
            await conn.execute("DELETE FROM routing_rules WHERE id = $1", rule_id)
```

---

## 12.4 Provider-Specific Routing

| Provider | Routing Engine | Capabilities |
|----------|---------------|-------------|
| **AWS IoT Core** | IoT Rules Engine | SQL filter, 15+ action types (Lambda, Kinesis, S3, DynamoDB, etc.) |
| **Azure IoT Hub** | Message Routing | Routes, enrichments, endpoints (Event Hub, Blob, Service Bus) |
| **Mosquitto** | Bridge + plugins | Manual bridge config to external systems |

> ☕ *Java note*: If you're using Spring Cloud Stream or Apache Camel for routing, the concept is identical — define source, filter, and destination. The difference is that our platform configures routing *at the IoT provider level*, not in our own application.

---

## 12.5 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Control plane routing** | We configure rules, the provider executes them |
| **Topic scoping** | All routing rules are automatically scoped to tenant namespace |
| **Draft → Deploy** | Rules are created as drafts, then deployed to provider |
| **Provider differences** | AWS has rich SQL filtering; Mosquitto requires manual bridges |

### What's Next

**Chapter 13** begins **Part IV: Provider Integration & Operations** — deep-diving into multi-provider integration patterns and operational concerns.
