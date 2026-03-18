# Chapter 12: Device Lifecycle Service

> **Part IV — Platform Services**

The Device Lifecycle Service is the orchestrator. It coordinates certificate issuance, policy generation, and provider registration into a single transactional workflow. If any step fails, it rolls back to leave the platform in a consistent state.

---

## 12.1 Provisioning Workflow

```
Provision() call sequence
────────────────────────
① CheckDeviceQuota         → error if tenant at limit or suspended
② Generate device UUID
③ CertificateService.Issue → Vault PKI: sign X.509 cert (EC P-256, 1yr)
④ PolicyEngine.Generate    → build publish/subscribe topic lists
⑤ IoTProvider.CreateDevice → register with broker/cloud
   └─ failure ─► Rollback: CertificateService.Revoke
⑥ db.CreateDevice          → persist device record (status=active)
⑦ db.UpdateDeviceCert      → link cert to device
⑧ events.DeviceProvisioned → NATS notification
⑨ audit.Log                → immutable audit record
⑩ Return device + cert bundle (private key delivered ONCE)
```

---

## 12.2 Device Service Implementation

```go
// internal/device/service.go
package device

import (
	"context"
	"fmt"

	"github.com/google/uuid"
	"github.com/your-org/iot-platform/internal/audit"
	"github.com/your-org/iot-platform/internal/certificate"
	"github.com/your-org/iot-platform/internal/db"
	"github.com/your-org/iot-platform/internal/events"
	"github.com/your-org/iot-platform/internal/policy"
	"github.com/your-org/iot-platform/internal/providers"
	"github.com/your-org/iot-platform/internal/tenant"
)

type Service struct {
	db       *db.Queries
	certs    *certificate.Service
	policies *policy.Engine
	tenants  *tenant.Service
	auditor  *audit.Service
	events   *events.Publisher
}

type ProvisionReq struct {
	TenantID    uuid.UUID
	Name        string
	Description string
	Metadata    map[string]string
	ActorID     string
}

type ProvisionResult struct {
	Device     db.Device
	CertBundle *certificate.Bundle // PrivateKeyPEM is ONE TIME only
	Endpoint   string
}

func (s *Service) Provision(ctx context.Context, req ProvisionReq) (*ProvisionResult, error) {
	// ① Quota check
	if err := s.tenants.CheckDeviceQuota(ctx, req.TenantID); err != nil {
		return nil, fmt.Errorf("quota: %w", err)
	}

	tenantRow, err := s.db.GetTenant(ctx, req.TenantID)
	if err != nil {
		return nil, fmt.Errorf("loading tenant: %w", err)
	}

	// ② Device ID
	deviceID := uuid.New()

	// ③ Issue certificate
	bundle, err := s.certs.Issue(ctx, certificate.IssueReq{
		TenantID:   req.TenantID,
		DeviceID:   deviceID,
		DeviceName: req.Name,
	})
	if err != nil {
		return nil, fmt.Errorf("issuing cert: %w", err)
	}

	// ④ Generate policy rules
	policyRules := s.policies.ForDevice(req.TenantID.String(), deviceID.String())

	// ⑤ Register with provider
	provider, err := providers.Get(providers.ProviderType(tenantRow.Provider))
	if err != nil {
		return nil, fmt.Errorf("provider unavailable: %w", err)
	}

	creds, err := s.loadProviderCreds(ctx, req.TenantID, tenantRow.Provider)
	if err != nil {
		return nil, fmt.Errorf("loading creds: %w", err)
	}

	pDevice, err := provider.CreateDevice(ctx, providers.CreateDeviceReq{
		Credentials: creds,
		TenantID:    req.TenantID.String(),
		DeviceID:    deviceID.String(),
		DeviceName:  req.Name,
		CertPEM:     bundle.CertificatePEM,
		PolicyRules: policyRules,
	})
	if err != nil {
		// Rollback: revoke the certificate we just issued
		_ = s.certs.Revoke(ctx, bundle.ID, "provisioning_failed")
		s.auditor.Log(ctx, audit.Entry{
			TenantID: req.TenantID, ActorID: req.ActorID,
			Action: "device.provision", Outcome: "failure",
			Details: map[string]any{"error": err.Error()},
		})
		return nil, fmt.Errorf("provider registration: %w", err)
	}

	// ⑥ Persist device record
	deviceRow, err := s.db.CreateDevice(ctx, db.CreateDeviceParams{
		ID: deviceID, TenantID: req.TenantID,
		ExternalID: pDevice.ExternalID, Name: req.Name,
		Description: req.Description, Provider: tenantRow.Provider,
	})
	if err != nil {
		return nil, fmt.Errorf("persisting device: %w", err)
	}

	// ⑦ Link cert to device
	_, _ = s.db.UpdateDeviceCert(ctx, db.UpdateDeviceCertParams{
		ID: deviceID, CurrentCertID: bundle.ID, TenantID: req.TenantID,
	})

	// ⑧⑨ Events + audit
	s.events.DeviceProvisioned(ctx, deviceID.String(), req.TenantID.String(), tenantRow.Provider, req.Name)
	s.auditor.Log(ctx, audit.Entry{
		TenantID: req.TenantID, ActorID: req.ActorID,
		Action: "device.provision", Resource: "device",
		ResourceID: deviceID, Outcome: "success",
	})

	return &ProvisionResult{Device: deviceRow, CertBundle: bundle, Endpoint: pDevice.Endpoint}, nil
}
```

