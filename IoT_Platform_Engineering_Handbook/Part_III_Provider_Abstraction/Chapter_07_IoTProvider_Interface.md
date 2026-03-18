# Chapter 7: Designing the IoTProvider Interface

> **Part III — The Provider Abstraction Layer**

The provider abstraction is the most architecturally important decision in the codebase. Get this interface right and the platform can support any IoT backend forever. Get it wrong and AWS-specific logic leaks into business services.

---

## 7.1 The Interface

```go
// internal/providers/interface.go
package providers

import (
	"context"
	"time"
)

// IoTProvider is the complete contract for an IoT infrastructure backend.
// Every provider adapter (AWS, Azure, Mosquitto) must implement this interface.
// Platform services interact ONLY with this interface — never with provider SDKs directly.
type IoTProvider interface {
	ProviderType() ProviderType
	Name()         string

	ValidateCredentials(ctx context.Context, creds Credentials) error
	GetEndpoint(ctx context.Context, creds Credentials) (string, error)

	// Device Lifecycle
	CreateDevice(ctx context.Context, req CreateDeviceReq) (*Device, error)
	GetDevice(ctx context.Context, creds Credentials, externalID string) (*Device, error)
	DeleteDevice(ctx context.Context, creds Credentials, externalID string) error
	SuspendDevice(ctx context.Context, creds Credentials, externalID string) error
	ReactivateDevice(ctx context.Context, creds Credentials, externalID string) error

	// Certificate Operations
	AttachCertificate(ctx context.Context, req AttachCertReq) error
	DetachCertificate(ctx context.Context, req DetachCertReq) error

	// Policy Operations
	CreateOrUpdatePolicy(ctx context.Context, req PolicyReq) error
	DeletePolicy(ctx context.Context, creds Credentials, policyID string) error

	// Routing Configuration
	CreateRoutingRule(ctx context.Context, req RoutingRuleReq) error
	UpdateRoutingRule(ctx context.Context, req RoutingRuleReq) error
	DeleteRoutingRule(ctx context.Context, creds Credentials, ruleID string) error
}

type ProviderType string

const (
	ProviderMosquitto ProviderType = "mosquitto"
	ProviderAWS       ProviderType = "aws"
	ProviderAzure     ProviderType = "azure"
)

// Credentials holds all possible authentication fields.
// Only fields relevant to the active provider are populated.
// All values come from Vault — never hardcoded.
type Credentials struct {
	Type ProviderType
	// AWS
	AWSRoleARN    string
	AWSExternalID string
	AWSRegion     string
	// Azure
	AzureClientID     string
	AzureClientSecret string
	AzureTenantID     string
	AzureHubHostname  string
	// Mosquitto
	MosquittoAdminURL string
	MosquittoUser     string
	MosquittoPasswd   string
}

type PolicyRules struct {
	TenantID        string
	DeviceID        string
	PublishTopics   []string
	SubscribeTopics []string
}

type CreateDeviceReq struct {
	Credentials Credentials
	TenantID    string
	DeviceID    string
	DeviceName  string
	CertPEM     string
	PolicyRules PolicyRules
}

type Device struct {
	ExternalID string
	ARN        string    // AWS specific
	HubName    string    // Azure specific
	Endpoint   string
	CreatedAt  time.Time
}
```

---

## 7.2 Provider Registry

```go
// internal/providers/registry.go
package providers

import (
	"fmt"
	"sync"
)

type Registry struct {
	mu   sync.RWMutex
	map_ map[ProviderType]IoTProvider
}

var global = &Registry{map_: make(map[ProviderType]IoTProvider)}

func Register(p IoTProvider) {
	global.mu.Lock()
	defer global.mu.Unlock()
	global.map_[p.ProviderType()] = p
}

func Get(pt ProviderType) (IoTProvider, error) {
	global.mu.RLock()
	defer global.mu.RUnlock()
	p, ok := global.map_[pt]
	if !ok {
		return nil, fmt.Errorf("provider %q not registered", pt)
	}
	return p, nil
}

// In cmd/api/main.go, register all enabled providers at startup:
//
// providers.Register(mosquitto.New(cfg.Mosquitto))
// if cfg.AWS.Enabled   { providers.Register(aws.New(cfg.AWS))   }
// if cfg.Azure.Enabled { providers.Register(azure.New(cfg.Azure)) }
```

---

## 7.3 Mock Provider (for Testing)

```go
// internal/providers/mock/provider.go
package mock

import (
	"context"
	"time"
	"github.com/your-org/iot-platform/internal/providers"
)

type Provider struct {
	CreateDeviceFn  func(ctx context.Context, req providers.CreateDeviceReq) (*providers.Device, error)
	SuspendDeviceFn func(ctx context.Context, creds providers.Credentials, id string) error
	// Add more fields as needed
}

func (m *Provider) ProviderType() providers.ProviderType { return "mock" }
func (m *Provider) Name() string                         { return "Mock Provider" }

func (m *Provider) CreateDevice(ctx context.Context, req providers.CreateDeviceReq) (*providers.Device, error) {
	if m.CreateDeviceFn != nil {
		return m.CreateDeviceFn(ctx, req)
	}
	return &providers.Device{ExternalID: req.DeviceID, CreatedAt: time.Now()}, nil
}

func (m *Provider) SuspendDevice(ctx context.Context, creds providers.Credentials, id string) error {
	if m.SuspendDeviceFn != nil {
		return m.SuspendDeviceFn(ctx, creds, id)
	}
	return nil
}
// ... implement remaining interface methods returning nil/defaults
```
