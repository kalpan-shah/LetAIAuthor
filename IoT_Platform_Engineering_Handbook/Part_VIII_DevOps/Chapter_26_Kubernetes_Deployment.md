# Chapter 26: Kubernetes Deployment

> **Part VIII — DevOps**

The complete Kubernetes deployment uses Kustomize for environment overlays and ArgoCD for GitOps synchronisation. Every resource is designed for zero-downtime rolling updates.

---

## 26.1 Kustomize Layout

```
deploy/k8s/
├── base/                         # Shared across all environments
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── serviceaccount.yaml
│   ├── api-deployment.yaml
│   ├── api-service.yaml
│   ├── worker-deployment.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── network-policy.yaml
└── overlays/
    ├── local/                    # Docker Desktop / k3s
    │   ├── kustomization.yaml    # replicas: 1, no HPA, debug logging
    │   └── patches/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    └── production/
        ├── kustomization.yaml    # replicas: 3, HPA, production image tag
        └── patches/
```

---

## 26.2 API Deployment

```yaml
# deploy/k8s/base/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iot-api
  namespace: iot-platform
  labels:
    app:     iot-api
    version: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels: { app: iot-api }

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge:       1    # One extra pod during update
      maxUnavailable: 0    # Never reduce below desired capacity

  template:
    metadata:
      labels: { app: iot-api }
      annotations:
        # Vault Agent Injector — mounts secrets as files before app starts
        vault.hashicorp.com/agent-inject:              "true"
        vault.hashicorp.com/role:                      "iot-platform-api"
        vault.hashicorp.com/agent-inject-secret-db:    "database/creds/iot-platform"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "database/creds/iot-platform" -}}
          DATABASE_URL=postgres://{{ .Data.username }}:{{ .Data.password }}@postgres:5432/iotplatform?sslmode=require
          {{- end }}
        # Prometheus auto-discovery
        prometheus.io/scrape: "true"
        prometheus.io/port:   "8080"
        prometheus.io/path:   "/metrics"

    spec:
      serviceAccountName: iot-platform-api

      # Pod-level security context
      securityContext:
        runAsNonRoot: true
        runAsUser:    65534
        runAsGroup:   65534
        fsGroup:      65534
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: api
          image: ghcr.io/your-org/iot-platform:sha-abc1234  # Managed by ArgoCD/kustomize
          imagePullPolicy: IfNotPresent

          ports:
            - { name: http, containerPort: 8080, protocol: TCP }

          env:
            - { name: PORT,      value: "8080" }
            - { name: ENV,       value: "production" }
            - { name: LOG_LEVEL, value: "info" }
            - { name: OTEL_SERVICE_NAME, value: "iot-platform-api" }
            - { name: OTEL_EXPORTER_OTLP_ENDPOINT, value: "http://jaeger:4318" }
            - name: NATS_URL
              valueFrom:
                secretKeyRef:
                  name: platform-secrets
                  key:  nats-url

          # Container-level security context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem:   true
            capabilities: { drop: [ALL] }

          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 256Mi }

          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds:       30
            timeoutSeconds:      5
            failureThreshold:    3

          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds:       10
            timeoutSeconds:      3
            failureThreshold:    2

          startupProbe:
            httpGet: { path: /healthz, port: 8080 }
            failureThreshold: 30
            periodSeconds:    5     # 30 × 5s = 150s max startup time

          # Writable temp dir for Vault Agent
          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}

      # Spread pods across nodes for HA
      topologySpreadConstraints:
        - maxSkew:           1
          topologyKey:       kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: { app: iot-api }

      # Prefer different availability zones
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: topology.kubernetes.io/zone
                labelSelector:
                  matchLabels: { app: iot-api }
```

---

## 26.3 Horizontal Pod Autoscaler

```yaml
# deploy/k8s/base/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: iot-api-hpa
  namespace: iot-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind:       Deployment
    name:       iot-api

  minReplicas: 3
  maxReplicas: 20

  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
    - type: Resource
      resource:
        name: memory
        target: { type: Utilization, averageUtilization: 80 }

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 minutes before scaling down
      policies:
        - type: Pods, value: 1, periodSeconds: 60  # Remove at most 1 pod/min
    scaleUp:
      stabilizationWindowSeconds: 0     # React immediately
      policies:
        - type: Pods, value: 4, periodSeconds: 60  # Add up to 4 pods/min
```

---

## 26.4 PodDisruptionBudget

```yaml
# deploy/k8s/base/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: iot-api-pdb
  namespace: iot-platform
spec:
  minAvailable: 2    # At least 2 pods must remain during node drains / upgrades
  selector:
    matchLabels: { app: iot-api }
```

---

## 26.5 Readiness & Liveness Handlers

```go
// internal/handler/health.go
package handler

import (
    "context"
    "encoding/json"
    "net/http"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/redis/go-redis/v9"
    "github.com/your-org/iot-platform/internal/vault"
)

func Liveness() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("ok"))
    }
}

// Readiness checks all critical dependencies before accepting traffic.
// Returns 503 if any dependency is unhealthy.
func Readiness(db *pgxpool.Pool, rdb *redis.Client, v *vault.Client) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
        defer cancel()

        status := map[string]string{}
        healthy := true

        // PostgreSQL
        if err := db.Ping(ctx); err != nil {
            status["postgres"] = "unhealthy: " + err.Error()
            healthy = false
        } else {
            status["postgres"] = "ok"
        }

        // Redis
        if err := rdb.Ping(ctx).Err(); err != nil {
            status["redis"] = "unhealthy: " + err.Error()
            healthy = false
        } else {
            status["redis"] = "ok"
        }

        // Vault — check we can still authenticate
        if err := v.HealthCheck(ctx); err != nil {
            status["vault"] = "unhealthy: " + err.Error()
            healthy = false
        } else {
            status["vault"] = "ok"
        }

        code := http.StatusOK
        if !healthy {
            code = http.StatusServiceUnavailable
        }
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(code)
        json.NewEncoder(w).Encode(status)
    }
}
```

---

## 26.6 ArgoCD Application (GitOps)

```yaml
# deploy/k8s/argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name:      iot-platform
  namespace: argocd
  annotations:
    # Notify Slack on failed syncs
    notifications.argoproj.io/subscribe.on-sync-failed.slack:    deployments
    notifications.argoproj.io/subscribe.on-health-degraded.slack: deployments
spec:
  project: default

  source:
    repoURL:        https://github.com/your-org/iot-platform
    targetRevision: main
    path:           deploy/k8s/overlays/production

  destination:
    server:    https://kubernetes.default.svc
    namespace: iot-platform

  syncPolicy:
    automated:
      prune:    true    # Remove Kubernetes resources deleted from Git
      selfHeal: true    # Re-sync if cluster state drifts from Git

    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
      - ServerSideApply=true

    retry:
      limit: 5
      backoff:
        duration:    5s
        factor:      2
        maxDuration: 3m

  ignoreDifferences:
    # Vault Agent Injector adds init containers at runtime
    - group: apps
      kind:  Deployment
      jsonPointers:
        - /spec/template/spec/initContainers
        - /spec/template/spec/volumes
```