---

## 12.3 Zero-Downtime Certificate Rotation

```go
// RotateCertificate performs zero-downtime cert rotation.
// Order: issue new → attach new → update DB → detach old → revoke old.
// The device is never offline because both certs are valid during the transition window.
func (s *Service) RotateCertificate(ctx context.Context, deviceID, tenantID uuid.UUID, actorID string) (*certificate.Bundle, error) {
	dev, err := s.db.GetDeviceByID(ctx, deviceID, tenantID)
	if err != nil {
		return nil, fmt.Errorf("device not found: %w", err)
	}
	oldCertID := dev.CurrentCertID

	// Issue new cert
	newBundle, err := s.certs.Issue(ctx, certificate.IssueReq{
		TenantID: tenantID, DeviceID: deviceID, DeviceName: dev.Name,
	})
	if err != nil {
		return nil, err
	}

	tenantRow, _ := s.db.GetTenant(ctx, tenantID)
	provider, _  := providers.Get(providers.ProviderType(tenantRow.Provider))
	creds, _     := s.loadProviderCreds(ctx, tenantID, tenantRow.Provider)

	// Attach new cert (device now accepts BOTH certs)
	if err := provider.AttachCertificate(ctx, providers.AttachCertReq{
		Credentials: creds, ExternalID: dev.ExternalID, CertPEM: newBundle.CertificatePEM,
	}); err != nil {
		_ = s.certs.Revoke(ctx, newBundle.ID, "rotation_failed")
		return nil, fmt.Errorf("attaching new cert: %w", err)
	}

	// Update DB to point to new cert
	s.db.UpdateDeviceCert(ctx, db.UpdateDeviceCertParams{
		ID: deviceID, CurrentCertID: newBundle.ID, TenantID: tenantID,
	})

	// Detach old cert, then revoke it
	if oldCertID != uuid.Nil {
		_ = provider.DetachCertificate(ctx, providers.DetachCertReq{
			Credentials: creds, ExternalID: dev.ExternalID, CertID: oldCertID.String(),
		})
		_ = s.certs.Revoke(ctx, oldCertID, "rotation_superseded")
	}

	s.auditor.Log(ctx, audit.Entry{
		TenantID: tenantID, ActorID: actorID,
		Action: "device.cert.rotate", Resource: "certificate",
		ResourceID: newBundle.ID, Outcome: "success",
	})
	return newBundle, nil
}
```
