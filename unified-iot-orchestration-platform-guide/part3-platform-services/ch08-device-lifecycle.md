# Chapter 8: Device Lifecycle Service

> *"Every IoT device has a story — born, activated, sometimes suspended, and eventually retired. Our job is to manage that story reliably."*

---

## 8.1 Introduction

The Device Lifecycle Service manages the full journey of an IoT device — from creation through provisioning, operation, suspension, and eventual deletion. It orchestrates interactions between the database, the Provider Abstraction Layer, and the certificate service to ensure that every device exists consistently across both our platform and the cloud provider.

---

## 8.2 Device State Machine

```
                 ┌──────────┐
       create    │          │
    ────────────▶│ pending  │
                 │          │
                 └────┬─────┘
                      │
                provision (cert + policy created)
                      │
                      ▼
                 ┌──────────┐
    ┌───────────▶│          │◀──────────┐
    │            │  active  │           │
    │            │          │     reactivate
    │            └──┬───┬───┘           │
    │               │   │               │
    │        suspend│   │delete    ┌────┴─────┐
    │               │   │         │          │
    │               ▼   │         │suspended │
    │          ┌────────┐│        │          │
    │          │        ││        └────┬─────┘
    │          │suspend-││             │
    │          │ed      ││      delete │
    │          └────────┘│             │
    │                    │             │
    │                    ▼             ▼
    │              ┌──────────┐
    │              │          │
    │              │ deleted  │  (Soft delete — cleanup scheduled)
    │              │          │
    │              └──────────┘
```

---

## 8.3 Device Service Implementation

🐍 **Python**:

