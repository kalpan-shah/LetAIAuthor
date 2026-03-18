# Chapter 24: CI/CD Pipelines with GitHub Actions & ArgoCD

> **Part VIII — DevOps**

Every code change goes through a five-stage pipeline before it reaches production. ArgoCD then handles the actual deployment using a GitOps model — the cluster converges to whatever state is in the Git repository.

---

## 24.1 Pipeline Architecture

```
Developer pushes code / opens PR
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│              GitHub Actions CI Pipeline                         │
│                                                                 │
│  ┌──────┐  ┌───────────┐  ┌─────────────────┐  ┌──────────┐  │
│  │ lint │─►│ test-unit │─►│ test-integration │─►│ security │  │
│  └──────┘  └───────────┘  └─────────────────┘  └────┬─────┘  │
│                                                       │         │
│                                              ┌────────▼──────┐  │
│                                              │  build-push   │  │
│                                              │ ghcr.io/...:  │  │
│                                              │ sha-abc1234   │  │
│                                              └────────┬──────┘  │
│                                                       │         │
│                                              ┌────────▼──────┐  │
│                                              │  update-tag   │  │
│                                              │  (main only)  │  │
│                                              └───────────────┘  │
└───────────────────────────────────────────────────┬─────────────┘
                                                     │
                                         Commits updated image tag
                                         to deploy/k8s/overlays/production/
                                                     │
                                         ArgoCD detects change (polling 3m)
                                                     │
                                         Rolling deploy to cluster
                                         maxSurge=1, maxUnavailable=0
```

---

## 24.2 Complete GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, "release/**"]
  pull_request:
    branches: [main]

env:
  GO_VERSION: "1.22"
  IMAGE: ghcr.io/${{ github.repository }}/iot-platform

jobs:

  lint:
    name: Lint
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "${{ env.GO_VERSION }}" }
      - uses: golangci/golangci-lint-action@v6
        with:
          version: v1.59
          args: --timeout=5m

  test-unit:
    name: Unit Tests
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "${{ env.GO_VERSION }}" }
      - run: go test ./... -short -race -count=1 -cover -coverprofile=coverage.out
      - uses: codecov/codecov-action@v4
        with: { files: coverage.out }

  test-integration:
    name: Integration Tests
    runs-on: ubuntu-24.04
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB:       testdb
          POSTGRES_USER:     test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-retries 10
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
      nats:
        image: nats:2.10-alpine
        args: ["-js"]
        ports: ["4222:4222"]
    env:
      DATABASE_URL: postgres://test:test@localhost:5432/testdb?sslmode=disable
      REDIS_URL:    redis://localhost:6379/0
      NATS_URL:     nats://localhost:4222
      VAULT_ADDR:   http://localhost:8200
      VAULT_TOKEN:  root
      ENV:          test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "${{ env.GO_VERSION }}" }

      - name: Start Vault in dev mode
        run: |
          docker run -d --network=host \
            -e VAULT_DEV_ROOT_TOKEN_ID=root \
            hashicorp/vault:1.17
          sleep 4
          bash scripts/init-vault.sh

      - name: Run database migrations
        run: go run ./cmd/migrate up

      - name: Run integration tests
        run: go test ./... -run Integration -race -count=1 -timeout=120s

  security:
    name: Security Scan
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: GitLeaks secret scan
        uses: gitleaks/gitleaks-action@v2

      - name: gosec static analysis
        uses: securego/gosec@master
        with: { args: "-severity medium ./..." }

      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type:  fs
          scan-ref:   "."
          severity:   CRITICAL,HIGH
          exit-code:  "1"

  build-push:
    name: Build & Push Image
    needs: [lint, test-unit, test-integration, security]
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=sha,prefix=sha-,format=short
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v6
        with:
          context:    "."
          file:       deploy/docker/Dockerfile
          platforms:  linux/amd64,linux/arm64
          push:       true
          tags:       ${{ steps.meta.outputs.tags }}
          labels:     ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to:   type=gha,mode=max

  update-image-tag:
    name: Update Image Tag (GitOps)
    needs: build-push
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GH_PAT }}  # PAT with repo write access
          ref:   main

      - name: Update kustomization with new image tag
        run: |
          SHORT_SHA=$(echo "${{ github.sha }}" | head -c 7)
          cd deploy/k8s/overlays/production
          kustomize edit set image \
            iot-platform=${{ env.IMAGE }}:sha-${SHORT_SHA}
          git config user.email "ci@iotplatform.io"
          git config user.name  "CI Bot"
          git commit -am "chore(deploy): update image to sha-${SHORT_SHA} [skip ci]"
          git push
      # ArgoCD polls this repo every 3 minutes and triggers a rolling deploy
```

---

## 24.3 Production Dockerfile

```dockerfile
# Multi-stage build: compile → scratch
# Final image: ~15MB, no OS, no shell, no package manager

FROM golang:1.22-alpine AS builder
RUN apk add --no-cache ca-certificates git tzdata
WORKDIR /build

# Cache dependencies separately from source
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-w -s -extldflags=-static \
              -X main.version=$(git describe --tags --always) \
              -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    -trimpath \
    -o /build/api \
    ./cmd/api

# Verify the binary is statically linked
RUN ldd /build/api 2>&1 | grep -q "not a dynamic executable" || true

# ── Final image: scratch (no OS at all) ──────────────────────────────────────
FROM scratch

# CA certificates for HTTPS to AWS/Azure APIs
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Timezone data (for structured log timestamps)
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# The binary
COPY --from=builder /build/api /api

# Run as non-root numeric UID (works in scratch — no /etc/passwd needed)
USER 65534:65534

EXPOSE 8080

# Disable shell signal handling — ENTRYPOINT exec form only
ENTRYPOINT ["/api"]
```

---

## 24.4 golangci-lint Configuration

```yaml
# .golangci.yml
run:
  timeout: 5m
  go: "1.22"

linters:
  enable:
    - errcheck       # Check all errors are handled
    - gosimple       # Simplification suggestions
    - govet          # go vet checks
    - ineffassign    # Detect unused assignments
    - staticcheck    # Static analysis
    - unused         # Find unused code
    - gosec          # Security lints
    - noctx          # HTTP requests without context
    - bodyclose      # HTTP response body not closed
    - sqlclosecheck  # SQL rows not closed
    - contextcheck   # Incorrect context propagation
    - exhaustive     # Switch exhaustiveness for enums

linters-settings:
  gosec:
    excludes:
      - G104  # Errors unhandled (too noisy in tests)
  errcheck:
    check-type-assertions: true

issues:
  exclude-rules:
    - path: _test\.go
      linters: [gosec, errcheck]
```
