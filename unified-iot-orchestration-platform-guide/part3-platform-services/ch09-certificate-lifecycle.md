# Chapter 9: Certificate Lifecycle Service

> *"Certificates are the passports of the IoT world — without proper lifecycle management, you end up with expired identity documents and locked-out devices."*

---

## 9.1 Introduction

Every IoT device needs a cryptographic identity to communicate securely with its provider. The Certificate Lifecycle Service manages the full certificate journey: creation, attachment to device identities, rotation before expiry, and revocation when devices are decommissioned.

---

## 9.2 Certificate Generation Strategies

| Strategy | Used When | Source |
|----------|----------|--------|
| **Provider-generated** | AWS IoT Core | AWS creates and manages the cert keypair |
| **Vault PKI** | Mosquitto, self-hosted | Vault acts as an internal CA |
| **External CA** | Enterprise requirements | CFSSL / step-ca / commercial CA |
| **Customer-provided** | BYOC (Bring Your Own Cert) | Customer supplies pre-generated certs |

🐍 **Python — Certificate Service**:

```python
# app/services/certificate_service.py
from asyncpg import Pool
from app.vault.client import VaultClient, CertificateResponse
from app.providers.factory import ProviderFactory
from app.events.publisher import EventPublisher


class CertificateService:
    def __init__(self, db: Pool, vault: VaultClient, providers: ProviderFactory, events: EventPublisher):
        self.db = db
        self.vault = vault
        self.providers = providers
        self.events = events

    async def issue_certificate(self, tenant_id: str, device_id: str, strategy: str = "auto") -> dict:
        """Issue a new certificate for a device using the appropriate strategy."""

        async with self.db.acquire() as conn:
            device = await conn.fetchrow(
                "SELECT * FROM devices WHERE id = $1 AND tenant_id = $2", device_id, tenant_id
            )
            integration = await conn.fetchrow(
                "SELECT * FROM cloud_integrations WHERE tenant_id = $1 AND status = 'active'", tenant_id
            )

        provider = await self.providers.get_provider(dict(integration))

        if strategy == "auto":
            strategy = "provider" if integration["provider"] in ("aws", "azure") else "vault"

        if strategy == "vault":
            result = await self._issue_via_vault(tenant_id, device["name"])
        elif strategy == "provider":
            result = await self._issue_via_provider(provider, tenant_id, device["name"])
        else:
            raise ValueError(f"Unknown cert strategy: {strategy}")

        # Persist certificate record
        async with self.db.acquire() as conn:
            cert_row = await conn.fetchrow(
                """INSERT INTO certificates
                   (device_id, tenant_id, provider_cert_id, cert_pem,
                    fingerprint, status, issued_at, expires_at)
                   VALUES ($1, $2, $3, $4, $5, 'active', NOW(), $6)
                   RETURNING *""",
                device_id, tenant_id, result.provider_cert_id, result.cert_pem,
                self._compute_fingerprint(result.cert_pem), result.expires_at,
            )

        await self.events.publish_audit_event(
            tenant_id=tenant_id, action="certificate.issued", actor="api",
            resource_type="certificate", resource_id=str(cert_row["id"]),
            outcome="success",
            details={"strategy": strategy, "device_id": str(device_id)},
        )

        return {
            "certificate_id": str(cert_row["id"]),
            "cert_pem": result.cert_pem,
            "private_key_pem": result.private_key_pem,
            "ca_pem": result.ca_pem,
        }

    async def _issue_via_vault(self, tenant_id: str, device_name: str) -> CertificateResponse:
        common_name = f"{device_name}.{tenant_id}.iot.local"
        return await self.vault.issue_certificate(common_name, ttl="8760h")

    async def _issue_via_provider(self, provider, tenant_id, device_name):
        from app.providers.base import CreateCertRequest
        return await provider.create_certificate(
            CreateCertRequest(device_name=device_name, tenant_id=tenant_id)
        )

    async def rotate_certificate(self, tenant_id: str, device_id: str, cert_id: str) -> dict:
        """Rotate: issue new cert, attach to device, revoke old cert."""
        async with self.db.acquire() as conn:
            old_cert = await conn.fetchrow(
                "SELECT * FROM certificates WHERE id = $1 AND tenant_id = $2 AND status = 'active'",
                cert_id, tenant_id,
            )
            if not old_cert:
                raise CertificateNotFoundError(cert_id)

        # 1. Issue new certificate
        new_cert = await self.issue_certificate(tenant_id, device_id)

        # 2. Attach new cert at provider
        integration = await self._get_integration(tenant_id)
        provider = await self.providers.get_provider(integration)
        device = await self._get_device(device_id)

        if device["provider_device_id"]:
            await provider.attach_certificate_to_device(
                new_cert["provider_cert_id"] if "provider_cert_id" in new_cert else "",
                device["provider_device_id"],
            )

        # 3. Revoke old certificate
        await self.revoke_certificate(tenant_id, cert_id)

        # 4. Update device's active cert reference
        async with self.db.acquire() as conn:
            await conn.execute(
                "UPDATE devices SET active_cert_id = $1 WHERE id = $2",
                new_cert["certificate_id"], device_id,
            )

        return new_cert

    async def revoke_certificate(self, tenant_id: str, cert_id: str) -> None:
        """Revoke a certificate at both provider and in our database."""
        async with self.db.acquire() as conn:
            cert = await conn.fetchrow(
                "SELECT * FROM certificates WHERE id = $1 AND tenant_id = $2", cert_id, tenant_id
            )
            if not cert:
                raise CertificateNotFoundError(cert_id)

        # Revoke at provider
        if cert["provider_cert_id"]:
            integration = await self._get_integration(tenant_id)
            provider = await self.providers.get_provider(integration)
            await provider.revoke_certificate(cert["provider_cert_id"])

        # Update database
        async with self.db.acquire() as conn:
            await conn.execute(
                "UPDATE certificates SET status = 'revoked', revoked_at = NOW() WHERE id = $1",
                cert_id,
            )

        await self.events.publish_audit_event(
            tenant_id=tenant_id, action="certificate.revoked", actor="api",
            resource_type="certificate", resource_id=str(cert_id), outcome="success",
        )

    @staticmethod
    def _compute_fingerprint(cert_pem: str) -> str:
        import hashlib
        from cryptography.x509 import load_pem_x509_certificate
        cert = load_pem_x509_certificate(cert_pem.encode())
        return hashlib.sha256(cert.public_bytes(serialization.Encoding.DER)).hexdigest()
```

