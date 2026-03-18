# Chapter 6: Event Bus with NATS JetStream

> **Part II — Core Infrastructure**

Platform services communicate through a durable event bus. When a device is provisioned, the Device Service publishes a `DeviceProvisioned` event. The Audit Service consumes it and writes a log record. The Quota Service consumes it and updates the device count. This decoupling allows services to be developed, tested, and deployed independently.

---

## 6.1 Event Schema

```go
// internal/events/events.go
package events

import "time"

type BaseEvent struct {
	EventID   string    `json:"event_id"`
	Timestamp time.Time `json:"timestamp"`
	Source    string    `json:"source"`
	Version   string    `json:"version"`
}

type DeviceProvisioned struct {
	BaseEvent
	DeviceID   string `json:"device_id"`
	TenantID   string `json:"tenant_id"`
	Provider   string `json:"provider"`
	DeviceName string `json:"device_name"`
}

type CertificateIssued struct {
	BaseEvent
	CertificateID string    `json:"certificate_id"`
	DeviceID      string    `json:"device_id"`
	TenantID      string    `json:"tenant_id"`
	SerialNumber  string    `json:"serial_number"`
	ExpiresAt     time.Time `json:"expires_at"`
}

type CertificateExpiringSoon struct {
	BaseEvent
	CertificateID string    `json:"certificate_id"`
	DeviceID      string    `json:"device_id"`
	TenantID      string    `json:"tenant_id"`
	ExpiresAt     time.Time `json:"expires_at"`
	DaysRemaining int       `json:"days_remaining"`
}

type TenantSuspended struct {
	BaseEvent
	TenantID string `json:"tenant_id"`
	Reason   string `json:"reason"`
}

// NATS subject naming: platform.{resource}.{action}
const (
	SubjectDeviceProvisioned       = "platform.device.provisioned"
	SubjectDeviceSuspended         = "platform.device.suspended"
	SubjectDeviceDeleted           = "platform.device.deleted"
	SubjectCertificateIssued       = "platform.certificate.issued"
	SubjectCertificateRevoked      = "platform.certificate.revoked"
	SubjectCertificateExpiringSoon = "platform.certificate.expiring-soon"
	SubjectTenantCreated           = "platform.tenant.created"
	SubjectTenantSuspended         = "platform.tenant.suspended"
)
```

---

## 6.2 NATS Publisher

```go
// internal/events/publisher.go
package events

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"time"

	"github.com/google/uuid"
	"github.com/nats-io/nats.go"
	"github.com/nats-io/nats.go/jetstream"
)

type Publisher struct {
	js     jetstream.JetStream
	source string
}

func NewPublisher(nc *nats.Conn, source string) (*Publisher, error) {
	js, err := jetstream.New(nc)
	if err != nil {
		return nil, fmt.Errorf("jetstream init: %w", err)
	}

	_, err = js.CreateOrUpdateStream(context.Background(), jetstream.StreamConfig{
		Name:      "IOT_PLATFORM_EVENTS",
		Subjects:  []string{"platform.>"},
		Retention: jetstream.LimitsPolicy,
		MaxAge:    30 * 24 * time.Hour,  // 30-day retention
		Storage:   jetstream.FileStorage, // Persistent
		Replicas:  1,                     // Set to 3 in production cluster
	})
	if err != nil {
		return nil, fmt.Errorf("creating stream: %w", err)
	}
	return &Publisher{js: js, source: source}, nil
}

func (p *Publisher) Publish(ctx context.Context, subject string, event interface{}) error {
	data, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshaling event: %w", err)
	}
	_, err = p.js.Publish(ctx, subject, data,
		jetstream.WithMsgID(uuid.New().String()), // Deduplication key
	)
	if err != nil {
		// Non-fatal: log but do not fail the calling operation
		slog.Error("event publish failed", "subject", subject, "error", err)
	}
	return nil
}

func (p *Publisher) DeviceProvisioned(ctx context.Context, deviceID, tenantID, provider, name string) {
	_ = p.Publish(ctx, SubjectDeviceProvisioned, DeviceProvisioned{
		BaseEvent: BaseEvent{
			EventID:   uuid.New().String(),
			Timestamp: time.Now().UTC(),
			Source:    p.source,
			Version:   "1",
		},
		DeviceID:   deviceID,
		TenantID:   tenantID,
		Provider:   provider,
		DeviceName: name,
	})
}
```

---

## 6.3 Stream Configuration Best Practices

| Setting | Development | Production |
|---|---|---|
| `Replicas` | 1 | 3 (requires 3-node NATS cluster) |
| `MaxAge` | 7 days | 30 days |
| `Storage` | File | File |
| `Retention` | LimitsPolicy | LimitsPolicy |
| `MaxMsgs` | Unlimited | 10M |
| Authentication | None | TLS + credentials |
