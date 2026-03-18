# Chapter 8: Eclipse Mosquitto Provider (Local & Agnostic)

> **Part III — The Provider Abstraction Layer**

The Mosquitto adapter is the fully local provider. It works with zero cloud accounts on any machine with Docker. For development, testing, air-gapped deployments, and teams that have not yet chosen a cloud provider, this is the primary provider.

---

## 8.1 Mosquitto Dynamic Security Plugin

Mosquitto 2.x includes the **Dynamic Security plugin** — a built-in access control system with a REST-like admin interface. The platform uses this to create per-device clients and per-device ACL roles without editing static configuration files.

```conf
# Add to mosquitto.conf to enable Dynamic Security
plugin /usr/lib/mosquitto/mosquitto_dynamic_security.so
plugin_opt_config_file /mosquitto/config/dynsec.json
```

---

## 8.2 Mosquitto Provider Implementation

```go
// internal/providers/mosquitto/provider.go
package mosquitto

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"time"

	"github.com/your-org/iot-platform/internal/providers"
)

type Config struct {
	AdminURL string // e.g. http://mosquitto:8888
	User     string
	Password string
}

type Provider struct {
	cfg    Config
	client *http.Client
}

func New(cfg Config) *Provider {
	return &Provider{cfg: cfg, client: &http.Client{Timeout: 15 * time.Second}}
}

func (p *Provider) ProviderType() providers.ProviderType { return providers.ProviderMosquitto }
func (p *Provider) Name() string                         { return "Eclipse Mosquitto" }

// CreateDevice: create client → create role with ACLs → assign role to client
func (p *Provider) CreateDevice(ctx context.Context, req providers.CreateDeviceReq) (*providers.Device, error) {
	// ① Create Mosquitto client
	if err := p.dynSecCommand(ctx, map[string]interface{}{
		"commands": []map[string]interface{}{{
			"command":  "createClient",
			"clientid": req.DeviceID,
			"username": req.DeviceID,
			"textname": req.DeviceName,
			// When using certificate auth, Mosquitto uses CN as username.
			// No password needed — the certificate IS the credential.
		}},
	}); err != nil {
		return nil, fmt.Errorf("creating dynsec client: %w", err)
	}

	// ② Create ACL role with topic permissions
	roleName := "role-" + req.DeviceID
	acls := buildACLs(req.PolicyRules)
	if err := p.dynSecCommand(ctx, map[string]interface{}{
		"commands": []map[string]interface{}{{
			"command":  "createRole",
			"rolename": roleName,
			"acls":     acls,
		}},
	}); err != nil {
		return nil, fmt.Errorf("creating dynsec role: %w", err)
	}

	// ③ Assign role to client
	if err := p.dynSecCommand(ctx, map[string]interface{}{
		"commands": []map[string]interface{}{{
			"command":  "addClientRole",
			"clientid": req.DeviceID,
			"rolename": roleName,
			"priority": 0,
		}},
	}); err != nil {
		return nil, fmt.Errorf("assigning role: %w", err)
	}

	return &providers.Device{ExternalID: req.DeviceID, CreatedAt: time.Now()}, nil
}

func (p *Provider) SuspendDevice(ctx context.Context, creds providers.Credentials, externalID string) error {
	return p.dynSecCommand(ctx, map[string]interface{}{
		"commands": []map[string]interface{}{{
			"command":  "disableClient",
			"clientid": externalID,
		}},
	})
}

func (p *Provider) DeleteDevice(ctx context.Context, creds providers.Credentials, externalID string) error {
	roleName := "role-" + externalID
	return p.dynSecCommand(ctx, map[string]interface{}{
		"commands": []map[string]interface{}{
			{"command": "removeClientRole", "clientid": externalID, "rolename": roleName},
			{"command": "deleteRole", "rolename": roleName},
			{"command": "deleteClient", "clientid": externalID},
		},
	})
}

func buildACLs(rules providers.PolicyRules) []map[string]interface{} {
	acls := make([]map[string]interface{}, 0)
	for _, t := range rules.PublishTopics {
		acls = append(acls, map[string]interface{}{
			"acltype": "publishClientSend", "topic": t, "allow": true,
		})
	}
	for _, t := range rules.SubscribeTopics {
		acls = append(acls, map[string]interface{}{
			"acltype": "subscribeLiteral", "topic": t, "allow": true,
		})
	}
	return acls
}

func (p *Provider) dynSecCommand(ctx context.Context, payload interface{}) error {
	body, _ := json.Marshal(payload)
	req, _ := http.NewRequestWithContext(ctx, http.MethodPost,
		p.cfg.AdminURL+"/api/v1/dynamic-security", bytes.NewReader(body))
	req.SetBasicAuth(p.cfg.User, p.cfg.Password)
	req.Header.Set("Content-Type", "application/json")
	resp, err := p.client.Do(req)
	if err != nil {
		return fmt.Errorf("dynsec HTTP: %w", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 400 {
		return fmt.Errorf("dynsec returned HTTP %d", resp.StatusCode)
	}
	return nil
}
```
