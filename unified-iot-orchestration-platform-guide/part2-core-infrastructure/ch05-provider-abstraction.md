# Chapter 5: The IoT Provider Abstraction Layer

> *"All problems in computer science can be solved by another level of indirection."* — David Wheeler

---

## 5.1 Introduction

The **Provider Abstraction Layer (PAL)** is the architectural linchpin of the entire platform. It sits between your business logic and the cloud IoT providers, ensuring that:

1. **No business logic service ever calls a provider API directly.**
2. **Adding a new provider requires implementing one interface — not rewriting services.**
3. **Provider-specific behavior is encapsulated and testable in isolation.**

Without the PAL, every service would have `if provider == "aws" ... else if provider == "azure"` branches scattered throughout the codebase. With the PAL, services speak a unified language.

> 💡 **For Beginners**: Think of the PAL as a universal power adapter. Your laptop (the platform) has one plug, and the adapter (PAL) translates it to US, EU, or UK sockets (AWS, Azure, Mosquitto). The laptop doesn't need to know what country it's in.

---

## 5.2 The Provider Interface

This is the contract every provider must fulfill. We revisit the interface from Chapter 1 with full production detail:

🔵 **Go — Complete Provider Interface**:

```go
// internal/provider/provider.go
package provider

import (
    "context"
    "time"
)

// Provider defines the contract for all IoT provider adapters.
type Provider interface {
    // ── Device Operations ──────────────────────────────────

    // CreateDevice registers a new device with the IoT provider.
    CreateDevice(ctx context.Context, req CreateDeviceRequest) (*DeviceResult, error)

    // DeleteDevice removes a device from the IoT provider.
    DeleteDevice(ctx context.Context, providerDeviceID string) error

    // SuspendDevice disables a device without deleting it.
    SuspendDevice(ctx context.Context, providerDeviceID string) error

    // ReactivateDevice re-enables a suspended device.
    ReactivateDevice(ctx context.Context, providerDeviceID string) error

    // ── Certificate Operations ─────────────────────────────

    // CreateCertificate generates and registers a device certificate.
    CreateCertificate(ctx context.Context, req CreateCertRequest) (*CertificateResult, error)

    // RevokeCertificate marks a certificate as invalid at the provider.
    RevokeCertificate(ctx context.Context, providerCertID string) error

    // RotateCertificate creates a new cert and revokes the old one.
    RotateCertificate(ctx context.Context, oldProviderCertID string) (*CertificateResult, error)

    // AttachCertificateToDevice binds a certificate to a device.
    AttachCertificateToDevice(ctx context.Context, providerCertID, providerDeviceID string) error

    // ── Policy Operations ──────────────────────────────────

    // CreatePolicy creates an access control policy at the provider.
    CreatePolicy(ctx context.Context, req CreatePolicyRequest) (*PolicyResult, error)

    // AttachPolicy binds a policy to a certificate.
    AttachPolicyToCertificate(ctx context.Context, providerPolicyID, providerCertID string) error

    // DeletePolicy removes a policy from the provider.
    DeletePolicy(ctx context.Context, providerPolicyID string) error

    // ── Routing Operations ─────────────────────────────────

    // CreateRoutingRule configures a telemetry routing rule.
    CreateRoutingRule(ctx context.Context, req CreateRoutingRuleRequest) (*RoutingRuleResult, error)

    // DeleteRoutingRule removes a routing rule.
    DeleteRoutingRule(ctx context.Context, providerRuleID string) error

    // ── Infrastructure ─────────────────────────────────────

    // GetEndpoint returns the provider's IoT endpoint URL.
    GetEndpoint(ctx context.Context) (string, error)

    // ValidateCredentials checks if the stored credentials are valid.
    ValidateCredentials(ctx context.Context) error

    // ProviderName returns the provider's identifier ("aws", "azure", "mosquitto").
    ProviderName() string
}
```

🐍 **Python — Complete Provider Interface**:

