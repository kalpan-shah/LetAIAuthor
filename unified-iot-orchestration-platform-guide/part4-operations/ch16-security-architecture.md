# Chapter 16: Security Architecture & Threat Modeling

> *"Security is not a feature — it's a property of the system."*

---

## 16.1 Introduction

An IoT orchestration platform is a high-value target. It holds cloud provider credentials, signs device certificates, and controls access to potentially thousands of IoT devices. A breach here doesn't just expose data — it could allow an attacker to control physical infrastructure.

This chapter covers the complete security architecture: threat modeling, defense layers, and concrete mitigations.

---

## 16.2 Threat Model (STRIDE)

| Threat | Category | Target | Severity | Mitigation |
|--------|----------|--------|----------|------------|
| Stolen API key grants full access | **Spoofing** | API layer | Critical | Short-lived JWT, RBAC, API key rotation |
| Attacker reads cloud credentials from DB | **Info Disclosure** | Database | Critical | Credentials never in DB — Vault only |
| Attacker modifies audit logs | **Tampering** | Audit service | High | HMAC signing via Vault Transit |
| Rogue tenant creates unlimited devices | **Elevation of Privilege** | Quota service | High | Server-side quota enforcement, rate limiting |
| Tenant A reads Tenant B's topics | **Info Disclosure** | MQTT topics | Critical | Namespace isolation, RLS, per-device policies |
| Provider API calls intercepted | **Info Disclosure** | Provider adapter | High | TLS for all provider calls, STS sessions |
| Denial of service via mass provisioning | **DoS** | API, providers | High | Rate limiting, circuit breakers, quotas |
| Private key extracted from database | **Info Disclosure** | Certificate store | Critical | Private keys never stored — returned once |
| Unauthorized certificate issuance | **Spoofing** | Vault PKI | Critical | Service-specific Vault policies |

---

## 16.3 Defense-in-Depth Architecture

```
┌───────────────────────────────────────────────────────────┐
│ Layer 6: Network Security                                  │
│  • Kubernetes NetworkPolicies                              │
│  • mTLS between services (Istio/Linkerd)                   │
│  • Ingress TLS termination                                 │
├───────────────────────────────────────────────────────────┤
│ Layer 5: API Security                                      │
│  • JWT authentication, RBAC authorization                  │
│  • Request rate limiting, input validation                 │
│  • API key management with scoping                         │
├───────────────────────────────────────────────────────────┤
│ Layer 4: Application Security                              │
│  • Tenant middleware (every request scoped)                 │
│  • Idempotency tokens (no replay attacks)                  │
│  • Configuration validation                                │
├───────────────────────────────────────────────────────────┤
│ Layer 3: Data Security                                     │
│  • Row-Level Security in PostgreSQL                        │
│  • Encryption at rest (database, Vault)                    │
│  • HMAC-signed audit logs                                  │
├───────────────────────────────────────────────────────────┤
│ Layer 2: Secrets Management                                │
│  • Vault for all credentials                               │
│  • Dynamic secrets with TTL                                │
│  • Kubernetes auth (no static tokens)                      │
├───────────────────────────────────────────────────────────┤
│ Layer 1: Infrastructure Security                           │
│  • Distroless container images                             │
│  • Non-root container execution                            │
│  • Read-only filesystem                                    │
│  • Regular base image updates                              │
└───────────────────────────────────────────────────────────┘
```

---

## 16.4 Authentication & Authorization

### JWT + RBAC Implementation

🐍 **Python — RBAC middleware**:

