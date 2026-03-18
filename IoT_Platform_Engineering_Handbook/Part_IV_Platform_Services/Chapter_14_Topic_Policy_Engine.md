# Chapter 14: Topic Namespace & Policy Engine

> **Part IV — Platform Services**

The Policy Engine translates the platform's abstract concept of device ownership into concrete access control rules that each IoT provider enforces at the broker level. Correctness here is a security requirement — mistakes allow cross-tenant data leakage.

---

## 14.1 Topic Namespace Specification

```
Topic structure: tenant/{tenant_id}/device/{device_id}/{channel}

Publish channels (device → broker):
  tenant/{t}/device/{d}/telemetry        main sensor data
  tenant/{t}/device/{d}/telemetry/+      sub-channels (e.g. /temperature)
  tenant/{t}/device/{d}/state            device shadow updates
  tenant/{t}/device/{d}/commands/ack     command acknowledgements
  tenant/{t}/device/{d}/events/+         alert events

Subscribe channels (broker → device):
  tenant/{t}/device/{d}/commands         operator commands
  tenant/{t}/device/{d}/commands/+       sub-commands
  tenant/{t}/device/{d}/state/get        state read requests
  tenant/{t}/device/{d}/config           configuration updates
  tenant/{t}/device/{d}/ota              firmware update notifications

What is BLOCKED for every device (enforced at broker):
  tenant/{OTHER}/...                     cross-tenant access
  tenant/{t}/device/{OTHER}/...          cross-device access within tenant
  #  (subscribe all)                     wildcard all topics
```

---

## 14.2 Policy Engine Implementation

```go
// internal/policy/engine.go
package policy

import (
	"fmt"
	"github.com/your-org/iot-platform/internal/providers"
)

type Engine struct{}

func NewEngine() *Engine { return &Engine{} }

// ForDevice generates provider-agnostic topic rules for one device.
func (e *Engine) ForDevice(tenantID, deviceID string) providers.PolicyRules {
	base := fmt.Sprintf("tenant/%s/device/%s", tenantID, deviceID)
	return providers.PolicyRules{
		TenantID: tenantID,
		DeviceID: deviceID,
		PublishTopics: []string{
			base + "/telemetry",
			base + "/telemetry/+",
			base + "/state",
			base + "/commands/ack",
			base + "/events/+",
		},
		SubscribeTopics: []string{
			base + "/commands",
			base + "/commands/+",
			base + "/state/get",
			base + "/config",
			base + "/ota",
		},
	}
}

// ── AWS Policy Translation ────────────────────────────────────────────────────

type AWSPolicyDoc struct {
	Version   string    `json:"Version"`
	Statement []AWSStmt `json:"Statement"`
}

type AWSStmt struct {
	Effect   string   `json:"Effect"`
	Action   []string `json:"Action"`
	Resource []string `json:"Resource"`
}

func (e *Engine) ToAWSPolicy(rules providers.PolicyRules, region, accountID string) *AWSPolicyDoc {
	arn := func(svc, res string) string {
		return fmt.Sprintf("arn:aws:iot:%s:%s:%s/%s", region, accountID, svc, res)
	}
	pubARNs := make([]string, len(rules.PublishTopics))
	for i, t := range rules.PublishTopics { pubARNs[i] = arn("topic", t) }

	subARNs := make([]string, 0, len(rules.SubscribeTopics)*2)
	for _, t := range rules.SubscribeTopics {
		subARNs = append(subARNs, arn("topic", t), arn("topicfilter", t))
	}
	return &AWSPolicyDoc{
		Version: "2012-10-17",
		Statement: []AWSStmt{
			{Effect: "Allow", Action: []string{"iot:Connect"},
				Resource: []string{arn("client", rules.DeviceID)}},
			{Effect: "Allow", Action: []string{"iot:Publish", "iot:Retain"},
				Resource: pubARNs},
			{Effect: "Allow", Action: []string{"iot:Subscribe", "iot:Receive"},
				Resource: subARNs},
		},
	}
}

// ── Mosquitto ACL Translation ─────────────────────────────────────────────────

func (e *Engine) ToMosquittoACL(rules providers.PolicyRules) []string {
	lines := []string{
		fmt.Sprintf("# Device %s (tenant %s)", rules.DeviceID, rules.TenantID),
		fmt.Sprintf("user %s", rules.DeviceID),
	}
	for _, t := range rules.PublishTopics {
		lines = append(lines, fmt.Sprintf("topic write %s", t))
	}
	for _, t := range rules.SubscribeTopics {
		lines = append(lines, fmt.Sprintf("topic read %s", t))
	}
	return append(lines, "")
}
```