```python
# app/providers/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any


# ── Request/Response DTOs ──────────────────────────────

@dataclass
class CreateDeviceRequest:
    device_name: str
    tenant_id: str
    metadata: dict = field(default_factory=dict)


@dataclass
class DeviceResult:
    provider_device_id: str        # e.g., AWS Thing ARN
    endpoint: str                  # Provider's MQTT endpoint
    metadata: dict = field(default_factory=dict)


@dataclass
class CreateCertRequest:
    device_name: str
    tenant_id: str
    validity_days: int = 365


@dataclass
class CertificateResult:
    provider_cert_id: str          # e.g., AWS certificate ID
    cert_pem: str                  # PEM-encoded certificate
    private_key_pem: str           # PEM-encoded private key (returned ONCE)
    ca_pem: str | None = None      # CA certificate chain
    expires_at: str | None = None


@dataclass
class CreatePolicyRequest:
    policy_name: str
    tenant_id: str
    device_id: str
    allowed_topics: list[str]      # Topics the device can publish/subscribe to


@dataclass
class PolicyResult:
    provider_policy_id: str
    policy_document: dict


@dataclass
class CreateRoutingRuleRequest:
    rule_name: str
    source_topic: str
    destination: dict              # {"type": "kinesis", "arn": "..."}
    filter_sql: str | None = None


@dataclass
class RoutingRuleResult:
    provider_rule_id: str


# ── Abstract Provider ──────────────────────────────────

class IoTProvider(ABC):
    """Base class for all IoT provider adapters."""

    @property
    @abstractmethod
    def provider_name(self) -> str:
        """Return provider identifier: 'aws', 'azure', 'mosquitto'."""
        ...

    # ── Device Operations ──
    @abstractmethod
    async def create_device(self, req: CreateDeviceRequest) -> DeviceResult: ...

    @abstractmethod
    async def delete_device(self, provider_device_id: str) -> None: ...

    @abstractmethod
    async def suspend_device(self, provider_device_id: str) -> None: ...

    @abstractmethod
    async def reactivate_device(self, provider_device_id: str) -> None: ...

    # ── Certificate Operations ──
    @abstractmethod
    async def create_certificate(self, req: CreateCertRequest) -> CertificateResult: ...

    @abstractmethod
    async def revoke_certificate(self, provider_cert_id: str) -> None: ...

    @abstractmethod
    async def rotate_certificate(self, old_cert_id: str) -> CertificateResult: ...

    @abstractmethod
    async def attach_certificate_to_device(
        self, provider_cert_id: str, provider_device_id: str
    ) -> None: ...

    # ── Policy Operations ──
    @abstractmethod
    async def create_policy(self, req: CreatePolicyRequest) -> PolicyResult: ...

    @abstractmethod
    async def attach_policy_to_certificate(
        self, provider_policy_id: str, provider_cert_id: str
    ) -> None: ...

    @abstractmethod
    async def delete_policy(self, provider_policy_id: str) -> None: ...

    # ── Routing Operations ──
    @abstractmethod
    async def create_routing_rule(
        self, req: CreateRoutingRuleRequest
    ) -> RoutingRuleResult: ...

    @abstractmethod
    async def delete_routing_rule(self, provider_rule_id: str) -> None: ...

    # ── Infrastructure ──
    @abstractmethod
    async def get_endpoint(self) -> str: ...

    @abstractmethod
    async def validate_credentials(self) -> None: ...
```

---

## 5.3 The AWS IoT Core Adapter

🐍 **Python — AWS Adapter (key methods)**:

```python
# app/providers/aws_provider.py
import boto3
import json
from botocore.config import Config

from .base import (
    IoTProvider, CreateDeviceRequest, DeviceResult,
    CreateCertRequest, CertificateResult,
    CreatePolicyRequest, PolicyResult,
)


class AwsIoTProvider(IoTProvider):
    """AWS IoT Core adapter using boto3."""

    def __init__(self, role_arn: str, external_id: str, region: str):
        self._role_arn = role_arn
        self._external_id = external_id
        self._region = region
        self._client = self._assume_role_and_create_client()

    @property
    def provider_name(self) -> str:
        return "aws"

    def _assume_role_and_create_client(self):
        """Assume the tenant's cross-account IAM role."""
        sts = boto3.client("sts")
        credentials = sts.assume_role(
            RoleArn=self._role_arn,
            ExternalId=self._external_id,
            RoleSessionName="iot-platform",
            DurationSeconds=3600,
        )["Credentials"]

        return boto3.client(
            "iot",
            region_name=self._region,
            aws_access_key_id=credentials["AccessKeyId"],
            aws_secret_access_key=credentials["SecretAccessKey"],
            aws_session_token=credentials["SessionToken"],
            config=Config(retries={"max_attempts": 3, "mode": "adaptive"}),
        )

    async def create_device(self, req: CreateDeviceRequest) -> DeviceResult:
        # AWS calls this a "Thing"
        response = self._client.create_thing(
            thingName=req.device_name,
            attributePayload={
                "attributes": {
                    "tenant_id": req.tenant_id,
                    "managed_by": "iot-platform",
                    **{k: str(v) for k, v in req.metadata.items()},
                }
            },
        )

        endpoint = await self.get_endpoint()

        return DeviceResult(
            provider_device_id=response["thingArn"],
            endpoint=endpoint,
            metadata={"thing_name": response["thingName"]},
        )

    async def create_certificate(self, req: CreateCertRequest) -> CertificateResult:
        response = self._client.create_keys_and_certificate(setAsActive=True)

        return CertificateResult(
            provider_cert_id=response["certificateId"],
            cert_pem=response["certificatePem"],
            private_key_pem=response["keyPair"]["PrivateKey"],
            ca_pem=None,  # AWS provides the root CA separately
            expires_at=None,  # AWS-managed certs don't have a fixed expiry
        )

    async def create_policy(self, req: CreatePolicyRequest) -> PolicyResult:
        policy_document = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": ["iot:Publish", "iot:Receive"],
                    "Resource": [
                        f"arn:aws:iot:{self._region}:*:topic/{topic}"
                        for topic in req.allowed_topics
                    ],
                },
                {
                    "Effect": "Allow",
                    "Action": "iot:Subscribe",
                    "Resource": [
                        f"arn:aws:iot:{self._region}:*:topicfilter/{topic}"
                        for topic in req.allowed_topics
                    ],
                },
                {
                    "Effect": "Allow",
                    "Action": "iot:Connect",
                    "Resource": f"arn:aws:iot:{self._region}:*:client/{req.device_id}",
                },
            ],
        }

        response = self._client.create_policy(
            policyName=req.policy_name,
            policyDocument=json.dumps(policy_document),
        )

        return PolicyResult(
            provider_policy_id=response["policyName"],
            policy_document=policy_document,
        )

    async def attach_policy_to_certificate(
        self, provider_policy_id: str, provider_cert_id: str
    ) -> None:
        cert_arn = self._client.describe_certificate(
            certificateId=provider_cert_id
        )["certificateDescription"]["certificateArn"]

        self._client.attach_policy(
            policyName=provider_policy_id,
            target=cert_arn,
        )

    async def attach_certificate_to_device(
        self, provider_cert_id: str, provider_device_id: str
    ) -> None:
        # provider_device_id is the Thing ARN; extract the thing name
        thing_name = provider_device_id.split("/")[-1]
        cert_arn = self._client.describe_certificate(
            certificateId=provider_cert_id
        )["certificateDescription"]["certificateArn"]

        self._client.attach_thing_principal(
            thingName=thing_name,
            principal=cert_arn,
        )

    async def get_endpoint(self) -> str:
        response = self._client.describe_endpoint(endpointType="iot:Data-ATS")
        return response["endpointAddress"]

    async def validate_credentials(self) -> None:
        # If we can describe the endpoint, credentials are valid
        self._client.describe_endpoint(endpointType="iot:Data-ATS")

    async def delete_device(self, provider_device_id: str) -> None:
        thing_name = provider_device_id.split("/")[-1]
        self._client.delete_thing(thingName=thing_name)

    async def suspend_device(self, provider_device_id: str) -> None:
        # AWS doesn't have a native "suspend" — we detach all principals
        thing_name = provider_device_id.split("/")[-1]
        principals = self._client.list_thing_principals(thingName=thing_name)
        for principal in principals.get("principals", []):
            self._client.detach_thing_principal(
                thingName=thing_name, principal=principal
            )

    async def reactivate_device(self, provider_device_id: str) -> None:
        # Re-attach the stored certificate
        raise NotImplementedError("Requires cert reference from platform DB")

    async def revoke_certificate(self, provider_cert_id: str) -> None:
        self._client.update_certificate(
            certificateId=provider_cert_id,
            newStatus="REVOKED",
        )

    async def rotate_certificate(self, old_cert_id: str) -> CertificateResult:
        # Revoke old, create new
        await self.revoke_certificate(old_cert_id)
        return await self.create_certificate(
            CreateCertRequest(device_name="", tenant_id="")  # Populated by caller
        )

    async def create_routing_rule(self, req) -> None:
        raise NotImplementedError("Routing rules via AWS IoT Rules Engine — Chapter 12")

    async def delete_routing_rule(self, provider_rule_id: str) -> None:
        raise NotImplementedError("Routing rules via AWS IoT Rules Engine — Chapter 12")
```