```python
# app/services/device_service.py
import json
from asyncpg import Pool
from app.providers.factory import ProviderFactory
from app.providers.base import CreateDeviceRequest, CreateCertRequest, CreatePolicyRequest
from app.services.quota_service import QuotaService
from app.events.publisher import EventPublisher


DEVICE_STATE_MACHINE = {
    "pending":   {"active", "deleted"},
    "active":    {"suspended", "deleted"},
    "suspended": {"active", "deleted"},
    "deleted":   set(),
}


class DeviceService:
    def __init__(
        self,
        db: Pool,
        provider_factory: ProviderFactory,
        quota_service: QuotaService,
        events: EventPublisher,
    ):
        self.db = db
        self.providers = provider_factory
        self.quotas = quota_service
        self.events = events

    async def create_device(self, tenant_id: str, name: str, metadata: dict = None) -> dict:
        """Create a new device (starts in 'pending' state)."""
        # 1. Check quota
        if not await self.quotas.check_quota(tenant_id, "device"):
            raise QuotaExceededException("Device limit reached")

        # 2. Insert into database
        async with self.db.acquire() as conn:
            row = await conn.fetchrow(
                """INSERT INTO devices (tenant_id, name, metadata, status)
                   VALUES ($1, $2, $3::jsonb, 'pending')
                   RETURNING *""",
                tenant_id, name, json.dumps(metadata or {}),
            )
            device = dict(row)

        await self.events.publish_audit_event(
            tenant_id=tenant_id,
            action="device.created",
            actor="api",
            resource_type="device",
            resource_id=str(device["id"]),
            outcome="success",
        )

        return device

    async def provision_device(self, tenant_id: str, device_id: str) -> dict:
        """Full provisioning: register with provider, create cert, attach policy."""

        async with self.db.acquire() as conn:
            device = await conn.fetchrow(
                "SELECT * FROM devices WHERE id = $1 AND tenant_id = $2", device_id, tenant_id
            )
            if not device:
                raise DeviceNotFoundError(device_id)
            if device["status"] != "pending":
                raise InvalidStateError(f"Device is '{device['status']}', must be 'pending'")

            # Get integration config
            integration = await conn.fetchrow(
                "SELECT * FROM cloud_integrations WHERE tenant_id = $1 AND status = 'active'",
                tenant_id,
            )
            if not integration:
                raise NoActiveIntegrationError(tenant_id)

        # 2. Create device at provider
        provider = await self.providers.get_provider(dict(integration))
        provider_result = await provider.create_device(
            CreateDeviceRequest(
                device_name=device["name"],
                tenant_id=tenant_id,
                metadata=device["metadata"] or {},
            )
        )

        # 3. Create certificate
        cert_result = await provider.create_certificate(
            CreateCertRequest(device_name=device["name"], tenant_id=tenant_id)
        )

        # 4. Attach cert to device at provider
        await provider.attach_certificate_to_device(
            cert_result.provider_cert_id, provider_result.provider_device_id
        )

        # 5. Create and attach policy
        topic_prefix = f"tenant/{tenant_id}/device/{device_id}"
        policy_result = await provider.create_policy(
            CreatePolicyRequest(
                policy_name=f"{tenant_id}-{device['name']}",
                tenant_id=tenant_id,
                device_id=str(device_id),
                allowed_topics=[
                    f"{topic_prefix}/telemetry",
                    f"{topic_prefix}/state",
                    f"{topic_prefix}/commands",
                ],
            )
        )

        await provider.attach_policy_to_certificate(
            policy_result.provider_policy_id, cert_result.provider_cert_id
        )

        # 6. Persist everything in a single transaction
        async with self.db.acquire() as conn:
            async with conn.transaction():
                # Save certificate
                cert_row = await conn.fetchrow(
                    """INSERT INTO certificates
                       (device_id, tenant_id, provider_cert_id, cert_pem, status, issued_at)
                       VALUES ($1, $2, $3, $4, 'active', NOW())
                       RETURNING id""",
                    device_id, tenant_id, cert_result.provider_cert_id, cert_result.cert_pem,
                )

                # Save policy
                await conn.execute(
                    """INSERT INTO policies
                       (device_id, tenant_id, policy_name, policy_document, provider_policy_id, attached_at)
                       VALUES ($1, $2, $3, $4, $5, NOW())""",
                    device_id, tenant_id, f"{tenant_id}-{device['name']}",
                    json.dumps(policy_result.policy_document), policy_result.provider_policy_id,
                )

                # Update device to active
                updated = await conn.fetchrow(
                    """UPDATE devices
                       SET status = 'active',
                           provider_device_id = $1,
                           active_cert_id = $2
                       WHERE id = $3
                       RETURNING *""",
                    provider_result.provider_device_id, cert_row["id"], device_id,
                )

        # 7. Emit events
        await self.events.publish_audit_event(
            tenant_id=tenant_id,
            action="device.provisioned",
            actor="api",
            resource_type="device",
            resource_id=str(device_id),
            outcome="success",
            details={
                "provider": provider.provider_name,
                "provider_device_id": provider_result.provider_device_id,
            },
        )

        return {
            "device": dict(updated),
            "certificate": {
                "cert_pem": cert_result.cert_pem,
                "private_key_pem": cert_result.private_key_pem,
                "ca_pem": cert_result.ca_pem,
            },
            "endpoint": provider_result.endpoint,
        }

    async def suspend_device(self, tenant_id: str, device_id: str) -> dict:
        """Suspend a device — detach from provider without deleting."""
        return await self._transition(tenant_id, device_id, "suspended", self._do_suspend)

    async def reactivate_device(self, tenant_id: str, device_id: str) -> dict:
        """Reactivate a suspended device."""
        return await self._transition(tenant_id, device_id, "active", self._do_reactivate)

    async def delete_device(self, tenant_id: str, device_id: str) -> dict:
        """Soft-delete a device — schedule provider cleanup."""
        return await self._transition(tenant_id, device_id, "deleted", self._do_delete)

    async def _transition(self, tenant_id, device_id, target, side_effect_fn):
        async with self.db.acquire() as conn:
            device = await conn.fetchrow(
                "SELECT * FROM devices WHERE id = $1 AND tenant_id = $2 FOR UPDATE",
                device_id, tenant_id,
            )
            if not device:
                raise DeviceNotFoundError(device_id)

            current = device["status"]
            if target not in DEVICE_STATE_MACHINE.get(current, set()):
                raise InvalidStateError(f"Cannot transition from '{current}' to '{target}'")

            # Execute side effects
            if side_effect_fn:
                await side_effect_fn(tenant_id, device)

            updated = await conn.fetchrow(
                "UPDATE devices SET status = $1::device_status WHERE id = $2 RETURNING *",
                target, device_id,
            )

        await self.events.publish_audit_event(
            tenant_id=tenant_id,
            action=f"device.{target}",
            actor="api",
            resource_type="device",
            resource_id=str(device_id),
            outcome="success",
        )
        return dict(updated)

    async def _do_suspend(self, tenant_id, device):
        """Detach certs at provider to prevent device from connecting."""
        if device["provider_device_id"]:
            integration = await self._get_integration(tenant_id)
            provider = await self.providers.get_provider(integration)
            await provider.suspend_device(device["provider_device_id"])

    async def _do_reactivate(self, tenant_id, device):
        """Re-attach certs at provider."""
        if device["provider_device_id"]:
            integration = await self._get_integration(tenant_id)
            provider = await self.providers.get_provider(integration)
            await provider.reactivate_device(device["provider_device_id"])

    async def _do_delete(self, tenant_id, device):
        """Revoke certs and delete device at provider."""
        if device["provider_device_id"]:
            integration = await self._get_integration(tenant_id)
            provider = await self.providers.get_provider(integration)
            # Revoke all active certs
            async with self.db.acquire() as conn:
                certs = await conn.fetch(
                    "SELECT * FROM certificates WHERE device_id = $1 AND status = 'active'",
                    device["id"],
                )
                for cert in certs:
                    await provider.revoke_certificate(cert["provider_cert_id"])
                    await conn.execute(
                        "UPDATE certificates SET status = 'revoked', revoked_at = NOW() WHERE id = $1",
                        cert["id"],
                    )
            # Delete device at provider
            await provider.delete_device(device["provider_device_id"])
```

