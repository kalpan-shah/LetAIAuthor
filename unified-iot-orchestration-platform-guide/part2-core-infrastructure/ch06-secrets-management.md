# Chapter 6: Secrets Management with Vault

> *"The only truly secure system is one that is powered off, cast in a block of concrete, and sealed in a lead-lined room."* — Gene Spafford. *Our job is to get as close as practical.*

---

## 6.1 Introduction

An IoT orchestration platform handles some of the most sensitive material in any infrastructure:

- **Cloud provider credentials** (AWS IAM roles, Azure service principals)
- **Device private keys** (generated during provisioning)
- **Internal service tokens** (database passwords, API keys)
- **CA private keys** (for issuing device certificates)

Storing these in environment variables, config files, or even Kubernetes Secrets is insufficient for a multi-tenant platform. We need:

1. **Dynamic secrets** — credentials that are generated on-demand and automatically expire
2. **Audit trail** — every secret access logged with who, when, and what
3. **Fine-grained access control** — each service only accesses what it needs
4. **Automatic rotation** — secrets rotate without application restarts

**HashiCorp Vault** provides all four.

> 💡 **For Beginners**: Think of Vault like a bank vault for digital secrets. You can't just walk in and take whatever you want — you need to authenticate, you can only access your own safety deposit box, and the bank logs every visit.

---

## 6.2 Vault Architecture in Our Platform

```
┌─────────────────────────────────────────────────────────┐
│                    PLATFORM SERVICES                     │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Tenant   │  │ Device   │  │ Cert     │              │
│  │ Service  │  │ Service  │  │ Service  │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │             │                     │
│       └──────────────┼─────────────┘                     │
│                      │                                   │
│                      ▼                                   │
│           ┌──────────────────┐                           │
│           │  Vault Client    │  (Authenticated per-svc)  │
│           └────────┬─────────┘                           │
└────────────────────┼─────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│                   HASHICORP VAULT                         │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │ KV v2: iot-platform/credentials/                  │   │
│  │   tenant-{id}-aws     → role_arn, external_id     │   │
│  │   tenant-{id}-azure   → client_id, client_secret  │   │
│  │   tenant-{id}-mqtt    → broker_url, password      │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │ PKI: iot-platform/pki/                            │   │
│  │   Root CA → "IoT Platform CA"                     │   │
│  │   Role: device-cert → EC P-256, 1yr max TTL       │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │ Transit: iot-platform/transit/                    │   │
│  │   Key: audit-signing → HMAC for audit log         │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │ Auth: Kubernetes / AppRole                        │   │
│  │   Role: tenant-service → read credentials         │   │
│  │   Role: cert-service   → issue PKI certs          │   │
│  │   Role: audit-service  → transit for signing      │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### Vault Engines We Use

| Engine | Path | Purpose |
|--------|------|---------|
| **KV v2** | `iot-platform/credentials/` | Store cloud provider credentials per tenant |
| **PKI** | `iot-platform/pki/` | Issue and revoke device X.509 certificates |
| **Transit** | `iot-platform/transit/` | HMAC signing for audit log integrity |
| **Kubernetes Auth** | `auth/kubernetes/` | Authenticate platform services to Vault |

---

## 6.3 Vault Client Implementation

🔵 **Go — Vault Client**:

```go
// internal/vault/client.go
package vault

import (
    "context"
    "fmt"
    "time"

    vaultapi "github.com/hashicorp/vault/api"
)

type Client struct {
    client *vaultapi.Client
}

func NewClient(addr, token string) (*Client, error) {
    config := vaultapi.DefaultConfig()
    config.Address = addr
    config.Timeout = 10 * time.Second

    client, err := vaultapi.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("create vault client: %w", err)
    }

    client.SetToken(token)
    return &Client{client: client}, nil
}

// ── KV v2 Operations ──────────────────────────────────

func (c *Client) ReadSecret(ctx context.Context, path string) (map[string]string, error) {
    secret, err := c.client.KVv2("iot-platform/credentials").Get(ctx, path)
    if err != nil {
        return nil, fmt.Errorf("read secret %s: %w", path, err)
    }

    result := make(map[string]string)
    for k, v := range secret.Data {
        if str, ok := v.(string); ok {
            result[k] = str
        }
    }
    return result, nil
}