```python
# app/middleware/auth.py
from fastapi import Request, HTTPException
from jose import jwt, JWTError
from enum import Enum


class Role(str, Enum):
    ADMIN = "admin"                # Full platform access
    TENANT_ADMIN = "tenant_admin"  # Full access within tenant
    OPERATOR = "operator"          # Read + provision, no delete
    VIEWER = "viewer"              # Read-only access
    DEVICE = "device"              # Device-level (claim tokens)


# Permission matrix
PERMISSIONS = {
    Role.ADMIN: {"*"},
    Role.TENANT_ADMIN: {
        "tenant.read", "tenant.update",
        "device.*", "certificate.*",
        "policy.*", "routing.*", "audit.read",
    },
    Role.OPERATOR: {
        "tenant.read",
        "device.read", "device.create", "device.provision",
        "certificate.read", "certificate.issue",
        "routing.read",
        "audit.read",
    },
    Role.VIEWER: {
        "tenant.read", "device.read", "certificate.read",
        "routing.read", "audit.read",
    },
}


class AuthMiddleware:
    def __init__(self, jwt_secret: str, jwt_algorithm: str = "HS256"):
        self.secret = jwt_secret
        self.algorithm = jwt_algorithm

    def verify_token(self, request: Request) -> dict:
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if not token:
            raise HTTPException(status_code=401, detail="Missing auth token")

        try:
            payload = jwt.decode(token, self.secret, algorithms=[self.algorithm])
        except JWTError:
            raise HTTPException(status_code=401, detail="Invalid token")

        return payload

    def require_permission(self, permission: str):
        """Decorator/dependency to check permissions."""
        def checker(request: Request):
            payload = self.verify_token(request)
            role = Role(payload.get("role", "viewer"))
            allowed = PERMISSIONS.get(role, set())

            if "*" not in allowed and permission not in allowed:
                # Check wildcard pattern (e.g., "device.*" matches "device.create")
                category = permission.split(".")[0] + ".*"
                if category not in allowed:
                    raise HTTPException(status_code=403, detail="Insufficient permissions")

            return payload
        return checker
```

---

## 16.5 Container Security

```dockerfile
# Dockerfile (security-hardened)
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /server ./cmd/server

# Distroless — no shell, no package manager, minimal attack surface
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /server /server

# Run as non-root user (UID 65534 = nobody)
USER nonroot:nonroot

# Read-only filesystem
VOLUME ["/tmp"]

EXPOSE 8080 9090
ENTRYPOINT ["/server"]
```

```yaml
# Kubernetes security context
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      containers:
        - name: iot-platform
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            limits:
              memory: "256Mi"
              cpu: "500m"
```

---

## 16.6 Network Policies

```yaml
# Kubernetes NetworkPolicy — isolate platform services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: iot-platform-policy
  namespace: iot-platform
spec:
  podSelector:
    matchLabels:
      app: iot-platform
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ingress-gateway
      ports:
        - port: 8080    # REST
        - port: 9090    # gRPC
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    - to:
        - podSelector:
            matchLabels:
              app: vault
      ports:
        - port: 8200
    - to:                # Allow outbound to cloud providers (AWS, Azure)
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
```

---

## 16.7 Security Checklist

| Category | Check | Status |
|----------|-------|--------|
| **Secrets** | No secrets in env vars or config files | ☐ |
| **Secrets** | All cloud credentials in Vault | ☐ |
| **Secrets** | Vault policies follow least privilege | ☐ |
| **Auth** | JWT tokens have short TTL (< 1 hour) | ☐ |
| **Auth** | RBAC enforced on all endpoints | ☐ |
| **Data** | RLS enabled on all tenant-scoped tables | ☐ |
| **Data** | Private keys never stored in DB | ☐ |
| **Data** | Audit logs HMAC-signed | ☐ |
| **Network** | TLS on all internal communication | ☐ |
| **Network** | NetworkPolicies applied | ☐ |
| **Container** | Non-root, read-only filesystem | ☐ |
| **Container** | Distroless or minimal base image | ☐ |
| **Dependencies** | Regular vulnerability scanning | ☐ |
| **API** | Input validation on all endpoints | ☐ |
| **API** | Rate limiting per tenant | ☐ |

---

## 16.8 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **STRIDE threat model** | Systematic analysis of attack surfaces |
| **Defense-in-depth** | 6 layers from infrastructure to network |
| **RBAC** | Role-based permissions checked per request |
| **Container hardening** | Distroless, non-root, read-only FS |
| **Network isolation** | K8s NetworkPolicies for east-west traffic |

### What's Next

**Chapter 17** covers **DevOps & SecOps Workflows** — CI/CD pipelines, security scanning, deployment strategies, and operational boundaries.
