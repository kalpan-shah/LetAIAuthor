# Chapter 19: Disaster Recovery & High Availability

> *"Everything fails, all the time."* — Werner Vogels, CTO Amazon

---

## 19.1 Introduction

For an IoT control plane, "downtime" doesn't just mean the dashboard is unavailable — it means new devices can't be provisioned, certificates can't be rotated, and security policies can't be updated. This chapter covers how to design for failures and recover quickly when they happen.

---

## 19.2 Failure Modes

| Component | Failure Mode | Impact | RTO | RPO |
|-----------|-------------|--------|-----|-----|
| **PostgreSQL** | Primary crash | All writes fail | 30s (auto-failover) | 0 (sync replication) |
| **PostgreSQL** | Disk corruption | Data loss | 1h (restore from backup) | 15 min (WAL archiving) |
| **Redis** | Instance crash | Cache cold start, rate limiting gaps | 10s (restart) | Best-effort |
| **Vault** | Sealed/unavailable | No cert issuance, no credential access | 2 min (auto-unseal) | 0 |
| **NATS** | Instance crash | Audit events buffered at publisher | 30s (cluster failover) | 0 (JetStream replication) |
| **Application** | Pod crash | Brief request failures | 10s (K8s restart) | 0 (stateless) |
| **Provider API** | AWS/Azure outage | Can't provision for affected provider | N/A (external) | N/A |

> 💡 **RTO** = Recovery Time Objective (how fast you recover). **RPO** = Recovery Point Objective (how much data you can afford to lose).

---

## 19.3 High Availability Architecture

```
                        Load Balancer
                            │
                ┌───────────┼───────────┐
                │           │           │
           ┌────▼────┐ ┌───▼────┐ ┌───▼────┐
           │ App-1   │ │ App-2  │ │ App-3  │  (K8s replicas)
           └────┬────┘ └───┬────┘ └───┬────┘
                │          │          │
       ┌────────┴──────────┴──────────┴────────┐
       │                                        │
  ┌────▼────────┐                      ┌────────▼───────┐
  │ PostgreSQL  │                      │ PostgreSQL     │
  │ Primary     │───── Streaming ─────▶│ Replica        │
  │ (read/write)│     Replication      │ (read-only)    │
  └─────────────┘                      └────────────────┘
       │
  ┌────▼────────┐     ┌──────────────┐
  │ Redis       │     │ NATS Cluster │
  │ (Sentinel   │     │ (3-node      │
  │  cluster)   │     │  JetStream)  │
  └─────────────┘     └──────────────┘
       │
  ┌────▼────────┐
  │ Vault       │
  │ (3-node HA  │
  │  with Raft) │
  └─────────────┘
```

---

## 19.4 PostgreSQL HA with Streaming Replication

```yaml
# Kubernetes StatefulSet for PostgreSQL HA (using Patroni)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: PATRONI_SCOPE
              value: iot-platform
            - name: PATRONI_REPLICATION_USERNAME
              value: replicator
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
```

### Backup Strategy

| Type | Frequency | Retention | Storage |
|------|-----------|-----------|---------|
| **WAL archiving** | Continuous | 7 days | S3/GCS/Azure Blob |
| **Base backup** | Daily at 02:00 UTC | 30 days | S3/GCS/Azure Blob |
| **Logical backup** | Weekly | 90 days | S3/GCS/Azure Blob |

```bash
# Automated backup with pgBackRest
pgbackrest --stanza=iot-platform --type=full backup    # Weekly full
pgbackrest --stanza=iot-platform --type=diff backup    # Daily differential
pgbackrest --stanza=iot-platform --type=incr backup    # Hourly incremental
```

### Recovery Procedure

```bash
# Point-in-Time Recovery (PITR)
# Recover to 5 minutes before the incident
pgbackrest --stanza=iot-platform \
    --type=time \
    --target="2024-03-15 14:25:00" \
    --target-action=promote \
    restore
```

---

## 19.5 Vault HA with Raft

```hcl
# vault-config.hcl — Production HA configuration
storage "raft" {
  path    = "/vault/data"
  node_id = "vault-1"

  retry_join {
    leader_api_addr = "http://vault-0.vault:8200"
  }
  retry_join {
    leader_api_addr = "http://vault-1.vault:8200"
  }
  retry_join {
    leader_api_addr = "http://vault-2.vault:8200"
  }
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = false
  tls_cert_file = "/vault/tls/server.crt"
  tls_key_file  = "/vault/tls/server.key"
}

api_addr     = "https://vault-0.vault:8200"
cluster_addr = "https://vault-0.vault:8201"

seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/vault-unseal"
}
```

> ☕ *Legacy note*: If your organization already uses Azure Key Vault or AWS KMS for secret management, Vault can auto-unseal using those services — no operator intervention needed on restart.

---

## 19.6 Multi-Region Strategy

For global IoT deployments, consider a multi-region architecture:

```
Region A (Primary)              Region B (DR)
┌─────────────────┐            ┌─────────────────┐
│ App Cluster     │            │ App Cluster     │
│ (active)        │            │ (standby)       │
├─────────────────┤            ├─────────────────┤
│ PostgreSQL      │───WAL───▶  │ PostgreSQL      │
│ (primary)       │  shipping  │ (standby)       │
├─────────────────┤            ├─────────────────┤
│ Vault (active)  │───Raft──▶  │ Vault (follower)│
├─────────────────┤            ├─────────────────┤
│ NATS (cluster)  │◀──Mirror─▶│ NATS (cluster)  │
└─────────────────┘            └─────────────────┘
```

### Failover Decision Matrix

| Scenario | Action | Automated? |
|----------|--------|-----------|
| Single pod crash | K8s restart | ✅ |
| Database primary crash | Patroni promotes replica | ✅ |
| Full AZ failure | K8s reschedules to other AZ | ✅ |
| Full region failure | DNS failover to Region B | ⚠️ Semi (manual validation) |
| Cloud provider outage | Platform available, provider ops degraded | N/A |

---

## 19.7 Disaster Recovery Testing

### Chaos Engineering Schedule

| Test | Frequency | Tool | What It Validates |
|------|-----------|------|-------------------|
| Kill random pod | Weekly | Chaos Monkey / Litmus | Pod restart and service continuity |
| Kill database primary | Monthly | Manual | Patroni failover, zero data loss |
| Simulate full AZ failure | Quarterly | Litmus / Gremlin | Multi-AZ resilience |
| Vault seal event | Monthly | Manual | Auto-unseal, service recovery |
| Full DR failover | Biannually | Manual | Regional failover procedure |

---

## 19.8 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **RTO/RPO targets** | 30s RTO for most components, 0 RPO with sync replication |
| **PostgreSQL HA** | Patroni-managed streaming replication + pgBackRest backups |
| **Vault HA** | 3-node Raft cluster with KMS auto-unseal |
| **Multi-region** | WAL shipping for DR, DNS failover |
| **Chaos testing** | Regular failure injection validates recovery procedures |

### What's Next

**Chapter 20** — the final chapter — covers **Production Deployment Strategies** to bring everything together.