func (c *Client) WriteSecret(ctx context.Context, path string, data map[string]interface{}) error {
    _, err := c.client.KVv2("iot-platform/credentials").Put(ctx, path, data)
    if err != nil {
        return fmt.Errorf("write secret %s: %w", path, err)
    }
    return nil
}

func (c *Client) DeleteSecret(ctx context.Context, path string) error {
    return c.client.KVv2("iot-platform/credentials").Delete(ctx, path)
}

// ── PKI Operations ────────────────────────────────────

type CertificateResponse struct {
    CertPEM       string
    PrivateKeyPEM string
    CaPEM         string
    SerialNumber  string
    Expiration    time.Time
}

func (c *Client) IssueCertificate(ctx context.Context, commonName string, ttl string) (*CertificateResponse, error) {
    secret, err := c.client.Logical().WriteWithContext(ctx,
        "iot-platform/pki/issue/device-cert",
        map[string]interface{}{
            "common_name": commonName,
            "ttl":         ttl,
        },
    )
    if err != nil {
        return nil, fmt.Errorf("issue certificate for %s: %w", commonName, err)
    }

    return &CertificateResponse{
        CertPEM:       secret.Data["certificate"].(string),
        PrivateKeyPEM: secret.Data["private_key"].(string),
        CaPEM:         secret.Data["issuing_ca"].(string),
        SerialNumber:  secret.Data["serial_number"].(string),
    }, nil
}

func (c *Client) RevokeCertificate(ctx context.Context, serialNumber string) error {
    _, err := c.client.Logical().WriteWithContext(ctx,
        "iot-platform/pki/revoke",
        map[string]interface{}{
            "serial_number": serialNumber,
        },
    )
    return err
}

// ── Transit Operations ────────────────────────────────

func (c *Client) SignData(ctx context.Context, keyName string, data []byte) (string, error) {
    secret, err := c.client.Logical().WriteWithContext(ctx,
        fmt.Sprintf("iot-platform/transit/hmac/%s", keyName),
        map[string]interface{}{
            "input": data,
        },
    )
    if err != nil {
        return "", fmt.Errorf("sign data: %w", err)
    }
    return secret.Data["hmac"].(string), nil
}
```

🐍 **Python — Vault Client**:

```python
# app/vault/client.py
import hvac
from dataclasses import dataclass


@dataclass
class CertificateResponse:
    cert_pem: str
    private_key_pem: str
    ca_pem: str
    serial_number: str
    expiration: str | None = None


class VaultClient:
    """HashiCorp Vault client for secrets, PKI, and transit operations."""

    def __init__(self, addr: str, token: str):
        self._client = hvac.Client(url=addr, token=token)

    def is_authenticated(self) -> bool:
        return self._client.is_authenticated()

    # ── KV v2 Operations ──

    async def read_secret(self, path: str) -> dict:
        """Read cloud credentials from KV v2."""
        response = self._client.secrets.kv.v2.read_secret_version(
            path=path,
            mount_point="iot-platform/credentials",
        )
        return response["data"]["data"]

    async def write_secret(self, path: str, data: dict) -> None:
        """Store cloud credentials in KV v2."""
        self._client.secrets.kv.v2.create_or_update_secret(
            path=path,
            secret=data,
            mount_point="iot-platform/credentials",
        )

    async def delete_secret(self, path: str) -> None:
        """Remove credentials from KV v2."""
        self._client.secrets.kv.v2.delete_metadata_and_all_versions(
            path=path,
            mount_point="iot-platform/credentials",
        )

    # ── PKI Operations ──

    async def issue_certificate(
        self, common_name: str, ttl: str = "8760h"
    ) -> CertificateResponse:
        """Issue a device certificate using Vault's PKI engine."""
        response = self._client.secrets.pki.generate_certificate(
            name="device-cert",
            common_name=common_name,
            extra_params={"ttl": ttl},
            mount_point="iot-platform/pki",
        )

        data = response["data"]
        return CertificateResponse(
            cert_pem=data["certificate"],
            private_key_pem=data["private_key"],
            ca_pem=data["issuing_ca"],
            serial_number=data["serial_number"],
            expiration=data.get("expiration"),
        )

    async def revoke_certificate(self, serial_number: str) -> None:
        """Revoke a certificate by serial number."""
        self._client.secrets.pki.revoke_certificate(
            serial_number=serial_number,
            mount_point="iot-platform/pki",
        )

    # ── Transit Operations ──

    async def sign_data(self, key_name: str, plaintext: bytes) -> str:
        """HMAC-sign data using Vault Transit engine."""
        import base64

        response = self._client.secrets.transit.generate_hmac(
            name=key_name,
            hash_input=base64.b64encode(plaintext).decode(),
            mount_point="iot-platform/transit",
        )
        return response["data"]["hmac"]
