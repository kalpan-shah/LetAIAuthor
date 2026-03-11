# Chapter 15: SRE Practices & Runbooks

> *"Hope is not a strategy."* — Google SRE Handbook

---

## 15.1 Introduction

Site Reliability Engineering (SRE) bridges the gap between development and operations. For an IoT control plane, SRE means defining **what "reliable" means** (SLOs), measuring it (SLIs), budgeting for failures (error budgets), and having clear procedures when things go wrong (runbooks).

---

## 15.2 Service Level Objectives (SLOs)

| Service | SLI | SLO | Measurement |
|---------|-----|-----|-------------|
| **Device Provisioning** | Success rate | 99.9% | `successes / total` over 30 days |
| **Device Provisioning** | Latency (P99) | < 5 seconds | `histogram_quantile(0.99, ...)` |
| **API Availability** | Uptime | 99.95% | Probe checks every 30s |
| **Certificate Issuance** | Success rate | 99.99% | `successes / total` |
| **Audit Log Ingestion** | Completeness | 100% | Compare NATS published vs. DB stored |
| **Provider Reconciliation** | Drift detection | < 1 hour | Time from drift to alert |

### Error Budget Calculation

```
Monthly error budget = 1 - SLO_target

Example for Device Provisioning (99.9% SLO):
  Monthly minutes: 43,200
  Allowed downtime: 43,200 × 0.001 = 43.2 minutes/month
  
  If 20 minutes consumed by an incident:
  Remaining budget: 23.2 minutes
  Budget consumed: 46.3%
```

> 💡 **Error Budget Policy**: When the error budget is exhausted, development velocity shifts to reliability work. No new features ship until the budget recovers. This creates a healthy tension between velocity and reliability.

---

## 15.3 On-Call Rotation

### Escalation Policy

```
Tier 1: Platform On-Call Engineer (paged immediately)
  └─ 15 min to acknowledge
     └─ If no ack → Tier 2: Engineering Lead
        └─ 15 min to acknowledge
           └─ If no ack → Tier 3: VP Engineering
```

### On-Call Handoff Checklist

- [ ] Review open incidents and recent alerts
- [ ] Check error budget status for all SLOs
- [ ] Verify monitoring dashboards are accessible
- [ ] Confirm access to Vault, cloud provider consoles, Kubernetes
- [ ] Review any ongoing deployments or maintenance windows

---

## 15.4 Operational Runbooks

### Runbook: Provider Credentials Expired

**Symptoms**: `ProviderAPIErrors` alert firing, audit logs showing `CredentialsExpiredError`

**Impact**: All device operations for affected tenant(s) will fail.

**Steps**:

```
1. Identify affected tenant(s):
   → Query: iot_provider_auth_failures_total{provider="aws"} by tenant_id
   → Or check audit_logs: action = 'credentials.validation_failed'

2. Validate the credential issue:
   → vault kv get iot-platform/credentials/tenant-{id}-aws
   → Export VAULT_ADDR and VAULT_TOKEN
   → Try assuming the role manually

3. If credentials are truly expired:
   → Contact tenant to provide new credentials
   → vault kv put iot-platform/credentials/tenant-{id}-aws ...
   → Invalidate provider cache: POST /admin/cache/invalidate/{integration_id}

4. If credentials are valid but issue persists:
   → Check provider-side IAM policy changes
   → Check provider service health (AWS Health Dashboard, Azure Status)

5. Verify recovery:
   → Watch iot_provider_auth_failures_total drop to 0
   → Manually provision a test device for the tenant
```

### Runbook: Database Connection Pool Exhaustion

**Symptoms**: API requests timing out, `connection pool exhausted` in logs

**Steps**:

```
1. Check current connections:
   → SELECT count(*) FROM pg_stat_activity WHERE datname = 'iot_platform';
   → SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

2. Identify long-running queries:
   → SELECT pid, age(clock_timestamp(), query_start), query
     FROM pg_stat_activity
     WHERE state = 'active' AND query_start < NOW() - INTERVAL '30 seconds';

3. Kill stuck queries (if safe):
   → SELECT pg_terminate_backend(pid);

4. Increase pool size (temporary):
   → ALTER SYSTEM SET max_connections = 200;
   → SELECT pg_reload_conf();
   → Restart affected application pods to pick up new pool size

5. Root cause analysis:
   → Check for missing indexes (sequential scans on large tables)
   → Check for N+1 query patterns in recent deployments
   → Review connection pool settings vs. instance count formula
```

### Runbook: Certificate Mass Expiry

**Symptoms**: `CertificatesExpiringSoon` alert with high count, tenant reports device disconnections

**Steps**:

```
1. Assess scope:
   → SELECT tenant_id, COUNT(*) as expiring_count
     FROM certificates
     WHERE status = 'active' AND expires_at < NOW() + INTERVAL '7 days'
     GROUP BY tenant_id;

2. For each affected tenant:
   → Trigger batch rotation:
     POST /admin/certificates/batch-rotate
     {tenant_id, dry_run: true}  ← Preview first

   → If preview looks good:
     POST /admin/certificates/batch-rotate
     {tenant_id, dry_run: false, concurrency: 5}

3. Monitor rotation progress:
   → Watch iot_certificates_issued counter
   → Watch iot_certificates_revoked counter
   → Check for rotation failures in audit logs

4. Post-incident:
   → Investigate why expiry detection didn't trigger earlier
   → Adjust warning_days threshold if needed
```

---

## 15.5 Capacity Planning

| Resource | Current Usage | Growth Rate | Action Trigger |
|----------|-------------|-------------|---------------|
| PostgreSQL connections | 45/100 | +5/month | 70% → add read replica |
| Redis memory | 85 MB/128 MB | +10 MB/month | 80% → increase `maxmemory` |
| NATS messages/sec | 50/5000 | +20/month | 60% → add NATS cluster node |
| Vault tokens/hour | 200/5000 | +50/month | 50% → review token TTL |
| Device count (total) | 5,000 | +1,000/month | At 50k → shard by tenant |

### Capacity Planning Queries

```sql
-- Device growth rate (last 30 days)
SELECT
    DATE_TRUNC('day', created_at) as day,
    COUNT(*) as new_devices
FROM devices
WHERE created_at > NOW() - INTERVAL '30 days'
GROUP BY day
ORDER BY day;

-- Tenant size distribution
SELECT
    t.name,
    COUNT(d.id) as device_count,
    t.max_devices as quota,
    ROUND(COUNT(d.id)::numeric / t.max_devices * 100, 1) as usage_pct
FROM tenants t
LEFT JOIN devices d ON t.id = d.tenant_id AND d.status != 'deleted'
GROUP BY t.id, t.name, t.max_devices
ORDER BY device_count DESC;
```

---

## 15.6 Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **SLOs** | Define measurable targets: 99.9% provisioning success, P99 < 5s |
| **Error budgets** | Quantified risk tolerance — exhaustion pauses feature work |
| **Runbooks** | Step-by-step guides for common failures |
| **Capacity planning** | Track growth rates, set action triggers before limits hit |

### What's Next

**Chapter 16** covers **Security Architecture & Threat Modeling** — comprehensive security analysis and defense strategies.
