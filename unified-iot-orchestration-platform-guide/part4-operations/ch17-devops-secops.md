# Chapter 17: DevOps & SecOps Workflows

> *"DevOps without SecOps is a highway without guardrails."*

---

## 17.1 Introduction

This chapter defines the operational workflows that keep the platform running securely and reliably. We draw a clear line between **DevOps** (build, test, deploy) and **SecOps** (scan, audit, respond) responsibilities, while acknowledging they overlap significantly in practice.

---

## 17.2 DevOps vs. SecOps Boundaries

| Concern | DevOps | SecOps |
|---------|--------|--------|
| **CI/CD pipeline** | ✅ Owns | Reviews & gates |
| **Dependency updates** | Implements | ✅ Prioritizes CVEs |
| **Container images** | Builds | ✅ Scans & approves |
| **Infrastructure as Code** | Writes | ✅ Reviews for security |
| **Secret rotation** | Implements automation | ✅ Defines policy |
| **Incident response** | First responder | ✅ Coordinates if security-related |
| **Access control** | Implements RBAC | ✅ Defines roles & policies |
| **Compliance** | Provides evidence | ✅ Owns audit |

---

## 17.3 CI/CD Pipeline — Complete

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─── STAGE 1: Quality Gates ─────────────────────
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - uses: golangci/golangci-lint-action@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install ruff && ruff check app/ && ruff format --check app/

  test:
    runs-on: ubuntu-latest
    services:
      postgres: { image: 'postgres:16', env: { POSTGRES_DB: test, POSTGRES_PASSWORD: test }, ports: ['5432:5432'] }
      redis: { image: 'redis:7', ports: ['6379:6379'] }
      nats: { image: 'nats:2.10', ports: ['4222:4222'], options: '--jetstream' }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - run: go test ./... -race -coverprofile=coverage.out
      - run: go tool cover -func=coverage.out | tail -1
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements-dev.txt && pytest --cov=app --cov-report=xml

  # ─── STAGE 2: Security Scanning (SecOps Gate) ──────
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Go vulnerability check
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - run: go install golang.org/x/vuln/cmd/govulncheck@latest && govulncheck ./...

      # Python dependency audit
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install pip-audit && pip-audit -r requirements.txt

      # Secret scanning
      - uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified

      # SAST (Static Application Security Testing)
      - uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten

  # ─── STAGE 3: Build ────────────────────────────
  build:
    needs: [lint, test, security-scan]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Build Go image
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.go
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-go:${{ github.sha }}

      # Scan image for vulnerabilities (SecOps gate)
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-go:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: '1'

  # ─── STAGE 4: Deploy (main branch only) ────────
  deploy-staging:
    needs: [build]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v3
      - run: |
          helm upgrade --install iot-platform ./helm/iot-platform \
            --namespace iot-platform-staging \
            --set image.tag=${{ github.sha }} \
            --set env=staging \
            --wait --timeout 5m

  deploy-production:
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v3
      - run: |
          helm upgrade --install iot-platform ./helm/iot-platform \
            --namespace iot-platform-prod \
            --set image.tag=${{ github.sha }} \
            --set env=production \
            --set replicas=3 \
            --wait --timeout 10m
```

---

## 17.4 Deployment Strategy: Blue-Green with Canary

```
Phase 1: Deploy canary (5% traffic)
  │
  ├─ Monitor for 15 minutes
  │   ├─ Error rate < 0.1%? → Continue
  │   └─ Error rate > 0.1%? → Rollback immediately
  │
Phase 2: Expand to 25% traffic
  │
  ├─ Monitor for 15 minutes
  │   ├─ P99 latency within baseline? → Continue
  │   └─ Regression detected? → Rollback
  │
Phase 3: Full deployment (100%)
  │
  └─ Keep old version running for 1 hour
      └─ Instant rollback if issues surface
```

---

## 17.5 Secret Rotation Schedule

| Secret Type | Rotation Frequency | Automated? | Owner |
|------------|-------------------|-----------|-------|
| AWS STS sessions | Every hour | ✅ Auto | Platform |
| Azure client secrets | Every 90 days | ⚠️ Semi | SecOps + Tenant |
| Database passwords | Every 30 days | ✅ Vault dynamic | DevOps |
| API signing keys (JWT) | Every 7 days | ✅ Auto | Platform |
| Vault root token | Initial setup only | N/A | SecOps |
| PKI root CA | Every 10 years | Manual | SecOps |
| PKI intermediate CA | Every 1 year | ✅ Vault auto-rotate | Platform |

---

## 17.6 Incident Response Framework

### Severity Levels

| Level | Definition | Response Time | Example |
|-------|-----------|--------------|---------|
| **SEV1** | Total platform outage | 15 min | Database down, all APIs failing |
| **SEV2** | Partial outage, multi-tenant impact | 30 min | One provider adapter failing |
| **SEV3** | Single-tenant impact | 2 hours | Tenant credentials expired |
| **SEV4** | Degraded performance | Next business day | Elevated latency, non-critical |

### Security Incident Response

```
1. DETECT
   → Alert fires or report received
   → Assess: Is this a security incident?

2. CONTAIN
   → Isolate affected tenant(s)
   → Rotate potentially compromised credentials
   → Block suspicious IP addresses

3. INVESTIGATE
   → Review audit logs (HMAC-verified)
   → Check Vault access logs
   → Analyze provider CloudTrail/Activity logs

4. REMEDIATE
   → Patch vulnerability
   → Rotate all affected secrets
   → Update network policies if needed

5. RECOVER
   → Re-enable affected services
   → Re-provision affected devices if needed
   → Verify tenant isolation intact

6. POSTMORTEM
   → Document timeline
   → Identify root cause
   → Define prevention measures
   → Update runbooks and security checklist
```

---

## 17.7 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **DevOps/SecOps boundary** | DevOps builds and deploys; SecOps defines policy and gates |
| **CI/CD pipeline** | 4 stages: lint/test → security scan → build → deploy |
| **Canary deployment** | Progressive traffic shifting with automated rollback |
| **Secret rotation** | Scheduled per secret type, automated where possible |
| **Incident response** | Structured SEV levels with security-specific procedures |

### What's Next

**Part V: Advanced Topics** begins with **Chapter 18: Performance Optimization**.