---

## 5.4 The Mosquitto Adapter

The Mosquitto adapter demonstrates a self-hosted provider — no cloud APIs, but direct broker management:

🐍 **Python — Mosquitto Adapter (key methods)**:

```python
# app/providers/mosquitto_provider.py
import subprocess
import json
from pathlib import Path

from .base import (
    IoTProvider, CreateDeviceRequest, DeviceResult,
    CreateCertRequest, CertificateResult,
    CreatePolicyRequest, PolicyResult,
)


class MosquittoProvider(IoTProvider):
    """Eclipse Mosquitto adapter — manages a self-hosted MQTT broker."""

    def __init__(
        self,
        broker_url: str,
        admin_username: str,
        admin_password: str,
        acl_file_path: str = "/mosquitto/config/acl",
        password_file_path: str = "/mosquitto/config/passwords",
        ca_cert_path: str = "/mosquitto/certs/ca.pem",
        ca_key_path: str = "/mosquitto/certs/ca.key",
    ):
        self._broker_url = broker_url
        self._admin_user = admin_username
        self._admin_pass = admin_password
        self._acl_path = Path(acl_file_path)
        self._password_path = Path(password_file_path)
        self._ca_cert = Path(ca_cert_path)
        self._ca_key = Path(ca_key_path)

    @property
    def provider_name(self) -> str:
        return "mosquitto"

    async def create_device(self, req: CreateDeviceRequest) -> DeviceResult:
        """For Mosquitto, a 'device' is just a client identity."""
        # Generate a unique client ID
        client_id = f"{req.tenant_id}_{req.device_name}"

        # Add password entry (in production, use dynamic security plugin)
        self._add_password_entry(client_id, self._generate_password())

        return DeviceResult(
            provider_device_id=client_id,
            endpoint=self._broker_url,
            metadata={"client_id": client_id},
        )

    async def create_certificate(self, req: CreateCertRequest) -> CertificateResult:
        """Generate a client certificate signed by our CA."""
        import subprocess

        client_id = f"{req.tenant_id}_{req.device_name}"

        # Generate private key
        key_path = f"/tmp/{client_id}.key"
        subprocess.run([
            "openssl", "ecparam", "-genkey", "-name", "prime256v1",
            "-out", key_path,
        ], check=True)

        # Generate CSR
        csr_path = f"/tmp/{client_id}.csr"
        subprocess.run([
            "openssl", "req", "-new", "-key", key_path,
            "-out", csr_path, "-subj", f"/CN={client_id}",
        ], check=True)

        # Sign with CA
        cert_path = f"/tmp/{client_id}.pem"
        subprocess.run([
            "openssl", "x509", "-req", "-in", csr_path,
            "-CA", str(self._ca_cert), "-CAkey", str(self._ca_key),
            "-CAcreateserial", "-out", cert_path,
            "-days", str(req.validity_days), "-sha256",
        ], check=True)

        # Read generated files
        cert_pem = Path(cert_path).read_text()
        private_key = Path(key_path).read_text()
        ca_pem = self._ca_cert.read_text()

        # Cleanup temp files
        for p in [key_path, csr_path, cert_path]:
            Path(p).unlink(missing_ok=True)

        return CertificateResult(
            provider_cert_id=client_id,
            cert_pem=cert_pem,
            private_key_pem=private_key,
            ca_pem=ca_pem,
        )

    async def create_policy(self, req: CreatePolicyRequest) -> PolicyResult:
        """For Mosquitto, policies are ACL file entries."""
        acl_entries = []
        for topic in req.allowed_topics:
            acl_entries.append(f"topic readwrite {topic}")

        # Append to ACL file
        client_id = f"{req.tenant_id}_{req.device_id}"
        acl_block = f"\nuser {client_id}\n" + "\n".join(acl_entries) + "\n"

        with open(self._acl_path, "a") as f:
            f.write(acl_block)

        # Signal Mosquitto to reload config
        subprocess.run(["mosquitto_ctrl", "dynsec", "reload"], check=False)

        return PolicyResult(
            provider_policy_id=f"acl:{client_id}",
            policy_document={"user": client_id, "acl": acl_entries},
        )

    async def get_endpoint(self) -> str:
        return self._broker_url

    async def validate_credentials(self) -> None:
        """Check we can reach the broker."""
        import paho.mqtt.client as mqtt

        client = mqtt.Client()
        client.username_pw_set(self._admin_user, self._admin_pass)
        client.connect(self._broker_url.replace("tcp://", "").split(":")[0], 1883, 5)
        client.disconnect()

    # ... other methods follow the same pattern
    async def delete_device(self, provider_device_id: str) -> None:
        self._remove_password_entry(provider_device_id)

    async def suspend_device(self, provider_device_id: str) -> None:
        self._remove_acl_entries(provider_device_id)

    async def reactivate_device(self, provider_device_id: str) -> None:
        raise NotImplementedError("Re-attach ACL from stored policy")

    async def revoke_certificate(self, provider_cert_id: str) -> None:
        # Add to CRL or remove client cert from trust store
        pass

    async def rotate_certificate(self, old_cert_id: str) -> CertificateResult:
        await self.revoke_certificate(old_cert_id)
        return await self.create_certificate(
            CreateCertRequest(device_name=old_cert_id, tenant_id="")
        )

    async def attach_certificate_to_device(self, cert_id, device_id) -> None:
        pass  # Certificate CN matches client ID — no explicit attachment needed

    async def attach_policy_to_certificate(self, policy_id, cert_id) -> None:
        pass  # ACL is user-based, not cert-based in Mosquitto

    async def delete_policy(self, provider_policy_id: str) -> None:
        client_id = provider_policy_id.replace("acl:", "")
        self._remove_acl_entries(client_id)

    async def create_routing_rule(self, req):
        raise NotImplementedError("Mosquitto uses bridge config for routing")

    async def delete_routing_rule(self, provider_rule_id):
        raise NotImplementedError("Mosquitto bridge management")

    # ── Helper methods ──
    def _add_password_entry(self, username: str, password: str):
        subprocess.run([
            "mosquitto_passwd", "-b", str(self._password_path),
            username, password,
        ], check=True)

    def _remove_password_entry(self, username: str):
        subprocess.run([
            "mosquitto_passwd", "-D", str(self._password_path), username,
        ], check=True)

    @staticmethod
    def _generate_password(length: int = 32) -> str:
        import secrets
        return secrets.token_urlsafe(length)

    def _remove_acl_entries(self, client_id: str):
        """Remove ACL block for a specific client."""
        lines = self._acl_path.read_text().splitlines()
        filtered = []
        skip = False
        for line in lines:
            if line.strip() == f"user {client_id}":
                skip = True
                continue
            if skip and line.startswith("user "):
                skip = False
            if not skip:
                filtered.append(line)
        self._acl_path.write_text("\n".join(filtered) + "\n")
```