---

## 9.3 Automatic Expiry Detection

A background job scans for certificates nearing expiry:

```python
# app/workers/cert_expiry_checker.py
import asyncio
from datetime import datetime, timedelta, timezone


class CertExpiryChecker:
    """Background worker that detects and alerts on expiring certificates."""

    def __init__(self, db, events, check_interval_hours=6, warning_days=30):
        self.db = db
        self.events = events
        self.interval = check_interval_hours * 3600
        self.warning_threshold = timedelta(days=warning_days)

    async def run_forever(self):
        while True:
            await self.check_expiring_certs()
            await asyncio.sleep(self.interval)

    async def check_expiring_certs(self):
        threshold = datetime.now(timezone.utc) + self.warning_threshold

        async with self.db.acquire() as conn:
            expiring = await conn.fetch(
                """SELECT c.*, d.name as device_name, d.tenant_id
                   FROM certificates c
                   JOIN devices d ON c.device_id = d.id
                   WHERE c.status = 'active'
                     AND c.expires_at IS NOT NULL
                     AND c.expires_at < $1
                   ORDER BY c.expires_at ASC""",
                threshold,
            )

        for cert in expiring:
            days_left = (cert["expires_at"] - datetime.now(timezone.utc)).days

            await self.events.publish_audit_event(
                tenant_id=str(cert["tenant_id"]),
                action="certificate.expiry_warning",
                actor="system",
                resource_type="certificate",
                resource_id=str(cert["id"]),
                outcome="warning",
                details={
                    "device_name": cert["device_name"],
                    "expires_at": cert["expires_at"].isoformat(),
                    "days_remaining": days_left,
                },
            )
```

---

## 9.4 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Multi-strategy issuance** | Auto-select between Vault PKI and provider-native certs |
| **Rotation** | Issue new → attach → revoke old, atomically |
| **Revocation** | Revoke at provider AND update database |
| **Expiry monitoring** | Background worker scans and alerts before expiry |

### What's Next

**Chapter 10** builds the **Policy & Topic Namespace Engine** — generating tenant-isolated topic structures and translating policies to provider-specific formats.

---

# Chapter 10: Policy & Topic Namespace Engine

> *"The namespace is the foundation of tenant isolation in MQTT. Get it wrong, and Tenant A reads Tenant B's data."*

---

## 10.1 Introduction

The Policy & Topic Namespace Engine serves two critical functions:
1. **Define standardized topic structures** that enforce tenant and device isolation
2. **Generate provider-specific access control policies** that restrict each device to its own namespace

---

## 10.2 Topic Namespace Structure

