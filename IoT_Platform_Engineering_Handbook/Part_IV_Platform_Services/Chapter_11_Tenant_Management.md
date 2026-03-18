# Chapter 11: Tenant Management Service

> **Part IV — Platform Services**

The Tenant Service owns the lifecycle of every customer organisational unit. It enforces quota limits, validates provider readiness, and gates all device operations on tenant status.

---

## 11.1 Tenant Lifecycle States

```
  [active]  ←─────────────────── reactivate ───────────────────
     │                                                           │
     │── suspend ──► [suspended]  (provisioning blocked)        │
     │                    │── reactivate ──────────────────────►│
     │                    │── archive ──► [archived] (permanent)
     │── archive ──────────────────────► [archived]
```

---

## 11.2 Tenant Service Implementation

```go
// internal/tenant/service.go
package tenant

import (
	"context"
	"fmt"
	"regexp"
	"strings"

	"github.com/google/uuid"
	"github.com/your-org/iot-platform/internal/audit"
	"github.com/your-org/iot-platform/internal/db"
	"github.com/your-org/iot-platform/internal/events"
)

type Service struct {
	db      *db.Queries
	auditor *audit.Service
	events  *events.Publisher
}

type CreateInput struct {
	Name            string `json:"name"     validate:"required,min=2,max=255"`
	Provider        string `json:"provider" validate:"required,oneof=aws azure mosquitto"`
	Region          string `json:"region"`
	MaxDevices      int    `json:"max_devices"`
	MaxCertsPerDay  int    `json:"max_certs_per_day"`
	MaxRoutingRules int    `json:"max_routing_rules"`
}

func (s *Service) Create(ctx context.Context, in CreateInput, actorID string) (*db.Tenant, error) {
	if in.MaxDevices      == 0 { in.MaxDevices      = 1000 }
	if in.MaxCertsPerDay  == 0 { in.MaxCertsPerDay  = 500  }
	if in.MaxRoutingRules == 0 { in.MaxRoutingRules = 50   }

	t, err := s.db.CreateTenant(ctx, db.CreateTenantParams{
		Name:            in.Name,
		Slug:            slugify(in.Name),
		Provider:        in.Provider,
		Region:          in.Region,
		MaxDevices:      int32(in.MaxDevices),
		MaxCertsPerDay:  int32(in.MaxCertsPerDay),
		MaxRoutingRules: int32(in.MaxRoutingRules),
	})
	if err != nil {
		return nil, fmt.Errorf("creating tenant: %w", err)
	}

	s.events.TenantCreated(ctx, t.ID.String(), t.Provider)
	s.auditor.Log(ctx, audit.Entry{
		ActorID: actorID, Action: "tenant.create",
		Resource: "tenant", ResourceID: t.ID, Outcome: "success",
	})
	return &t, nil
}

func (s *Service) Suspend(ctx context.Context, id uuid.UUID, actorID, reason string) error {
	_, err := s.db.UpdateTenantStatus(ctx, db.UpdateTenantStatusParams{
		ID: id, Status: "suspended",
	})
	if err != nil {
		return err
	}
	s.events.TenantSuspended(ctx, id.String(), reason)
	s.auditor.Log(ctx, audit.Entry{
		TenantID: id, ActorID: actorID, Action: "tenant.suspend",
		Resource: "tenant", ResourceID: id, Outcome: "success",
		Details: map[string]any{"reason": reason},
	})
	return nil
}

// CheckDeviceQuota returns an error if the tenant is suspended or at its device limit.
func (s *Service) CheckDeviceQuota(ctx context.Context, tenantID uuid.UUID) error {
	t, err := s.db.GetTenant(ctx, tenantID)
	if err != nil {
		return err
	}
	if t.Status != "active" {
		return fmt.Errorf("tenant %s is %s", tenantID, t.Status)
	}
	count, err := s.db.CountActiveDevicesByTenant(ctx, tenantID)
	if err != nil {
		return err
	}
	if int32(count) >= t.MaxDevices {
		return fmt.Errorf("device quota exceeded: %d/%d", count, t.MaxDevices)
	}
	return nil
}

var nonAlnum = regexp.MustCompile(`[^a-z0-9]+`)

func slugify(name string) string {
	s := strings.ToLower(name)
	s = nonAlnum.ReplaceAllString(s, "-")
	s = strings.Trim(s, "-")
	if len(s) > 98 { s = s[:98] }
	return s
}
```