```

---

## 6.4 Kubernetes Authentication

In production, services authenticate to Vault using their Kubernetes service account — no static tokens:

```bash
# Configure Vault to accept Kubernetes auth
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a role for each service
vault write auth/kubernetes/role/tenant-service \
    bound_service_account_names=tenant-service \
    bound_service_account_namespaces=iot-platform \
    policies=tenant-service-policy \
    ttl=1h

vault write auth/kubernetes/role/cert-service \
    bound_service_account_names=cert-service \
    bound_service_account_namespaces=iot-platform \
    policies=cert-service-policy \
    ttl=1h
```

### Service-Specific Policies

```hcl
# policies/tenant-service-policy.hcl
# Tenant service can read/write cloud credentials, but cannot issue certs

path "iot-platform/credentials/data/*" {
  capabilities = ["create", "read", "update", "delete"]
}

path "iot-platform/credentials/metadata/*" {
  capabilities = ["list", "read", "delete"]
}

# Deny PKI access
path "iot-platform/pki/*" {
  capabilities = ["deny"]
}
```

```hcl
# policies/cert-service-policy.hcl
# Certificate service can issue certs, but cannot read cloud credentials

path "iot-platform/pki/issue/device-cert" {
  capabilities = ["create", "update"]
}

path "iot-platform/pki/revoke" {
  capabilities = ["create", "update"]
}

# Can read but not write credentials (to validate certs with provider)
path "iot-platform/credentials/data/*" {
  capabilities = ["read"]
}
```

> ⚠️ **Principle of Least Privilege**: Each service gets exactly the Vault permissions it needs — nothing more. The tenant service can manage credentials but cannot issue certificates. The certificate service can issue certs but cannot write credentials.

> ☕ *For Java/C# developers*: This is similar to Spring Cloud Vault or Azure Key Vault with managed identities, but Vault's policy language is more granular. In .NET, the closest equivalent is Azure Managed Identity + Key Vault access policies, but without the built-in PKI engine.

---

## 6.5 Credential Rotation

When cloud credentials are rotated, the platform needs to update Vault and invalidate cached provider instances:

🐍 **Python — Credential rotation flow**:

```python
# services/integration_service.py

class IntegrationService:
    def __init__(self, db_pool, vault_client, provider_factory, event_publisher):
        self.db = db_pool
        self.vault = vault_client
        self.providers = provider_factory
        self.events = event_publisher

    async def rotate_credentials(
        self, tenant_id: str, integration_id: str, new_credentials: dict
    ) -> None:
        """Rotate cloud provider credentials for a tenant integration."""

        # 1. Write new credentials to Vault
        secret_path = f"tenant-{tenant_id}-{integration_id}"
        await self.vault.write_secret(secret_path, new_credentials)

        # 2. Invalidate cached provider instance (forces re-creation with new creds)
        self.providers.invalidate_cache(integration_id)

        # 3. Validate new credentials work
        integration = await self._get_integration(integration_id)
        provider = await self.providers.get_provider(integration)

        try:
            await provider.validate_credentials()
        except Exception as e:
            # Rollback: credentials don't work, keep old version
            # KV v2 preserves old versions — rollback is automatic
            await self.vault.rollback_secret(secret_path)
            self.providers.invalidate_cache(integration_id)
            raise CredentialValidationError(str(e))

        # 4. Update integration status
        async with self.db.acquire() as conn:
            await conn.execute(
                """UPDATE cloud_integrations
                   SET last_validated_at = NOW(), status = 'active', last_error = NULL
                   WHERE id = $1""",
                integration_id,
            )

        # 5. Emit audit event
        await self.events.publish_audit_event(
            tenant_id=tenant_id,
            action="credentials.rotated",
            actor="system",
            resource_type="integration",
            resource_id=integration_id,
            outcome="success",
        )