> ☕ *Comparison note*: In Java (Spring), this adapter pattern is identical to implementing an interface and annotating with `@Component` / `@Qualifier("mosquitto")`. The DI container selects the correct bean at runtime. In C#, you'd register `services.AddScoped<IIoTProvider, MosquittoProvider>()` with a factory that reads tenant config.

---

## 5.5 Provider Factory

The factory creates the right provider based on tenant configuration:

🔵 **Go**:

```go
// internal/provider/factory.go
package provider

import (
    "context"
    "fmt"
    "sync"
)

type Factory struct {
    vaultClient VaultClient
    cache       sync.Map  // tenant_id -> Provider
}

func NewFactory(vaultClient VaultClient) *Factory {
    return &Factory{vaultClient: vaultClient}
}

func (f *Factory) GetProvider(ctx context.Context, integration CloudIntegration) (Provider, error) {
    // Check cache first
    if cached, ok := f.cache.Load(integration.ID); ok {
        return cached.(Provider), nil
    }

    // Fetch credentials from Vault
    creds, err := f.vaultClient.ReadSecret(ctx, integration.VaultSecretPath)
    if err != nil {
        return nil, fmt.Errorf("fetch credentials: %w", err)
    }

    // Create the appropriate provider
    var p Provider
    switch integration.Provider {
    case "aws":
        p, err = NewAWSProvider(
            creds["role_arn"],
            creds["external_id"],
            creds["region"],
        )
    case "azure":
        p, err = NewAzureProvider(
            creds["tenant_id"],
            creds["client_id"],
            creds["client_secret"],
            creds["subscription_id"],
        )
    case "mosquitto":
        p, err = NewMosquittoProvider(
            creds["broker_url"],
            creds["admin_username"],
            creds["admin_password"],
        )
    default:
        return nil, fmt.Errorf("unsupported provider: %s", integration.Provider)
    }

    if err != nil {
        return nil, fmt.Errorf("create %s provider: %w", integration.Provider, err)
    }

    f.cache.Store(integration.ID, p)
    return p, nil
}
```

