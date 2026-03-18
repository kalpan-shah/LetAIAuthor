# Chapter 16: REST API Design & Implementation

> **Part V — Developer Interfaces**

The REST API is the primary interface for all developer and operator interactions. It is designed around OpenAPI 3.1, implemented with the Chi router, and protected by a middleware chain that enforces authentication, authorisation, rate limiting, and tenant scoping.

---

## 16.1 API Design Principles

- **Versioned routes:** all routes begin `/api/v1/`. When breaking changes are required, `/api/v2/` is introduced — v1 is never modified.
- **Tenant-scoped resources:** device and certificate routes are always nested under `/api/v1/tenants/{tenantId}/...`. This enforces tenant scoping at the routing level.
- **Idempotency keys:** provisioning requests accept an `Idempotency-Key` header. Duplicate requests within 24 hours return the original response.
- **Consistent error format:** `{"error":"human message","code":"machine_code","request_id":"uuid"}`.
- **One-time private key delivery:** The provisioning response includes `private_key_pem`. This is the only time it is returned.

---

## 16.2 Complete Route Map

```
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/logout

POST   /api/v1/tenants
GET    /api/v1/tenants
GET    /api/v1/tenants/{id}
PATCH  /api/v1/tenants/{id}
POST   /api/v1/tenants/{id}/suspend
POST   /api/v1/tenants/{id}/reactivate

POST   /api/v1/tenants/{id}/integrations
GET    /api/v1/tenants/{id}/integrations
POST   /api/v1/tenants/{id}/integrations/{prov}/validate
DELETE /api/v1/tenants/{id}/integrations/{prov}

POST   /api/v1/tenants/{id}/devices             ← Provision device
GET    /api/v1/tenants/{id}/devices             ← List (paginated)
GET    /api/v1/tenants/{id}/devices/{deviceId}
PATCH  /api/v1/tenants/{id}/devices/{deviceId}
POST   /api/v1/tenants/{id}/devices/{deviceId}/suspend
POST   /api/v1/tenants/{id}/devices/{deviceId}/reactivate
DELETE /api/v1/tenants/{id}/devices/{deviceId}

GET    /api/v1/tenants/{id}/devices/{dId}/certificates
POST   /api/v1/tenants/{id}/devices/{dId}/certificates/rotate
POST   /api/v1/tenants/{id}/certificates/{certId}/revoke

POST   /api/v1/tenants/{id}/routing-rules
GET    /api/v1/tenants/{id}/routing-rules
PUT    /api/v1/tenants/{id}/routing-rules/{ruleId}
DELETE /api/v1/tenants/{id}/routing-rules/{ruleId}

GET    /api/v1/tenants/{id}/audit-logs

GET    /healthz    ← Kubernetes liveness
GET    /readyz     ← Kubernetes readiness
GET    /metrics    ← Prometheus
```

---

## 16.3 Chi Router & Middleware Chain

