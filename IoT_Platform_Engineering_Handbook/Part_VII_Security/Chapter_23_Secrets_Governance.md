# Chapter 23: Secrets Governance & Compliance

> **Part VII — Security (SecOps)**

---

## 23.1 The Seven Golden Rules

**Rule 1: No secrets in code.**
No hardcoded passwords, API keys, or certificates anywhere in the repository. Enforced by GitLeaks in CI and pre-commit hooks. Any commit containing a secret pattern fails the pipeline immediately.

**Rule 2: No secrets in environment variables (production).**
In Kubernetes, use the Vault Agent Injector to mount secrets as files at pod startup. The application reads files — not env vars. Env vars are visible in `kubectl describe pod` output.

**Rule 3: No secrets in logs.**
The structured logger middleware scans all log fields for names containing `password`, `secret`, `key`, `token`, `private`, `cert`, and replaces their values with `[REDACTED]`. This is enforced at the logging layer, not by individual developers.

**Rule 4: No secrets in the database.**
The database stores Vault path references (`secret/data/certs/{id}`), serial numbers, and fingerprints. Never the certificate content or private keys. If the database is breached, no crypto material is exposed.

**Rule 5: Private keys delivered once.**
The provisioning API includes `private_key_pem` in the response exactly once. After that response is sent, the private key is only accessible through Vault — it never appears in the API again. The UI prominently warns users to save it immediately.

**Rule 6: All credentials have expiry.**
No permanent static credentials anywhere in the system:

| Credential Type | TTL |
|---|---|
| JWT access token | 24 hours |
| JWT refresh token | 7 days |
| Vault AppRole secret ID | 30 days |
| Vault-issued service token | 1 hour |
| AWS STS assumed-role credentials | 15 minutes |
| Azure Service Principal token | 1 hour |
| Database dynamic credentials | 15 minutes |
| Device X.509 certificate | 1 year (rotated at 30d warning) |

**Rule 7: Secret access is audited.**
Vault's audit log device records every read, write, and delete of every secret path — who, what operation, when, from which IP. This log is shipped to a SIEM (Splunk, Datadog, or Loki) for retention and alerting.

---

## 23.2 Vault Policy: Least Privilege

```hcl
# deploy/terraform/modules/vault-config/policies.tf

# Platform API policy — minimum required paths only
resource "vault_policy" "iot_platform_api" {
  name = "iot-platform-api"
  policy = <<-EOT
    # Issue device certificates
    path "pki/issue/device-cert" {
      capabilities = ["create", "update"]
    }

    # Revoke certificates (add to CRL)
    path "pki/revoke" {
      capabilities = ["create", "update"]
    }

    # Read and write provider credentials per tenant
    path "secret/data/providers/+/*" {
      capabilities = ["create", "read", "update", "delete"]
    }

    # Store and retrieve certificate key bundles
    path "secret/data/certs/+/*" {
      capabilities = ["create", "read", "delete"]
    }

    # Read short-lived database credentials
    path "database/creds/iot-platform" {
      capabilities = ["read"]
    }

    # EXPLICITLY DENIED (defence in depth):
    path "sys/*"     { capabilities = ["deny"] }
    path "auth/+/+/+/*" { capabilities = ["deny"] }
  EOT
}

# Worker policy — read-only, fewer paths
resource "vault_policy" "iot_platform_worker" {
  name = "iot-platform-worker"
  policy = <<-EOT
    path "pki/revoke" {
      capabilities = ["create", "update"]
    }
    path "secret/data/certs/+/*" {
      capabilities = ["read", "delete"]
    }
    path "database/creds/iot-platform" {
      capabilities = ["read"]
    }
  EOT
}
```

---

## 23.3 Vault Kubernetes Auth (Production Setup)

```bash
# Run once after cluster bootstrap

# 1. Enable Kubernetes auth method
vault auth enable kubernetes

# 2. Configure Vault to validate K8s service account tokens
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"

# 3. Bind platform API service account to policy
vault write auth/kubernetes/role/iot-platform-api \
  bound_service_account_names=iot-platform-api \
  bound_service_account_namespaces=iot-platform \
  policies=iot-platform-api \
  ttl=1h

# 4. Bind worker service account to worker policy
vault write auth/kubernetes/role/iot-platform-worker \
  bound_service_account_names=iot-platform-worker \
  bound_service_account_namespaces=iot-platform \
  policies=iot-platform-worker \
  ttl=1h
```

**Pod annotation for automatic Vault secret injection:**
```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "iot-platform-api"

    # Inject database credentials
    vault.hashicorp.com/agent-inject-secret-db: "database/creds/iot-platform"
    vault.hashicorp.com/agent-inject-template-db: |
      {{- with secret "database/creds/iot-platform" -}}
      DATABASE_URL=postgres://{{ .Data.username }}:{{ .Data.password }}@postgres:5432/iotplatform?sslmode=require
      {{- end }}

    # Inject NATS credentials
    vault.hashicorp.com/agent-inject-secret-nats: "secret/data/nats/credentials"
    vault.hashicorp.com/agent-inject-template-nats: |
      {{- with secret "secret/data/nats/credentials" -}}
      NATS_URL={{ .Data.data.url }}
      {{- end }}
```

---

## 23.4 Secret Scanning in CI

```yaml
# .github/workflows/ci.yml — security stage

  security:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }  # Full history for GitLeaks

      - name: GitLeaks — scan for secrets in code and history
        uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

      - name: gosec — Go static security analysis
        uses: securego/gosec@master
        with:
          args: >
            -severity medium
            -exclude G104
            ./...

      - name: Trivy — filesystem vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref:  "."
          severity:  CRITICAL,HIGH
          exit-code: "1"
```

**Pre-commit hook (installed via `pre-commit install`):**
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
```

---

## 23.5 Compliance Controls Summary

| Control Category | Implementation |
|---|---|
| Secret rotation | Vault leases auto-expire; device certs rotated by ExpiryMonitor (6h scan) |
| Access auditing | Vault audit log (every secret access) + platform `audit_logs` table |
| Least privilege | AppRole/K8s auth policies grant minimum required Vault paths only |
| No persistent cloud credentials | AWS: 15m STS tokens; DB: 15m Vault dynamic credentials |
| Certificate revocation | Vault CRL updated immediately on revocation; OCSP available |
| Immutable audit trail | `audit_logs` table: app DB user granted no `UPDATE` or `DELETE` |
| Secret scanning | GitLeaks in CI (every PR) + pre-commit hook (every commit) |
| Dependency CVEs | Trivy scan in CI; build fails on CRITICAL/HIGH unpatched CVEs |
| TLS everywhere | No plaintext connections in production; `sslmode=require` for PostgreSQL |