```
tenant/{tenant_id}/device/{device_id}/telemetry     # Device → Cloud
tenant/{tenant_id}/device/{device_id}/state          # Device state reports
tenant/{tenant_id}/device/{device_id}/commands        # Cloud → Device
tenant/{tenant_id}/device/{device_id}/config          # Configuration updates
tenant/{tenant_id}/device/{device_id}/ota             # Firmware updates
tenant/{tenant_id}/fleet/broadcast                    # Tenant-wide broadcasts
```

### Why This Structure?

| Level | Purpose |
|-------|---------|
| `tenant/{id}` | **Tenant isolation** — all topics scoped to tenant |
| `device/{id}` | **Device isolation** — each device has its own namespace |
| `telemetry` | **Direction** — distinguishes inbound from outbound |
| `commands` | **Cloud-to-device** — control channel |

---

## 10.3 Policy Generation

🐍 **Python — Namespace & Policy Engine**:

```python
# app/services/namespace_service.py
from dataclasses import dataclass


@dataclass
class TopicNamespace:
    tenant_id: str
    device_id: str
    device_name: str

    @property
    def telemetry_topic(self) -> str:
        return f"tenant/{self.tenant_id}/device/{self.device_id}/telemetry"

    @property
    def state_topic(self) -> str:
        return f"tenant/{self.tenant_id}/device/{self.device_id}/state"

    @property
    def commands_topic(self) -> str:
        return f"tenant/{self.tenant_id}/device/{self.device_id}/commands"

    @property
    def config_topic(self) -> str:
        return f"tenant/{self.tenant_id}/device/{self.device_id}/config"

    @property
    def all_publish_topics(self) -> list[str]:
        return [self.telemetry_topic, self.state_topic]

    @property
    def all_subscribe_topics(self) -> list[str]:
        return [self.commands_topic, self.config_topic]


class PolicyEngine:
    """Generate provider-specific access policies from a namespace."""

    def generate_policy(self, namespace: TopicNamespace, provider: str) -> dict:
        generators = {
            "aws": self._generate_aws_policy,
            "azure": self._generate_azure_policy,
            "mosquitto": self._generate_mosquitto_acl,
        }

        generator = generators.get(provider)
        if not generator:
            raise ValueError(f"Unsupported provider for policy generation: {provider}")

        return generator(namespace)

    def _generate_aws_policy(self, ns: TopicNamespace) -> dict:
        return {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": "iot:Connect",
                    "Resource": f"arn:aws:iot:*:*:client/{ns.device_name}",
                },
                {
                    "Effect": "Allow",
                    "Action": "iot:Publish",
                    "Resource": [f"arn:aws:iot:*:*:topic/{t}" for t in ns.all_publish_topics],
                },
                {
                    "Effect": "Allow",
                    "Action": ["iot:Subscribe"],
                    "Resource": [f"arn:aws:iot:*:*:topicfilter/{t}" for t in ns.all_subscribe_topics],
                },
                {
                    "Effect": "Allow",
                    "Action": "iot:Receive",
                    "Resource": [f"arn:aws:iot:*:*:topic/{t}" for t in ns.all_subscribe_topics],
                },
            ],
        }

    def _generate_azure_policy(self, ns: TopicNamespace) -> dict:
        """Azure IoT Hub uses device-level permissions — policy is simpler."""
        return {
            "device_id": ns.device_name,
            "permissions": {
                "device_to_cloud": True,
                "cloud_to_device": True,
                "twin_read": True,
                "twin_write": False,
                "custom_topics": ns.all_publish_topics + ns.all_subscribe_topics,
            },
        }

    def _generate_mosquitto_acl(self, ns: TopicNamespace) -> dict:
        """Mosquitto uses a flat ACL file format."""
        lines = [f"user {ns.device_name}"]
        for topic in ns.all_publish_topics:
            lines.append(f"topic write {topic}")
        for topic in ns.all_subscribe_topics:
            lines.append(f"topic read {topic}")

        return {
            "format": "mosquitto_acl",
            "content": "\n".join(lines),
        }
```

---

## 10.4 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Namespace hierarchy** | `tenant/{id}/device/{id}/{purpose}` enforces isolation |
| **Policy generation** | Single engine produces AWS, Azure, and Mosquitto policies |
| **Direction separation** | Publish vs. subscribe topics prevent unauthorized reads |
| **Cross-provider** | Same namespace logic, different policy formats |

### What's Next

**Chapter 11** covers **Provisioning Workflows** — multi-step onboarding strategies from factory provisioning to claim-based flows.
