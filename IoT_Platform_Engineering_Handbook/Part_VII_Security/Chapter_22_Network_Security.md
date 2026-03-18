# Chapter 22: Network Security & Zero-Trust Architecture

> **Part VII — Security (SecOps)**

The platform adopts a zero-trust posture: no service trusts another service by default. Every inter-service call is authenticated. Network policies enforce that only explicitly allowed communication paths exist.

---

## 22.1 Kubernetes NetworkPolicy: Default-Deny

```yaml
# deploy/k8s/base/network-policy.yaml

# Step 1: Deny ALL traffic by default in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: iot-platform
spec:
  podSelector: {}           # Applies to all pods
  policyTypes: [Ingress, Egress]

---
# Step 2: Allow API pods to receive traffic from ingress controller only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: iot-platform
spec:
  podSelector:
    matchLabels: { app: iot-api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - { protocol: TCP, port: 8080 }

---
# Step 3: Allow API pods to reach infrastructure components
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress-infra
  namespace: iot-platform
spec:
  podSelector:
    matchLabels: { app: iot-api }
  policyTypes: [Egress]
  egress:
    - to: [{ podSelector: { matchLabels: { app: postgresql } } }]
      ports: [{ protocol: TCP, port: 5432 }]
    - to: [{ podSelector: { matchLabels: { app: redis } } }]
      ports: [{ protocol: TCP, port: 6379 }]
    - to: [{ podSelector: { matchLabels: { app: vault } } }]
      ports: [{ protocol: TCP, port: 8200 }]
    - to: [{ podSelector: { matchLabels: { app: nats } } }]
      ports: [{ protocol: TCP, port: 4222 }]
    # External APIs: AWS STS, Azure ARM — TCP 443 only
    - to: [{}]
      ports: [{ protocol: TCP, port: 443 }]
    # DNS resolution
    - to: [{}]
      ports: [{ protocol: UDP, port: 53 }]

---
# Step 4: Allow Prometheus to scrape all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: iot-platform
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
        - podSelector:
            matchLabels: { app: prometheus }
      ports:
        - { protocol: TCP, port: 8080 }
```

---

## 22.2 TLS Configuration Requirements

| Connection | Minimum TLS | Certificate Type | Notes |
|---|---|---|---|
| Browser → API Gateway | TLS 1.3 | ECDSA P-256 (Let's Encrypt) | HSTS max-age=63072000, preload |
| API Gateway → Platform API | TLS 1.2+ | Vault PKI (rotated 90d) | Internal CA; no public trust required |
| Device → MQTT Broker (AWS/Azure) | TLS 1.2+ | Provider CA bundle | Device cert CN validated as device ID |
| Device → Mosquitto (:8883) | TLS 1.2+ | Mutual TLS | Device presents X.509 cert; CN = device ID |
| Platform → Vault | TLS 1.2+ | Vault self-signed | Pinned in Vault client config |
| Platform → PostgreSQL | TLS 1.2+ | `sslmode=require` minimum | Use `sslmode=verify-full` + CA cert for highest security |
| Platform → Redis | TLS 1.2+ | `tls.Config` in Go client | Required in production |

---

## 22.3 MQTT Security Configuration

```conf
# Mosquitto production TLS configuration
# deploy/docker/mosquitto/mosquitto.conf

# DISABLE plaintext listener in production
# listener 1883   ← REMOVED

# TLS-only listener
listener 8883
cafile   /mosquitto/config/certs/ca.crt
certfile /mosquitto/config/certs/server.crt
keyfile  /mosquitto/config/certs/server.key

# Require client certificate (mutual TLS)
require_certificate true

# Use the certificate CN as the MQTT username.
# This ties the device identity to the certificate — no separate password needed.
use_identity_as_username true

# ACL file enforces per-device topic restrictions
acl_file /mosquitto/config/acl

# Disable anonymous connections
allow_anonymous false

# WebSocket TLS (for browser-based device simulations)
listener 9001
protocol websockets
cafile   /mosquitto/config/certs/ca.crt
certfile /mosquitto/config/certs/server.crt
keyfile  /mosquitto/config/certs/server.key
require_certificate true
```

---

## 22.4 Pod Security Standards

```yaml
# deploy/k8s/base/pod-security-policy.yaml
# Applied as a namespace label (Kubernetes 1.25+)
apiVersion: v1
kind: Namespace
metadata:
  name: iot-platform
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn:    restricted
    pod-security.kubernetes.io/audit:   restricted
```

Every container spec must include:
```yaml
securityContext:
  runAsNonRoot:             true
  runAsUser:                65534   # nobody
  runAsGroup:               65534
  readOnlyRootFilesystem:   true
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop: [ALL]
```

---

## 22.5 Secrets Never in Environment Variables (Production)

```go
// cmd/api/main.go — production secret loading from files (Vault Agent Injector)
func loadConfig() *Config {
    // In development: read from environment variables
    if os.Getenv("ENV") == "development" {
        return &Config{
            DatabaseURL: os.Getenv("DATABASE_URL"),
            // ...
        }
    }

    // In production: read from files mounted by Vault Agent Injector
    // File content: DATABASE_URL=postgres://...
    dbURL  := readSecretFile("/vault/secrets/db",    "DATABASE_URL")
    natsURL := readSecretFile("/vault/secrets/nats", "NATS_URL")
    return &Config{
        DatabaseURL: dbURL,
        NatsURL:     natsURL,
    }
}

func readSecretFile(filePath, key string) string {
    data, err := os.ReadFile(filePath)
    if err != nil {
        log.Fatalf("cannot read secret file %s: %v", filePath, err)
    }
    // Parse KEY=VALUE format
    for _, line := range strings.Split(string(data), "\n") {
        if strings.HasPrefix(line, key+"=") {
            return strings.TrimPrefix(line, key+"=")
        }
    }
    log.Fatalf("key %s not found in %s", key, filePath)
    return ""
}
```
