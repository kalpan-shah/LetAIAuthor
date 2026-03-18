# Chapter 15: Telemetry Routing & Audit Service

> **Part IV — Platform Services**

Routing rules define what happens to device messages after they reach the IoT provider. The Audit Service maintains an immutable record of every platform action.

---

## 15.1 Routing Rule Model

| Field | Type | Description |
|---|---|---|
| `topic_filter` | MQTT pattern | `tenant/{id}/device/+/telemetry` — supports `+` and `#` wildcards |
| `destination_type` | enum | `kinesis` | `eventhub` | `s3` | `postgres` | `http` | `mqtt_bridge` |
| `destination_config` | JSONB | Provider-specific: stream name, connection string, bucket name |
| `sql_filter` | SQL string | Optional payload filter: `SELECT * WHERE temperature > 80` |
| `enabled` | bool | Rules can be toggled without deletion |

---

## 15.2 Audit Service

```go
// internal/audit/service.go
package audit

import (
	"context"
	"log/slog"

	"github.com/google/uuid"
	"github.com/your-org/iot-platform/internal/db"
)

type Entry struct {
	TenantID   uuid.UUID
	ActorID    string
	ActorType  string  // 'user' | 'api_key' | 'service' | 'system'
	Action     string  // e.g. 'device.provision'
	Resource   string  // 'device' | 'tenant' | 'certificate'
	ResourceID uuid.UUID
	Outcome    string  // 'success' | 'failure'
	Details    map[string]any
	IPAddress  string
}

type Service struct { db *db.Queries }

func NewService(q *db.Queries) *Service { return &Service{db: q} }

// Log writes an immutable audit record.
// It NEVER returns an error to the caller — a failed audit write
// is logged but does not fail the business operation.
func (s *Service) Log(ctx context.Context, e Entry) {
	if e.ActorType == "" { e.ActorType = "system" }
	detailsJSON, _ := marshalDetails(e.Details)

	err := s.db.CreateAuditLog(ctx, db.CreateAuditLogParams{
		TenantID:     e.TenantID,
		ActorID:      e.ActorID,
		ActorType:    e.ActorType,
		Action:       e.Action,
		ResourceType: e.Resource,
		ResourceID:   e.ResourceID,
		Outcome:      e.Outcome,
		Details:      detailsJSON,
		IPAddress:    e.IPAddress,
	})
	if err != nil {
		slog.Error("audit write failed",
			"action", e.Action, "actor", e.ActorID, "error", err)
	}
}
```

---

## 15.3 Example Routing Configurations

```json
// Route temperature telemetry to AWS Kinesis
{
  "topic_filter":     "tenant/+/device/+/telemetry/temperature",
  "destination_type": "kinesis",
  "destination_config": {
    "stream_name":  "iot-temperature-stream",
    "region":       "us-east-1",
    "partition_key": "device_id"
  },
  "sql_filter": "SELECT * WHERE temperature > 0",
  "enabled": true
}

// Route all telemetry to Azure Event Hub
{
  "topic_filter":     "tenant/+/device/+/telemetry",
  "destination_type": "eventhub",
  "destination_config": {
    "connection_string": "Endpoint=sb://...",
    "hub_name":          "iot-telemetry"
  },
  "enabled": true
}
```