🐍 **Python**:

```python
# app/providers/factory.py
from functools import lru_cache
from typing import Any

from .base import IoTProvider
from .aws_provider import AwsIoTProvider
from .azure_provider import AzureIoTProvider
from .mosquitto_provider import MosquittoProvider


class ProviderFactory:
    """Create the right provider adapter based on tenant configuration."""

    def __init__(self, vault_client):
        self._vault = vault_client
        self._cache: dict[str, IoTProvider] = {}

    async def get_provider(self, integration: dict) -> IoTProvider:
        integration_id = integration["id"]

        if integration_id in self._cache:
            return self._cache[integration_id]

        # Fetch credentials from Vault
        creds = await self._vault.read_secret(integration["vault_secret_path"])

        # Create the appropriate provider
        provider = self._create_provider(integration["provider"], creds)

        self._cache[integration_id] = provider
        return provider

    def _create_provider(self, provider_type: str, creds: dict) -> IoTProvider:
        factories = {
            "aws": lambda c: AwsIoTProvider(
                role_arn=c["role_arn"],
                external_id=c["external_id"],
                region=c["region"],
            ),
            "azure": lambda c: AzureIoTProvider(
                tenant_id=c["tenant_id"],
                client_id=c["client_id"],
                client_secret=c["client_secret"],
                subscription_id=c["subscription_id"],
            ),
            "mosquitto": lambda c: MosquittoProvider(
                broker_url=c["broker_url"],
                admin_username=c["admin_username"],
                admin_password=c["admin_password"],
            ),
        }

        factory = factories.get(provider_type)
        if not factory:
            raise ValueError(f"Unsupported provider: {provider_type}")

        return factory(creds)

    def invalidate_cache(self, integration_id: str):
        """Call when credentials are rotated or integration is updated."""
        self._cache.pop(integration_id, None)
```

