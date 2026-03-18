# Chapter 20: Threat Modelling & Security Architecture

> **Part VII — Security (SecOps)**

Security for an IoT platform cannot be retrofitted after the fact. This chapter applies the STRIDE threat model to systematically identify attack vectors, then maps each threat to a concrete mitigation that exists in the platform implementation.

---

## 20.1 STRIDE Threat Analysis

| Threat | Attack Vector | Mitigation in Platform |
|---|---|---|
| **Spoofing** | Rogue device claims identity of a legitimate device | Mutual TLS (mTLS): every device presents an X.509 cert. CN is scoped to `{device_id}.{tenant_id}.devices.iotplatform.local`. No password-only auth allowed at the broker. |
| **Spoofing** | API caller impersonates another user | Short-lived JWT (24h access token, 7d refresh). Algorithm constrained to HS256+. `aud` and `iss` claims validated on every request. Token is invalidated on logout via Redis blocklist. |
| **Tampering** | Device publishes to another tenant's topic | Policy engine generates per-device ACLs scoped to `tenant/{tenant_id}/device/{device_id}/#`. Enforced at broker level — no app-level check to bypass. |
| **Tampering** | Attacker modifies an audit log record | Audit table: application DB user has no `UPDATE` or `DELETE` permissions granted. Append-only. Old quarterly partitions archived to immutable object storage (S3 with Object Lock). |
| **Repudiation** | Operator denies performing a destructive action | Every API call writes an immutable audit record: actor ID, actor IP, HTTP method, resource, outcome, and timestamp. Records cannot be deleted by the platform itself. |
| **Information Disclosure** | Private key exposed in application logs | Private key is never written to any log statement. Stored only in Vault KV. Delivered once in the provisioning response. Log masking middleware redacts all fields matching sensitive name patterns. |
| **Information Disclosure** | Cross-tenant device data access | Tenant ID is a required, validated parameter on every device query. TenantScope middleware validates actor membership before any handler executes. All DB queries include `WHERE tenant_id = $n`. |
| **Denial of Service** | Provisioning flood exhausts DB connections or Vault rate limits | Per-tenant device quota enforced at the service layer. API rate limiting: 1,000 req/min per IP. Provisioning endpoint: 10 req/min per tenant. Circuit breaker on Vault and provider calls. |
| **Elevation of Privilege** | Tenant operator accesses platform admin endpoints | RBAC: every route declares the required permission. Middleware checks actor roles against the requirement before the handler runs. Admin routes require `platform:admin` — not grantable by tenant owners. |
| **Elevation of Privilege** | Compromised platform service account assumes all tenant roles | IAM role assumption requires `ExternalID` (= `tenant_id`). The platform cannot assume a tenant's role without knowing the correct external ID, which is never guessable. |

---

## 20.2 Trust Boundaries

```
╔══════════════════════════════════════════════════════════════════╗
║  INTERNET (untrusted)                                           ║
║  HTTPS only · TLS 1.2+ · Validated JWT on every request        ║
╚══════════════════════╤═══════════════════════════════════════════╝
                       │
╔══════════════════════▼═══════════════════════════════════════════╗
║  API GATEWAY — Kong / Traefik                                   ║
║  TLS termination · JWT signature validation · Rate limiting     ║
║  IP allow-list (optional) · DDoS mitigation                     ║
╚══════════════════════╤═══════════════════════════════════════════╝
                       │
╔══════════════════════▼═══════════════════════════════════════════╗
║  PLATFORM SERVICES  (k8s namespace: iot-platform)               ║
║  mTLS between pods · NetworkPolicy: default-deny                ║
║  Non-root UID 65534 · Read-only filesystem · No shell           ║
╚══════════════════════╤═══════════════════════════════════════════╝
                       │
╔══════════════════════▼═══════════════════════════════════════════╗
║  INFRASTRUCTURE                                                 ║
║  PostgreSQL: TLS required, app user: no UPDATE/DELETE audit     ║
║  Redis: AUTH + TLS, keyspace notifications OFF                  ║
║  Vault: Kubernetes auth, minimum-privilege policies             ║
║  NATS: TLS + NKey credentials, topic-level ACL                  ║
╚══════════════════════╤═══════════════════════════════════════════╝
                       │
╔══════════════════════▼═══════════════════════════════════════════╗
║  CLOUD PROVIDER APIs (external)                                 ║
║  AWS:   STS role assumption, ExternalID, 15-min cred TTL        ║
║  Azure: Service Principal token via Vault, 1h TTL               ║
║  Mosquitto: mTLS (device cert CN = device ID)                   ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 20.3 Attack Surface Inventory

| Surface | Exposed To | Controls |
|---|---|---|
| API Gateway (HTTPS :443) | Internet | TLS 1.2+, JWT validation, rate limiting |
| MQTT TLS port (:8883) | IoT devices | Mutual TLS, per-device ACL, no wildcard subscriptions |
| Vault API (:8200) | Platform services only | Not exposed to internet; Kubernetes network policy |
| PostgreSQL (:5432) | Platform services only | TLS, strong auth, restricted permissions |
| Prometheus (:9090) | Internal only | Not exposed externally; auth optional in production |
| Grafana (:3000) | Ops team | SSO / basic auth; not public |

---

## 20.4 Dependency Security Scanning

All dependencies are scanned in CI:

```yaml
# .github/workflows/ci.yml (security job)
- name: Trivy filesystem scan
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: fs
    scan-ref:  '.'
    severity:  CRITICAL,HIGH
    exit-code: '1'           # Fail build on critical/high CVEs

- name: Nancy Go dependency scan
  run: go list -json -m all | nancy sleuth

- name: Grype container image scan
  uses: anchore/scan-action@v3
  with:
    image: "ghcr.io/your-org/iot-platform:${{ github.sha }}"
    fail-build: true
    severity-cutoff: critical
```

---

## 20.5 Security Headers

All API responses include defensive HTTP headers:

```go
// internal/middleware/security_headers.go
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        h := w.Header()
        h.Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload")
        h.Set("X-Content-Type-Options",    "nosniff")
        h.Set("X-Frame-Options",           "DENY")
        h.Set("X-XSS-Protection",          "1; mode=block")
        h.Set("Referrer-Policy",           "strict-origin-when-cross-origin")
        h.Set("Content-Security-Policy",
            "default-src 'none'; frame-ancestors 'none'")
        h.Set("Permissions-Policy",        "geolocation=(), camera=(), microphone=()")
        next.ServeHTTP(w, r)
    })
}
```