```

---

## 6.6 Vault PKI vs. External CA

| Approach | When to Use | Pros | Cons |
|----------|------------|------|------|
| **Vault PKI** | Most deployments | Integrated, automated, easy rotation | Requires Vault HA for production |
| **CFSSL** | Air-gapped environments | Simple, open-source | No built-in rotation |
| **step-ca** | Smaller deployments | ACME support, lightweight | Less integration with Vault |
| **Cloud CA** (e.g., AWS IoT) | AWS-only deployments | Zero maintenance | Provider lock-in |

For our platform, **Vault PKI** is the default because:
1. We already run Vault for credential management
2. PKI integrates with the same auth and policy system
3. Certificate revocation lists (CRL) are automatic
4. TTLs enforce automatic expiration

---

## 6.7 Audit Log Integrity with Transit

Every audit log entry is HMAC-signed using Vault's Transit engine to detect tampering:

```python
# services/audit_service.py
import json
import hashlib


class AuditService:
    def __init__(self, vault_client, db_pool):
        self.vault = vault_client
        self.db = db_pool

    async def create_audit_entry(self, entry: dict) -> str:
        """Create a tamper-evident audit log entry."""
        # Serialize the entry deterministically
        canonical = json.dumps(entry, sort_keys=True, separators=(",", ":"))

        # Sign with Vault Transit
        hmac = await self.vault.sign_data(
            key_name="audit-signing",
            plaintext=canonical.encode(),
        )

        # Store entry with HMAC
        entry["integrity_hmac"] = hmac

        async with self.db.acquire() as conn:
            row = await conn.fetchrow(
                """INSERT INTO audit_logs
                   (tenant_id, action, actor, resource_type, resource_id,
                    outcome, details)
                   VALUES ($1, $2, $3, $4, $5, $6, $7)
                   RETURNING id""",
                entry["tenant_id"], entry["action"], entry["actor"],
                entry["resource_type"], entry.get("resource_id"),
                entry["outcome"],
                json.dumps({"hmac": hmac, **entry.get("details", {})}),
            )
            return str(row["id"])

    async def verify_audit_entry(self, entry_id: str) -> bool:
        """Verify that an audit log entry hasn't been tampered with."""
        async with self.db.acquire() as conn:
            row = await conn.fetchrow(
                "SELECT * FROM audit_logs WHERE id = $1", entry_id
            )

        if not row:
            return False

        # Reconstruct the original entry
        stored_hmac = row["details"].get("hmac")
        original = {
            "tenant_id": str(row["tenant_id"]),
            "action": row["action"],
            "actor": row["actor"],
            "resource_type": row["resource_type"],
            "resource_id": str(row["resource_id"]) if row["resource_id"] else None,
            "outcome": row["outcome"],
        }

        canonical = json.dumps(original, sort_keys=True, separators=(",", ":"))
        expected_hmac = await self.vault.sign_data(
            "audit-signing", canonical.encode()
        )

        return stored_hmac == expected_hmac
```

---

## 6.8 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Vault engines** | KV v2 for credentials, PKI for certs, Transit for signing |
| **Kubernetes auth** | Services authenticate via service accounts — no static tokens |
| **Least privilege** | Each service has its own Vault policy |
| **Credential rotation** | Update Vault → invalidate cache → validate → audit |
| **PKI management** | Vault issues/revokes device certs with auto-expiry |
| **Audit integrity** | HMAC-signed log entries via Transit engine |

### What's Next

In **Chapter 7**, we begin building the platform services — starting with the **Tenant Management Service**, the foundation all other services depend on.
