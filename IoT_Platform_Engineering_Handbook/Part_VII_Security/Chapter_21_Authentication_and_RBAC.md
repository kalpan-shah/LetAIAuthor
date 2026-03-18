# Chapter 21: Authentication, Authorisation & RBAC

> **Part VII — Security (SecOps)**

---

## 21.1 RBAC Role Definitions

| Role | Permissions |
|---|---|
| `platform:admin` | Full access to all tenants and all operations. Only platform operators hold this role. Cannot be granted by tenant owners. |
| `tenant:owner` | Full access within assigned tenants: create/delete devices, manage integrations, view all audit logs, grant tenant-scoped roles. |
| `tenant:operator` | Provision and manage devices, rotate certificates. Cannot delete tenants or modify integrations. |
| `tenant:viewer` | Read-only access to devices, certificates, and audit logs within their tenants. |
| `service:worker` | Internal service accounts for NATS consumers and background workers. Cannot access human-facing endpoints. |

---

## 21.2 JWT Token Structure

```json
{
  "iss":   "iot-platform",
  "aud":   ["iot-platform"],
  "sub":   "usr_abc123",
  "exp":   1718000000,
  "iat":   1717913600,
  "jti":   "jwt_unique_id_for_revocation",
  "uid":   "usr_abc123",
  "email": "ops@acme.com",
  "roles": ["tenant:operator"],
  "tids":  ["t-abc123", "t-def456"]
}
```

- **`roles`**: RBAC roles the user holds
- **`tids`**: Tenant IDs this user is authorised to manage (empty for `platform:admin` — they access all)
- **`jti`**: Unique token ID used for logout/revocation (stored in Redis blocklist until expiry)

---

## 21.3 Auth Service

```go
// internal/auth/service.go
package auth

import (
    "context"
    "errors"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
    "github.com/redis/go-redis/v9"
)

type Claims struct {
    jwt.RegisteredClaims
    UserID    string   `json:"uid"`
    Email     string   `json:"email"`
    Roles     []string `json:"roles"`
    TenantIDs []string `json:"tids"`
}

type Service struct {
    secret []byte
    redis  *redis.Client
    issuer string
}

func NewService(secret string, rdb *redis.Client) *Service {
    return &Service{secret: []byte(secret), redis: rdb, issuer: "iot-platform"}
}

func (s *Service) IssueAccessToken(ctx context.Context, user User) (string, error) {
    now := time.Now()
    claims := Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            Issuer:    s.issuer,
            Audience:  []string{"iot-platform"},
            Subject:   user.ID,
            IssuedAt:  jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(24 * time.Hour)),
            ID:        uuid.New().String(),
        },
        UserID:    user.ID,
        Email:     user.Email,
        Roles:     user.Roles,
        TenantIDs: user.TenantIDs,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(s.secret)
}

func (s *Service) ValidateToken(ctx context.Context, raw string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(raw, &Claims{}, func(t *jwt.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return s.secret, nil
    },
        jwt.WithIssuer(s.issuer),
        jwt.WithAudience("iot-platform"),
        jwt.WithExpirationRequired(),
    )
    if err != nil {
        return nil, fmt.Errorf("invalid token: %w", err)
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("token claims invalid")
    }

    // Check revocation list (set on logout)
    revoked, _ := s.redis.Exists(ctx, "revoked:jwt:"+claims.ID).Result()
    if revoked > 0 {
        return nil, errors.New("token has been revoked")
    }
    return claims, nil
}

func (s *Service) RevokeToken(ctx context.Context, jti string, expiry time.Time) error {
    ttl := time.Until(expiry)
    if ttl <= 0 {
        return nil // Already expired
    }
    return s.redis.Set(ctx, "revoked:jwt:"+jti, "1", ttl).Err()
}
```

---

## 21.4 Authentication Middleware

```go
// internal/middleware/auth.go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "github.com/go-chi/chi/v5"
    "github.com/your-org/iot-platform/internal/auth"
)

type contextKey string

const actorKey contextKey = "actor"

type Actor struct {
    ID        string
    Email     string
    Roles     []string
    TenantIDs []string
    TokenJTI  string
}

func Authenticate(authSvc *auth.Service) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            header := r.Header.Get("Authorization")
            if !strings.HasPrefix(header, "Bearer ") {
                writeJSON(w, 401, map[string]string{
                    "error": "authentication required",
                    "code":  "unauthenticated",
                })
                return
            }

            claims, err := authSvc.ValidateToken(r.Context(), strings.TrimPrefix(header, "Bearer "))
            if err != nil {
                writeJSON(w, 401, map[string]string{
                    "error": "invalid or expired token",
                    "code":  "unauthenticated",
                })
                return
            }

            actor := &Actor{
                ID:        claims.UserID,
                Email:     claims.Email,
                Roles:     claims.Roles,
                TenantIDs: claims.TenantIDs,
                TokenJTI:  claims.ID,
            }
            ctx := context.WithValue(r.Context(), actorKey, actor)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// TenantScope validates the actor has access to the tenant in the URL path.
func TenantScope(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        tenantID := chi.URLParam(r, "tenantId")
        actor    := ActorFromCtx(r.Context())
        if actor == nil {
            writeJSON(w, 401, map[string]string{"error": "not authenticated", "code": "unauthenticated"})
            return
        }

        // platform:admin bypasses tenant scoping
        for _, role := range actor.Roles {
            if role == "platform:admin" {
                next.ServeHTTP(w, r)
                return
            }
        }

        // Other roles must be listed in the JWT tids claim
        for _, tid := range actor.TenantIDs {
            if tid == tenantID {
                next.ServeHTTP(w, r)
                return
            }
        }

        writeJSON(w, 403, map[string]string{
            "error": "access denied to this tenant",
            "code":  "forbidden",
        })
    })
}

func ActorFromCtx(ctx context.Context) *Actor {
    a, _ := ctx.Value(actorKey).(*Actor)
    return a
}
```

---

## 21.5 Permission Matrix

| Endpoint | `platform:admin` | `tenant:owner` | `tenant:operator` | `tenant:viewer` |
|---|---|---|---|---|
| `POST /tenants` | ✅ | ❌ | ❌ | ❌ |
| `GET /tenants` | ✅ | ❌ | ❌ | ❌ |
| `POST /tenants/{id}/suspend` | ✅ | ❌ | ❌ | ❌ |
| `POST /tenants/{id}/integrations` | ✅ | ✅ | ❌ | ❌ |
| `POST /tenants/{id}/devices` | ✅ | ✅ | ✅ | ❌ |
| `GET /tenants/{id}/devices` | ✅ | ✅ | ✅ | ✅ |
| `DELETE /tenants/{id}/devices/{dId}` | ✅ | ✅ | ❌ | ❌ |
| `POST /tenants/{id}/devices/{dId}/certificates/rotate` | ✅ | ✅ | ✅ | ❌ |
| `GET /tenants/{id}/audit-logs` | ✅ | ✅ | ❌ | ❌ |
