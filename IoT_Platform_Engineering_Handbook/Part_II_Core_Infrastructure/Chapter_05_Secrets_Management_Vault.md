# Chapter 5: Secrets Management with HashiCorp Vault

> **Part II — Core Infrastructure**

Vault is the security backbone of the platform. It issues device certificates, stores provider credentials, and generates short-lived database passwords. Nothing sensitive ever appears in the application database, environment variables (in production), or log files.

---

## 5.1 Vault Architecture

```
Platform Services
    │
    │  AppRole authentication (Role ID + Secret ID)
    │  → Vault returns short-lived Vault token (1h TTL)
    ▼
Vault
    ├── PKI Secrets Engine (/pki)
    │   ├── Root CA (internal, self-signed, EC P-384)
    │   └── Role: device-cert  (EC P-256, 1-year TTL, client_flag=true)
    │       → Platform calls: vault write pki/issue/device-cert common_name=...
    │       ← Vault returns: certificate PEM, private key PEM, CA chain PEM
    │
    ├── KV v2 (/secret)
    │   ├── secret/data/providers/{tenant_id}/aws   → AWS role ARN, external ID
    │   ├── secret/data/providers/{tenant_id}/azure → Azure SP credentials
    │   └── secret/data/certs/{cert_id}             → cert PEM + key PEM
    │
    └── Database Secrets Engine (/database)  [production]
        → vault read database/creds/iot-platform
        ← Vault returns: temp username + password (15min TTL, auto-revoked)
```

---

## 5.2 Vault Client Implementation (Go)

```go
// internal/vault/client.go
package vault

import (
	"context"
	"fmt"
	"os"
	"sync"
	"time"

	vaultapi "github.com/hashicorp/vault/api"
)

// Client wraps the Vault API with AppRole re-authentication.
// In production, the Vault token is short-lived (1h).
// The client automatically re-authenticates when the token expires.
type Client struct {
	api         *vaultapi.Client
	mu          sync.Mutex
	tokenExpiry time.Time
	roleID      string
	secretID    string
}

func NewClient(addr string) (*Client, error) {
	cfg := vaultapi.DefaultConfig()
	cfg.Address = addr
	api, err := vaultapi.NewClient(cfg)
	if err != nil {
		return nil, fmt.Errorf("creating vault client: %w", err)
	}

	c := &Client{
		api:      api,
		roleID:   os.Getenv("VAULT_ROLE_ID"),
		secretID: os.Getenv("VAULT_SECRET_ID"),
	}

	// Development: use VAULT_TOKEN directly (dev mode)
	if token := os.Getenv("VAULT_TOKEN"); token != "" {
		api.SetToken(token)
		c.tokenExpiry = time.Now().Add(24 * time.Hour)
		return c, nil
	}

	// Production: authenticate with AppRole
	if err := c.authenticate(context.Background()); err != nil {
		return nil, fmt.Errorf("vault auth: %w", err)
	}
	return c, nil
}

func (c *Client) authenticate(ctx context.Context) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	secret, err := c.api.Auth().Login(ctx, &vaultapi.AppRoleLoginInput{
		RoleID:   c.roleID,
		SecretID: c.secretID,
	})
	if err != nil {
		return fmt.Errorf("vault approle login: %w", err)
	}
	_ = secret
	c.tokenExpiry = time.Now().Add(55 * time.Minute) // Re-auth at 55min (token=1h)
	return nil
}

func (c *Client) ensureAuth(ctx context.Context) error {
	if time.Until(c.tokenExpiry) > 5*time.Minute {
		return nil
	}
	return c.authenticate(ctx)
}

// IssueCertificate requests a new device certificate from Vault PKI.
func (c *Client) IssueCertificate(ctx context.Context, commonName, ttl string) (*CertBundle, error) {
	if err := c.ensureAuth(ctx); err != nil {
		return nil, err
	}
	secret, err := c.api.Logical().WriteWithContext(ctx,
		"pki/issue/device-cert",
		map[string]interface{}{
			"common_name": commonName,
			"ttl":         ttl,
		},
	)
	if err != nil {
		return nil, fmt.Errorf("vault pki issue: %w", err)
	}
	data := secret.Data
	return &CertBundle{
		CertificatePEM: data["certificate"].(string),
		PrivateKeyPEM:  data["private_key"].(string),
		CACertPEM:      data["issuing_ca"].(string),
		SerialNumber:   data["serial_number"].(string),
	}, nil
}

// RevokeCertificate adds a certificate to Vault's CRL by serial number.
func (c *Client) RevokeCertificate(ctx context.Context, serialNumber string) error {
	if err := c.ensureAuth(ctx); err != nil {
		return err
	}
	_, err := c.api.Logical().WriteWithContext(ctx, "pki/revoke",
		map[string]interface{}{"serial_number": serialNumber})
	return err
}

// StoreSecret saves data to Vault KV v2.
func (c *Client) StoreSecret(ctx context.Context, path string, data map[string]interface{}) error {
	if err := c.ensureAuth(ctx); err != nil {
		return err
	}
	_, err := c.api.Logical().WriteWithContext(ctx, path,
		map[string]interface{}{"data": data})
	return err
}

// ReadSecret retrieves data from Vault KV v2.
func (c *Client) ReadSecret(ctx context.Context, path string) (map[string]interface{}, error) {
	if err := c.ensureAuth(ctx); err != nil {
		return nil, err
	}
	secret, err := c.api.Logical().ReadWithContext(ctx, path)
	if err != nil {
		return nil, err
	}
	if secret == nil {
		return nil, ErrSecretNotFound
	}
	data, _ := secret.Data["data"].(map[string]interface{})
	return data, nil
}

type CertBundle struct {
	CertificatePEM string
	PrivateKeyPEM  string
	CACertPEM      string
	SerialNumber   string
}

var ErrSecretNotFound = fmt.Errorf("secret not found in vault")
```

---

## 5.3 Production: Vault Kubernetes Auth

In production, pods authenticate to Vault using Kubernetes service account tokens:

```bash
# Enable Kubernetes auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host='https://kubernetes.default.svc:443'

# Bind service account to policy
vault write auth/kubernetes/role/iot-platform \
  bound_service_account_names=iot-platform-api \
  bound_service_account_namespaces=iot-platform \
  policies=iot-platform \
  ttl=1h
```

```yaml
# Pod annotations for Vault Agent Injector (auto-mounts secrets as files)
annotations:
  vault.hashicorp.com/agent-inject: 'true'
  vault.hashicorp.com/role: 'iot-platform'
  vault.hashicorp.com/agent-inject-secret-db: 'database/creds/iot-platform'
  vault.hashicorp.com/agent-inject-template-db: |
    {{- with secret "database/creds/iot-platform" -}}
    DATABASE_URL=postgres://{{ .Data.username }}:{{ .Data.password }}@postgres:5432/iotplatform
    {{- end }}
```

> **⚠️ WARNING:** Never use `VAULT_TOKEN=root` or Vault dev mode in production. Production Vault requires auto-unseal (AWS KMS, Azure Key Vault, GCP KMS) or Shamir secret sharing with minimum 3 key holders.
