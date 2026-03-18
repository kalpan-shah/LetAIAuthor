# Chapter 13: Certificate Lifecycle Service

> **Part IV — Platform Services**

The Certificate Service requests signed X.509 certificates from Vault's PKI engine, stores only metadata in PostgreSQL, keeps the key bundle in Vault, and delivers the private key exactly once — in the provisioning response.

---

## 13.1 Certificate Service

```go
// internal/certificate/service.go
package certificate

import (
	"context"
	"crypto/sha256"
	"crypto/x509"
	"encoding/pem"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/your-org/iot-platform/internal/db"
	"github.com/your-org/iot-platform/internal/vault"
)

type Bundle struct {
	ID             uuid.UUID
	DeviceID       uuid.UUID
	TenantID       uuid.UUID
	SerialNumber   string
	Fingerprint    string
	CertificatePEM string
	PrivateKeyPEM  string  // DELIVERED ONCE — never stored in DB
	CACertPEM      string
	NotBefore      time.Time
	NotAfter       time.Time
}

type IssueReq struct {
	TenantID   uuid.UUID
	DeviceID   uuid.UUID
	DeviceName string
	TTL        string // default: 8760h (1 year)
}

type Service struct {
	vault *vault.Client
	db    *db.Queries
}

func NewService(v *vault.Client, q *db.Queries) *Service {
	return &Service{vault: v, db: q}
}

// Issue requests and stores a new device certificate.
// The private key is returned in Bundle but NEVER written to the DB.
func (s *Service) Issue(ctx context.Context, req IssueReq) (*Bundle, error) {
	ttl := req.TTL
	if ttl == "" { ttl = "8760h" }

	// CN scopes the cert under tenant + device namespace
	cn := fmt.Sprintf("%s.%s.devices.iotplatform.local", req.DeviceID, req.TenantID)

	// Request from Vault PKI
	vBundle, err := s.vault.IssueCertificate(ctx, cn, ttl)
	if err != nil {
		return nil, fmt.Errorf("vault pki issue: %w", err)
	}

	// Parse cert to extract metadata
	block, _ := pem.Decode([]byte(vBundle.CertificatePEM))
	cert, err := x509.ParseCertificate(block.Bytes)
	if err != nil {
		return nil, fmt.Errorf("parsing cert: %w", err)
	}
	fingerprint := fmt.Sprintf("%x", sha256.Sum256(cert.Raw))
	certID := uuid.New()

	// Store key bundle in Vault KV (for potential re-delivery)
	vaultPath := fmt.Sprintf("secret/data/certs/%s", certID)
	if err := s.vault.StoreSecret(ctx, vaultPath, map[string]interface{}{
		"certificate": vBundle.CertificatePEM,
		"private_key": vBundle.PrivateKeyPEM,
		"ca":          vBundle.CACertPEM,
	}); err != nil {
		return nil, fmt.Errorf("storing cert in vault: %w", err)
	}

	// Store METADATA ONLY in PostgreSQL
	_, err = s.db.CreateCertificate(ctx, db.CreateCertificateParams{
		ID: certID, DeviceID: req.DeviceID, TenantID: req.TenantID,
		SerialNumber: vBundle.SerialNumber, VaultPath: vaultPath,
		SubjectCN: cn, Fingerprint: fingerprint,
		NotBefore: cert.NotBefore, NotAfter: cert.NotAfter,
	})
	if err != nil {
		return nil, fmt.Errorf("storing cert record: %w", err)
	}

	return &Bundle{
		ID: certID, DeviceID: req.DeviceID, TenantID: req.TenantID,
		SerialNumber: vBundle.SerialNumber, Fingerprint: fingerprint,
		CertificatePEM: vBundle.CertificatePEM,
		PrivateKeyPEM:  vBundle.PrivateKeyPEM, // ONE TIME
		CACertPEM:      vBundle.CACertPEM,
		NotBefore: cert.NotBefore, NotAfter: cert.NotAfter,
	}, nil
}

// Revoke marks a cert in Vault's CRL and updates the DB status.
func (s *Service) Revoke(ctx context.Context, certID uuid.UUID, reason string) error {
	certRow, err := s.db.GetCertificate(ctx, certID)
	if err != nil {
		return err
	}
	if err := s.vault.RevokeCertificate(ctx, certRow.SerialNumber); err != nil {
		return fmt.Errorf("vault revoke: %w", err)
	}
	_, err = s.db.RevokeCertificate(ctx, db.RevokeCertificateParams{
		ID: certID, RevocationReason: reason,
	})
	return err
}
```

---

## 13.2 Certificate Expiry Monitor

```go
// internal/certificate/expiry_monitor.go
package certificate

import (
	"context"
	"log/slog"
	"time"

	"github.com/your-org/iot-platform/internal/events"
)

type ExpiryMonitor struct {
	svc         *Service
	events      *events.Publisher
	warningDays int
}

func NewExpiryMonitor(svc *Service, pub *events.Publisher, warningDays int) *ExpiryMonitor {
	if warningDays == 0 { warningDays = 30 }
	return &ExpiryMonitor{svc: svc, events: pub, warningDays: warningDays}
}

func (m *ExpiryMonitor) Run(ctx context.Context) {
	ticker := time.NewTicker(6 * time.Hour)
	defer ticker.Stop()
	m.scan(ctx) // Run immediately on start
	for {
		select {
		case <-ctx.Done(): return
		case <-ticker.C:   m.scan(ctx)
		}
	}
}

func (m *ExpiryMonitor) scan(ctx context.Context) {
	expiring, err := m.svc.db.GetCertificatesExpiringSoon(ctx, m.warningDays)
	if err != nil {
		slog.Error("cert expiry scan failed", "error", err)
		return
	}
	for _, row := range expiring {
		daysLeft := int(time.Until(row.NotAfter).Hours() / 24)
		slog.Warn("certificate expiring soon",
			"device_id",      row.DeviceID,
			"cert_id",        row.CertID,
			"expires_at",     row.NotAfter,
			"days_remaining", daysLeft,
		)
		m.events.CertificateExpiringSoon(ctx,
			row.CertID.String(), row.DeviceID.String(),
			row.TenantID.String(), row.NotAfter, daysLeft,
		)
	}
	slog.Info("cert expiry scan complete", "expiring_count", len(expiring))
}
```