---

## 8.4 Handling Partial Failures

The provisioning workflow involves multiple external calls. If step 3 fails, steps 1 and 2 need to be cleaned up:

```python
# app/services/device_service.py (continued)

async def provision_device_safe(self, tenant_id: str, device_id: str) -> dict:
    """Provisioning with compensation on failure."""
    provider = None
    provider_device_id = None
    provider_cert_id = None

    try:
        # ... Steps 1-5 as above, tracking created resources ...
        return result

    except Exception as e:
        # ── Compensate: undo what was created ──
        if provider:
            if provider_cert_id:
                try:
                    await provider.revoke_certificate(provider_cert_id)
                except Exception:
                    pass  # Log but don't mask original error

            if provider_device_id:
                try:
                    await provider.delete_device(provider_device_id)
                except Exception:
                    pass

        # Mark device as failed
        async with self.db.acquire() as conn:
            await conn.execute(
                """UPDATE devices
                   SET status = 'pending',
                       metadata = metadata || '{"last_error": $1}'::jsonb
                   WHERE id = $2""",
                str(e), device_id,
            )

        await self.events.publish_audit_event(
            tenant_id=tenant_id,
            action="device.provision_failed",
            actor="api",
            resource_type="device",
            resource_id=str(device_id),
            outcome="failure",
            details={"error": str(e)},
        )

        raise ProvisioningError(f"Failed to provision device: {e}") from e
```

> ☕ *Java developers*: This compensation pattern is equivalent to the Saga pattern commonly implemented in Spring Cloud with Axon Framework or Eventuate. In C#, it maps to the compensating transaction pattern used in Azure Durable Functions or MassTransit sagas.

---

## 8.5 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Device states** | pending → active → suspended → deleted (with constraints) |
| **Provisioning** | Multi-step: create + cert + policy + attach, in transaction |
| **Partial failure** | Compensating actions clean up provider resources on error |
| **Suspension** | Detaches certs at provider — device exists but can't connect |
| **Soft delete** | Revoke certs, delete at provider, mark as deleted in DB |

### What's Next

**Chapter 9** covers the **Certificate Lifecycle Service** — generation, rotation, revocation, and expiry tracking.
