# Chapter 2: Technology Stack — Rationale & Selection

> **Part I — Foundations**

Every technology in this stack was chosen against three criteria: provider agnosticism (works without a cloud account), operational maturity (battle-tested at scale with strong community support), and developer ergonomics (clear documentation, strong type safety, fast iteration loops).

---

## 2.1 Full Stack Overview

| Layer | Technology | Why This and Not Alternatives |
|---|---|---|
| **API Runtime** | Go 1.22 | Native concurrency, tiny binaries (scratch Docker images), strong type safety, AWS/Azure SDKs both first-class. Python: slower, GIL limits concurrency. Java: JVM overhead. |
| **API Framework** | Chi + OpenAPI 3.1 | Chi is minimal idiomatic Go — no magic. OpenAPI drives code generation and documentation simultaneously. Gin is fine too; Chi is simpler. |
| **Database** | PostgreSQL 16 + sqlc | ACID for device state. sqlc generates type-safe Go from raw SQL — no ORM magic, no N+1 surprises. Alternatives: CockroachDB for global distribution (same SQL API). |
| **Cache** | Redis 7 | Rate limiting, session storage, ephemeral pub-sub. Universal operator knowledge. Alternatives: Valkey (Redis fork) is a drop-in. |
| **Event Bus** | NATS JetStream | Durable, persistent, embeddable. Single binary — no ZooKeeper, no Kafka cluster management. 18M msgs/sec per node. Kafka: heavier ops but higher ecosystem. |
| **Secrets** | HashiCorp Vault | PKI engine for device certificates, dynamic secrets for DB creds, audit log. Industry standard. OpenBao is the open-source fork. AWS Secrets Manager: cloud-only. |
| **Container Runtime** | Docker (local), containerd (prod) | Docker: universal for local dev. containerd: Kubernetes-native, slimmer. |
| **Orchestration** | Kubernetes 1.29+ | Runs identically on EKS, AKS, GKE, bare metal, k3s. Provider-agnostic operations. No lock-in. |
| **IaC** | Terraform 1.8 / OpenTofu 1.7 | Multi-provider IaC. OpenTofu is the CNCF-hosted open fork — fully compatible. Use either. |
| **CI/CD** | GitHub Actions + ArgoCD | Actions: build/test/scan/publish. ArgoCD: GitOps sync to cluster. Works with GitLab CI and Flux too. |
| **Metrics** | Prometheus + Grafana | The observability standard. No vendor lock-in, open data format. |
| **Logs** | Loki + Fluent Bit | Grafana-native, cheap (object storage backend). Less querying power than Elasticsearch but 10x cheaper at scale. |
| **Tracing** | OpenTelemetry + Jaeger | OTel is the vendor-neutral standard. Jaeger for local. Tempo for production. Datadog / Honeycomb are also valid OTel backends. |
| **API Gateway** | Kong (production) / Traefik (local) | Kong: mature, plugin ecosystem, Kubernetes-native CRDs. Traefik: zero config for Docker Compose. Both support JWT validation, rate limiting. |
| **Local IoT Broker** | Eclipse Mosquitto 2.x | Standards-compliant MQTT 3.1.1/5.0. Docker-ready. Free forever. The fully local provider that requires zero cloud access. |

---

## 2.2 Why Go?

Go was chosen as the primary language after evaluating Python, Rust, Java, and Node.js. The decision factors were:

- **Goroutines:** IoT provisioning is inherently concurrent — many tenants provisioning devices simultaneously. Go's goroutine model makes this safe and readable without callback chains or async/await complexity.
- **Static binary compilation:** `CGO_ENABLED=0 go build` produces a single static binary. Docker images can be `FROM scratch` — 15 MB, no OS, minimal attack surface. Python images are typically 200–800 MB.
- **Type safety without ceremony:** Go's type system catches interface mismatches and nil dereferences at compile time. Unlike Rust, there is no ownership complexity. Unlike Java, there is no generics ceremony for simple cases.
- **Tooling:** `gofmt`, `go vet`, `golangci-lint`, `gopls`. The entire toolchain is standardised. No ESLint config wars, no pip version conflicts.
- **AWS and Azure SDKs:** AWS SDK v2 for Go and Azure SDK for Go are both production-grade, well-maintained, and idiomatic.

---

## 2.3 Why NATS JetStream Over Kafka?

Kafka is excellent but operationally expensive — it requires ZooKeeper (or KRaft in newer versions), separate broker processes, and careful partition management. For this platform's use case (internal control-plane events, not device telemetry), NATS JetStream provides everything needed:

| Requirement | NATS JetStream |
|---|---|
| Durable message persistence | Stored on disk, survives broker restart |
| At-least-once delivery | Consumer ACK with configurable retry |
| Exactly-once semantics | Deduplication window per stream |
| Cluster operation | Built-in clustering, no external coordinator |
| Local dev experience | Single Docker container, zero config |
| Throughput | 18–30M msgs/sec per node |
| Message replay | Consumers can replay from any point in the stream |
| Operational complexity | Single binary, embedded in Go if needed |

> **💡 TIP:** NATS JetStream does not replace your IoT provider. Device telemetry still flows to AWS IoT Core, Azure IoT Hub, or Mosquitto. NATS is only used for internal platform events: DeviceProvisioned, CertificateRotated, TenantSuspended, etc.

---

## 2.4 Why Vault for Secrets?

Every IoT device requires an X.509 certificate. At 10,000 devices, managing certificates manually is impossible. Vault provides:

- **PKI Secrets Engine:** Issue a signed certificate with a single API call. Vault manages the certificate authority chain, enforces TTLs, and maintains a CRL automatically.
- **Dynamic Secrets:** Instead of storing a static database password, Vault generates a short-lived database username/password on request and automatically revokes it when the lease expires. No rotation scripts.
- **Cloud credential brokering:** Vault can generate short-lived AWS IAM credentials or Azure SAS tokens. The platform never stores permanent cloud credentials.
- **Audit log:** Every secret access — who, what, when — is recorded. This is a compliance requirement in many industries.
- **Provider agnostic:** Vault runs on Docker, Kubernetes, bare metal, any cloud. The same Vault setup works for local dev and production.

> **⚠️ WARNING:** Vault's **dev mode** (`vault server -dev`) runs entirely in memory, auto-unseals with a root token, and is suitable only for local development. Never run dev mode in production. Production Vault requires a proper unseal strategy — either auto-unseal with a cloud KMS (AWS KMS, Azure Key Vault, GCP KMS) or Shamir secret sharing with at least 3 key holders.