```go
// cmd/api/router.go
func buildRouter(deps *Deps) http.Handler {
	r := chi.NewRouter()

	// Global middleware (every request)
	r.Use(middleware.RequestID)        // Inject X-Request-ID
	r.Use(mw.StructuredLogger)         // JSON request logging
	r.Use(mw.Recover)                  // Panic → 500
	r.Use(mw.Prometheus)               // Latency + status per route
	r.Use(mw.Tracing(deps.Tracer))     // OpenTelemetry span
	r.Use(mw.CORS(deps.Config.AllowedOrigins))
	r.Use(mw.SecurityHeaders)          // HSTS, X-Frame-Options, CSP
	r.Use(mw.RateLimit(deps.Redis, 1000, 60)) // 1000 req/min per IP

	// Public routes
	r.Group(func(r chi.Router) {
		r.Post("/api/v1/auth/login",   handler.Login(deps.Auth))
		r.Post("/api/v1/auth/refresh", handler.RefreshToken(deps.Auth))
		r.Get("/healthz",              handler.Liveness())
		r.Get("/readyz",               handler.Readiness(deps.DB, deps.Redis, deps.Vault))
		r.Get("/metrics",              handler.Metrics())
	})

	// Authenticated routes
	r.Group(func(r chi.Router) {
		r.Use(mw.Authenticate(deps.Auth))
		r.Use(mw.Authorise)

		r.Route("/api/v1/tenants", func(r chi.Router) {
			r.Post("/", handler.CreateTenant(deps.TenantSvc))
			r.Get("/",  handler.ListTenants(deps.TenantSvc))

			r.Route("/{tenantId}", func(r chi.Router) {
				r.Use(mw.TenantScope(deps.TenantSvc))
				r.Get("/",     handler.GetTenant(deps.TenantSvc))
				r.Patch("/",   handler.UpdateTenant(deps.TenantSvc))
				r.Post("/suspend",    handler.SuspendTenant(deps.TenantSvc))
				r.Post("/reactivate", handler.ReactivateTenant(deps.TenantSvc))

				r.Route("/devices", func(r chi.Router) {
					r.Post("/", handler.ProvisionDevice(deps.DeviceSvc))
					r.Get("/",  handler.ListDevices(deps.DeviceSvc))
					r.Get("/{deviceId}",    handler.GetDevice(deps.DeviceSvc))
					r.Patch("/{deviceId}",  handler.UpdateDevice(deps.DeviceSvc))
					r.Post("/{deviceId}/suspend",   handler.SuspendDevice(deps.DeviceSvc))
					r.Post("/{deviceId}/reactivate",handler.ReactivateDevice(deps.DeviceSvc))
					r.Delete("/{deviceId}", handler.DeleteDevice(deps.DeviceSvc))
					r.Post("/{deviceId}/certificates/rotate",
						handler.RotateCertificate(deps.DeviceSvc))
				})
				r.Get("/audit-logs", handler.ListAuditLogs(deps.AuditSvc))
			})
		})
	})
	return r
}
```

---

## 16.4 Provision Device Handler

```go
// internal/handler/device.go

func ProvisionDevice(svc *device.Service) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		tenantID, err := uuid.Parse(chi.URLParam(r, "tenantId"))
		if err != nil {
			writeError(w, 400, "invalid_tenant_id", "bad uuid"); return
		}
		actor := mw.ActorFromCtx(r.Context())

		var body struct {
			Name        string            `json:"name"        validate:"required"`
			Description string            `json:"description"`
			Metadata    map[string]string `json:"metadata"`
		}
		if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
			writeError(w, 400, "invalid_request", "cannot decode JSON"); return
		}

		result, err := svc.Provision(r.Context(), device.ProvisionReq{
			TenantID: tenantID, Name: body.Name,
			Description: body.Description, Metadata: body.Metadata,
			ActorID: actor.ID,
		})
		if err != nil {
			switch {
			case isQuotaError(err):
				writeError(w, 429, "quota_exceeded", err.Error())
			case isNotFoundError(err):
				writeError(w, 404, "tenant_not_found", err.Error())
			default:
				writeError(w, 500, "provision_failed", err.Error())
			}
			return
		}

		// 201 Created — private key delivered ONCE
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(map[string]any{
			"device":   result.Device,
			"endpoint": result.Endpoint,
			"certificate": map[string]any{
				"id":              result.CertBundle.ID,
				"certificate_pem": result.CertBundle.CertificatePEM,
				"ca_cert_pem":     result.CertBundle.CACertPEM,
				"private_key_pem": result.CertBundle.PrivateKeyPEM, // ONE TIME
				"not_after":       result.CertBundle.NotAfter,
			},
			"note": "private_key_pem is returned ONCE. Store it immediately.",
		})
	}
}
```

---

## 16.5 Example API Responses

**Successful provisioning (201):**
```json
{
  "device": {
    "id": "d-abc123",
    "tenant_id": "t-xyz456",
    "name": "factory-sensor-01",
    "status": "active",
    "provider": "mosquitto",
    "provisioned_at": "2025-06-15T10:23:45Z"
  },
  "endpoint": "localhost:8883",
  "certificate": {
    "id": "cert-789",
    "certificate_pem": "-----BEGIN CERTIFICATE-----\n...",
    "ca_cert_pem": "-----BEGIN CERTIFICATE-----\n...",
    "private_key_pem": "-----BEGIN EC PRIVATE KEY-----\n...",
    "not_after": "2026-06-15T10:23:45Z"
  },
  "note": "private_key_pem is returned ONCE. Store it immediately."
}
```

**Error (quota exceeded, 429):**
```json
{
  "error": "device quota exceeded: 1000/1000",
  "code": "quota_exceeded",
  "request_id": "req-e3f2a1"
}
```
