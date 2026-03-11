# Chapter 10: Policy & Topic Namespace Engine

> *"The namespace is the foundation of tenant isolation in MQTT. Get it wrong, and Tenant A reads Tenant B's data."*

---

## 10.1 Introduction

The Policy & Topic Namespace Engine serves two critical functions:
1. **Define standardized topic structures** that enforce tenant and device isolation
2. **Generate provider-specific access control policies** that restrict each device to its own namespace

This chapter covers the complete implementation — namespace design, policy translation, and provider-specific deployment.

---

## 10.2 Topic Namespace Design

### Standard Namespace Hierarchy

```
tenant/{tenant_id}/
├── device/{device_id}/
│   ├── telemetry           # Device publishes sensor data
│   ├── state               # Device reports current state
│   ├── commands            # Platform sends commands TO device
│   ├── config              # Platform sends configuration updates
│   └── ota                 # Firmware/OTA update channel
├── fleet/
│   ├── broadcast           # Tenant-wide messages to all devices
│   └── alerts              # Tenant-wide alerting channel
└── admin/
    └── health              # Device health heartbeats
```

### Isolation Guarantee

| Rule | Enforcement |
|------|------------|
| Device A cannot read Device B's data | Policy limits publish/subscribe to own `device/{id}` subtree |
| Tenant X cannot access Tenant Y's topics | Policy constrains to `tenant/{tenant_id}` prefix |
| Devices cannot publish to command channels | Publish restricted to telemetry/state topics |
| Platform can read all topics for a tenant | Admin policy uses wildcard within tenant scope |

---

## 10.3 Complete Namespace Service

🐍 **Python**:

```python
# app/services/namespace_service.py
from dataclasses import dataclass, field


@dataclass
class TopicNamespace:
    """Defines the complete topic namespace for a device."""
    tenant_id: str
    device_id: str
    device_name: str

    @property
    def base(self) -> str:
        return f"tenant/{self.tenant_id}/device/{self.device_id}"

    @property
    def telemetry(self) -> str:
        return f"{self.base}/telemetry"

    @property
    def state(self) -> str:
        return f"{self.base}/state"

    @property
    def commands(self) -> str:
        return f"{self.base}/commands"

    @property
    def config(self) -> str:
        return f"{self.base}/config"

    @property
    def ota(self) -> str:
        return f"{self.base}/ota"

    @property
    def publish_topics(self) -> list[str]:
        """Topics the DEVICE can publish to."""
        return [self.telemetry, self.state]

    @property
    def subscribe_topics(self) -> list[str]:
        """Topics the DEVICE can subscribe to."""
        return [self.commands, self.config, self.ota]

    @property
    def all_topics(self) -> list[str]:
        return self.publish_topics + self.subscribe_topics
```

🔵 **Go**:

```go
// internal/namespace/namespace.go
package namespace

import "fmt"

type TopicNamespace struct {
    TenantID   string
    DeviceID   string
    DeviceName string
}

func (ns *TopicNamespace) Base() string {
    return fmt.Sprintf("tenant/%s/device/%s", ns.TenantID, ns.DeviceID)
}

func (ns *TopicNamespace) Telemetry() string { return ns.Base() + "/telemetry" }
func (ns *TopicNamespace) State() string     { return ns.Base() + "/state" }
func (ns *TopicNamespace) Commands() string  { return ns.Base() + "/commands" }
func (ns *TopicNamespace) Config() string    { return ns.Base() + "/config" }
func (ns *TopicNamespace) OTA() string       { return ns.Base() + "/ota" }

func (ns *TopicNamespace) PublishTopics() []string {
    return []string{ns.Telemetry(), ns.State()}
}

func (ns *TopicNamespace) SubscribeTopics() []string {
    return []string{ns.Commands(), ns.Config(), ns.OTA()}
}
```

---

## 10.4 Policy Translation Engine

The policy engine translates a provider-agnostic namespace into provider-specific policy formats:

```python
# app/services/policy_engine.py
import json


class PolicyEngine:
    """Translates topic namespaces into provider-specific policies."""

    def generate(self, namespace: TopicNamespace, provider: str) -> dict:
        """Generate a provider-specific policy document."""
        generators = {
            "aws":       self._aws_iot_policy,
            "azure":     self._azure_iot_policy,
            "mosquitto": self._mosquitto_acl,
        }
        gen = generators.get(provider)
        if not gen:
            raise ValueError(f"Unsupported provider: {provider}")
        return gen(namespace)

    def _aws_iot_policy(self, ns: TopicNamespace) -> dict:
        """AWS IoT Core policy document (IAM-style JSON)."""
        return {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": "iot:Connect",
                    "Resource": f"arn:aws:iot:*:*:client/{ns.device_name}",
                },
                {
                    "Effect": "Allow",
                    "Action": "iot:Publish",
                    "Resource": [
                        f"arn:aws:iot:*:*:topic/{t}" for t in ns.publish_topics
                    ],
                },
                {
                    "Effect": "Allow",
                    "Action": "iot:Subscribe",
                    "Resource": [
                        f"arn:aws:iot:*:*:topicfilter/{t}" for t in ns.subscribe_topics
                    ],
                },
                {
                    "Effect": "Allow",
                    "Action": "iot:Receive",
                    "Resource": [
                        f"arn:aws:iot:*:*:topic/{t}" for t in ns.subscribe_topics
                    ],
                },
            ],
        }

    def _azure_iot_policy(self, ns: TopicNamespace) -> dict:
        """Azure IoT Hub permissions."""
        return {
            "device_id": ns.device_name,
            "permissions": {
                "device_to_cloud": True,
                "cloud_to_device": True,
                "twin_read": True,
                "twin_write": False,
            },
            "custom_topic_access": {
                "publish": ns.publish_topics,
                "subscribe": ns.subscribe_topics,
            },
        }

    def _mosquitto_acl(self, ns: TopicNamespace) -> dict:
        """Mosquitto ACL file entries."""
        lines = [f"user {ns.device_name}"]
        for topic in ns.publish_topics:
            lines.append(f"topic write {topic}")
        for topic in ns.subscribe_topics:
            lines.append(f"topic read {topic}")

        return {
            "format": "mosquitto_acl",
            "user": ns.device_name,
            "content": "\n".join(lines),
        }
```

> ☕ *Legacy note*: Java teams using Eclipse Hono or HiveMQ will recognize topic-level ACLs. The pattern is the same — define allowed topics per device identity. The difference is abstracting the policy generation so it works across providers.

---

## 10.5 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Namespace hierarchy** | `tenant/{id}/device/{id}/{channel}` enforces isolation |
| **Publish vs. Subscribe** | Devices can only publish telemetry, only subscribe to commands |
| **Cross-provider policies** | Single namespace → AWS JSON, Azure permissions, Mosquitto ACL |

### What's Next

**Chapter 11** builds **Provisioning Workflows** — orchestrated multi-step device onboarding strategies.