---

## 5.6 Testing Providers with a Mock Adapter

The PAL enables clean testing — mock the provider, test your business logic:

🐍 **Python — In-memory mock provider**:

```python
# tests/mocks/mock_provider.py
from app.providers.base import (
    IoTProvider, CreateDeviceRequest, DeviceResult,
    CreateCertRequest, CertificateResult,
    CreatePolicyRequest, PolicyResult,
    CreateRoutingRuleRequest, RoutingRuleResult,
)
import uuid


class MockIoTProvider(IoTProvider):
    """In-memory provider for testing — no cloud calls."""

    def __init__(self):
        self.devices: dict[str, dict] = {}
        self.certificates: dict[str, dict] = {}
        self.policies: dict[str, dict] = {}
        self.call_log: list[tuple[str, dict]] = []

    @property
    def provider_name(self) -> str:
        return "mock"

    def _log(self, method: str, **kwargs):
        self.call_log.append((method, kwargs))

    async def create_device(self, req: CreateDeviceRequest) -> DeviceResult:
        device_id = f"mock-device-{uuid.uuid4().hex[:8]}"
        self.devices[device_id] = {"name": req.device_name, "status": "active"}
        self._log("create_device", device_id=device_id, name=req.device_name)

        return DeviceResult(
            provider_device_id=device_id,
            endpoint="mqtt://mock-broker:1883",
        )

    async def create_certificate(self, req: CreateCertRequest) -> CertificateResult:
        cert_id = f"mock-cert-{uuid.uuid4().hex[:8]}"
        self.certificates[cert_id] = {"status": "active"}
        self._log("create_certificate", cert_id=cert_id)

        return CertificateResult(
            provider_cert_id=cert_id,
            cert_pem="-----BEGIN CERTIFICATE-----\nMOCK\n-----END CERTIFICATE-----",
            private_key_pem="-----BEGIN EC PRIVATE KEY-----\nMOCK\n-----END EC PRIVATE KEY-----",
        )

    # ... implement all other methods similarly

    async def validate_credentials(self) -> None:
        self._log("validate_credentials")

    async def get_endpoint(self) -> str:
        return "mqtt://mock-broker:1883"


# Usage in tests:
# provider = MockIoTProvider()
# device = await provider.create_device(CreateDeviceRequest(...))
# assert device.provider_device_id in provider.devices
# assert provider.call_log[-1][0] == "create_device"
```

---

## 5.7 Error Handling Across Providers

Each provider has different error types. The PAL normalizes them:

```python
# app/providers/errors.py

class ProviderError(Exception):
    """Base error for all provider operations."""
    def __init__(self, provider: str, operation: str, message: str, retryable: bool = False):
        self.provider = provider
        self.operation = operation
        self.retryable = retryable
        super().__init__(f"[{provider}] {operation}: {message}")


class DeviceAlreadyExistsError(ProviderError):
    pass

class CredentialsExpiredError(ProviderError):
    def __init__(self, provider: str):
        super().__init__(provider, "auth", "Credentials expired", retryable=False)

class RateLimitedError(ProviderError):
    def __init__(self, provider: str, retry_after: int = 60):
        self.retry_after = retry_after
        super().__init__(provider, "api", f"Rate limited, retry after {retry_after}s", retryable=True)

class ProviderUnavailableError(ProviderError):
    def __init__(self, provider: str):
        super().__init__(provider, "connectivity", "Provider unreachable", retryable=True)
```

---

## 5.8 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Provider interface** | Single contract all adapters implement |
| **AWS adapter** | Uses STS role assumption for cross-account access |
| **Mosquitto adapter** | Manages ACL files and certificates locally |
| **Provider factory** | Creates and caches adapters based on tenant config |
| **Mock provider** | Enables testing without cloud credentials |
| **Error normalization** | Consistent error types across providers |

### What's Next

In **Chapter 6**, we'll build the secrets management layer with HashiCorp Vault — storing cloud credentials, generating device certificates, and implementing dynamic secret rotation.
