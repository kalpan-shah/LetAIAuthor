# Chapter 1: Platform Overview & System Architecture

> **Part I — Foundations**

Before a single line of code is written, the team needs a shared mental model of what the system is, what it is not, and why it is structured the way it is. This chapter establishes that model.

---

## 1.1 What Problem Are We Solving?

Consider a company deploying 50,000 temperature sensors across 200 factory sites worldwide. Each sensor needs to: prove its identity to a backend system (**authenticate**), send readings every 30 seconds (**publish telemetry**), and receive configuration updates from operators (**subscribe to commands**).

At 10 devices, this is manageable manually. At 50,000 devices spread across AWS, Azure, and on-premises Mosquitto brokers — belonging to 40 different customer tenants — manual management is impossible. You need a **control plane**: a system that manages everything about device identity and access at scale, without ever touching the actual sensor data.

The **Unified IoT Orchestration Platform** is that control plane. It handles everything except the telemetry itself: provisioning devices, issuing and rotating their certificates, generating the access policies that restrict what each device can publish and subscribe to, and providing a unified API so engineers never need to learn AWS IoT Core, Azure IoT Hub, and Mosquitto separately.

---

## 1.2 What the Platform Is NOT

Understanding the boundary is as important as understanding the scope:

- **Not a telemetry pipeline.** MQTT messages from devices go directly to the IoT provider (AWS, Azure, or Mosquitto). The platform never sees device data. This keeps it out of the critical path — device telemetry cannot be blocked by a platform outage.
- **Not an application layer.** The platform does not store time-series data, run analytics, or trigger business logic. Those are your applications, which consume telemetry from the provider layer.
- **Not an IoT broker.** The platform does not implement MQTT. It configures the brokers that do.

> **📌 NOTE:** The control plane / data plane separation is fundamental. The platform is 100% control plane. It configures the infrastructure that carries data, but it does not carry data itself. This means a platform outage does not immediately affect running devices — they continue publishing telemetry. The impact of a platform outage is that you cannot provision new devices or rotate certificates until it recovers.

---

## 1.3 High-Level System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                      DEVELOPER INTERFACES (Part V)                    │
│   REST API  │  Web Console (React)  │  CLI  │  Webhook Events         │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │  HTTPS / gRPC
┌──────────────────────────────▼─────────────────────────────────────────┐
│                  PLATFORM SERVICES — CONTROL PLANE (Part IV)          │
│                                                                       │
│  Tenant Mgmt  │  Device Lifecycle  │  Certificate Engine             │
│  Policy Engine│  Provisioning WF   │  Telemetry Routing Config       │
│  Audit Logger │  Quota Enforcer    │  Credential Manager             │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │  Provider Interface (Go interface)
┌──────────────────────────────▼─────────────────────────────────────────┐
│                  PROVIDER ABSTRACTION LAYER (Part III)                │
│  ┌──────────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │  Mosquitto Adapter│  │ AWS IoT Adapter │  │ Azure IoT Adapter  │  │
│  └──────────────────┘  └─────────────────┘  └────────────────────┘  │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │
  ┌────────────────────────────┼─────────────────────────────────┐
  │                            │                                 │
  ▼                            ▼                                 ▼
Eclipse Mosquitto         AWS IoT Core                   Azure IoT Hub
(MQTT broker)             (cloud IoT)                    (cloud IoT)
      ↑                        ↑                               ↑
  Devices ──────── Telemetry flows directly ─────── Devices connect
                   (NOT through platform)            here directly

┌──────────────────────────────────────────────────────────────────────────┐
│                  INFRASTRUCTURE (Part II / Part VIII)                  │
│  PostgreSQL (state)  │  Redis (cache)  │  Vault (secrets + PKI)        │
│  NATS JetStream (events) │  Kubernetes (orchestration) │ Prometheus    │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Core Design Principles

### 1.4.1 Provider Agnosticism

The platform's core services never import AWS SDK or Azure SDK packages. Every provider-specific operation (create a device, attach a certificate, generate a policy) is called through a common Go **IoTProvider interface**. Provider adapters implement that interface. Adding a new provider — say, Google Cloud IoT Core or a private EMQX broker — means implementing the interface, not changing business logic.

### 1.4.2 Secrets Never Touch Application Logs or the Database

Private keys, cloud credentials, and database passwords are managed exclusively through **HashiCorp Vault**. The PostgreSQL database stores Vault path references, never secret values. Log statements never include secret material. This is enforced by code review checklist and automated secret-scanning in CI.

### 1.4.3 Tenant Isolation is Structural, Not Layered

Multi-tenancy is built into the data model, the API routing, and the policy engine from day one. It is not an afterthought. A device in Tenant A cannot receive commands addressed to Tenant B because the generated MQTT policy mathematically prevents it — there is no application-level check that could be bypassed.

### 1.4.4 The Control Plane Does Not Touch Telemetry

Device messages — sensor readings, state updates, commands — flow directly between the device and the IoT provider. The platform is never in this path. This means the platform's availability SLO does not directly affect device uptime.

### 1.4.5 Idempotency by Default

Every provisioning operation is designed to be idempotent. Call 'provision device' twice with the same device name: the second call either returns the existing device record cleanly, or fails with a clear duplicate error. This makes automation pipelines safe to retry without manual cleanup.

---

## 1.5 Device Provisioning: Step-by-Step Data Flow

```
Developer / CI system
    │
    │  POST /api/v1/tenants/{id}/devices
    │  Body: { name, description, metadata }
    ▼
API Gateway (Kong / Traefik)
    │  ① Validate JWT — reject if expired or wrong audience
    │  ② Enforce rate limit — 429 if over quota
    │  ③ Route to Device Service
    ▼
Device Service
    │  ④ Decode & validate request body
    │  ⑤ Load tenant — verify active, check device quota
    │  ⑥ Generate UUID for new device
    │  ⑦ Call Certificate Service
    │       └─ Vault PKI: issue X.509 cert with CN = device-{id}.tenant-{id}
    │       └─ Store cert metadata in PostgreSQL, store key bundle in Vault
    │  ⑧ Call Policy Engine
    │       └─ Generate publish/subscribe topic rules for this device+tenant
    │  ⑨ Call IoTProvider (Mosquitto / AWS / Azure)
    │       └─ Register device identity
    │       └─ Attach certificate
    │       └─ Apply generated policy / ACL
    │  ⑩ If provider call fails: revoke certificate, return error
    │  ⑪ Persist device record to PostgreSQL
    │  ⑫ Publish DeviceProvisioned event to NATS
    │  ⑬ Write audit log record
    ▼
201 Created
    { device_id, endpoint, certificate_pem, private_key_pem (one-time!) }

IMPORTANT: private_key_pem is returned ONCE in this response.
           It is not stored in the database. The device must save it immediately.
```

---

## 1.6 Supported Deployment Models

| Deployment Model | What Runs Where |
|---|---|
| **Local Dev (Docker Compose)** | Everything on one laptop: Mosquitto, Vault, PostgreSQL, Redis, NATS, and the platform API. Zero cloud cost, zero cloud accounts required. |
| **Single-Cloud (Kubernetes)** | Platform on Kubernetes (any cloud or bare metal). IoT provider: one of AWS, Azure, or Mosquitto. Simplest production model. |
| **Multi-Cloud (Kubernetes)** | Platform manages tenants across all three providers simultaneously. Each tenant is assigned one provider. Platform runs on any cluster. |
| **On-Premises / Air-Gapped** | Platform + Mosquitto on bare metal or private Kubernetes (k3s). No internet access required. Use Vault with local storage backend. |
