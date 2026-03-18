# Chapter 25: Infrastructure as Code with Terraform / OpenTofu

> **Part VIII — DevOps**

All cloud infrastructure is managed as code. This prevents configuration drift, enables environment parity (staging mirrors production), and provides an audit trail of every infrastructure change via Git history.

> **Note:** All Terraform shown here is compatible with OpenTofu 1.7+ — the CNCF-hosted open-source fork of Terraform. Use either interchangeably.

---

## 25.1 Module Structure

```
deploy/terraform/
├── modules/
│   ├── aws-iot-tenant/           # Per-tenant IAM role for AWS IoT Core
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── azure-iot-tenant/         # Per-tenant SP for Azure IoT Hub
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── vault-config/             # Vault PKI, policies, auth methods
│   │   ├── pki.tf
│   │   ├── policies.tf
│   │   ├── auth.tf
│   │   └── variables.tf
│   ├── k8s-platform/             # Namespace, ServiceAccounts, RBAC
│   │   ├── main.tf
│   │   └── variables.tf
│   └── observability/            # Prometheus, Grafana, Loki, Jaeger
│       ├── main.tf
│       └── variables.tf
└── environments/
    ├── local/                    # k3s or kind; minimal config
    │   └── main.tf
    ├── staging/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── backend.tf            # Remote state: S3 or Azure Blob
    └── production/
        ├── main.tf
        ├── variables.tf
        └── backend.tf
```

---

## 25.2 Vault Configuration Module

```hcl
# deploy/terraform/modules/vault-config/pki.tf

variable "vault_addr" { type = string }

resource "vault_mount" "pki" {
  path                      = "pki"
  type                      = "pki"
  default_lease_ttl_seconds = 86400         # 1 day default
  max_lease_ttl_seconds     = 315360000     # 10 years (root CA lifetime)
}

# Root CA — generated internally, key never leaves Vault
resource "vault_pki_secret_backend_root_cert" "root" {
  backend     = vault_mount.pki.path
  type        = "internal"
  common_name = "IoT Platform Root CA"
  ttl         = "315360000"
  key_type    = "ec"
  key_bits    = 384
  organization = "IoT Platform"
  country      = "US"
}

# CRL and CA distribution URLs
resource "vault_pki_secret_backend_config_urls" "urls" {
  backend                 = vault_mount.pki.path
  issuing_certificates    = ["${var.vault_addr}/v1/pki/ca"]
  crl_distribution_points = ["${var.vault_addr}/v1/pki/crl"]
  ocsp_servers            = ["${var.vault_addr}/v1/pki/ocsp"]
}

# Device certificate role: EC P-256, 1yr max, client auth only
resource "vault_pki_secret_backend_role" "device_cert" {
  backend           = vault_mount.pki.path
  name              = "device-cert"
  allowed_domains   = ["devices.iotplatform.local"]
  allow_subdomains  = true
  max_ttl           = "8760h"   # 1 year
  key_type          = "ec"
  key_bits          = 256
  server_flag       = false
  client_flag       = true
  require_cn        = true
  key_usage         = ["DigitalSignature"]
  ext_key_usage     = ["ClientAuth"]
}
```

```hcl
# deploy/terraform/modules/vault-config/policies.tf

resource "vault_policy" "iot_platform_api" {
  name = "iot-platform-api"
  policy = <<-EOT
    path "pki/issue/device-cert"       { capabilities = ["create", "update"] }
    path "pki/revoke"                  { capabilities = ["create", "update"] }
    path "secret/data/providers/+/*"   { capabilities = ["create","read","update","delete"] }
    path "secret/data/certs/+/*"       { capabilities = ["create","read","delete"] }
    path "database/creds/iot-platform" { capabilities = ["read"] }
  EOT
}
```

```hcl
# deploy/terraform/modules/vault-config/auth.tf

# Kubernetes auth method for production pod authentication
resource "vault_auth_backend" "kubernetes" {
  type = "kubernetes"
}

resource "vault_kubernetes_auth_backend_config" "config" {
  backend     = vault_auth_backend.kubernetes.path
  kubernetes_host = var.kubernetes_api_host
}

resource "vault_kubernetes_auth_backend_role" "api" {
  backend                          = vault_auth_backend.kubernetes.path
  role_name                        = "iot-platform-api"
  bound_service_account_names      = ["iot-platform-api"]
  bound_service_account_namespaces = ["iot-platform"]
  token_policies                   = [vault_policy.iot_platform_api.name]
  token_ttl                        = 3600  # 1 hour
}
```

---

## 25.3 AWS IoT Tenant Module

```hcl
# deploy/terraform/modules/aws-iot-tenant/main.tf

variable "platform_role_arn" { type = string }
variable "tenant_id"         { type = string }
variable "region"            { type = string }

resource "aws_iam_role" "iot_platform_tenant" {
  name = "iot-platform-tenant-${var.tenant_id}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = var.platform_role_arn }
      Action    = "sts:AssumeRole"
      Condition = {
        # ExternalID prevents confused-deputy attacks
        StringEquals = { "sts:ExternalId" = var.tenant_id }
      }
    }]
  })

  tags = {
    ManagedBy = "terraform"
    Platform  = "iot-platform"
    TenantID  = var.tenant_id
  }
}

resource "aws_iam_role_policy" "iot_management" {
  name = "iot-management"
  role = aws_iam_role.iot_platform_tenant.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = [
        "iot:CreateThing", "iot:DeleteThing", "iot:DescribeThing",
        "iot:RegisterCertificate", "iot:UpdateCertificate",
        "iot:AttachThingPrincipal", "iot:DetachThingPrincipal",
        "iot:CreatePolicy", "iot:AttachPolicy", "iot:DetachPolicy", "iot:DeletePolicy",
        "iot:DescribeEndpoint",
        "iot:CreateTopicRule", "iot:UpdateTopicRule", "iot:DeleteTopicRule",
      ]
      Resource = "*"
    }]
  })
}

output "role_arn"     { value = aws_iam_role.iot_platform_tenant.arn }
output "external_id"  { value = var.tenant_id }
```

---

## 25.4 Production Environment

```hcl
# deploy/terraform/environments/production/main.tf

terraform {
  required_version = ">= 1.7"
  required_providers {
    vault = { source = "hashicorp/vault", version = "~> 4.0" }
    aws   = { source = "hashicorp/aws",   version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "iot-platform-tfstate"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "iot-platform-tflock"
  }
}

module "vault_config" {
  source     = "../../modules/vault-config"
  vault_addr = var.vault_addr
  kubernetes_api_host = var.kubernetes_api_host
}

# Call this module once per new tenant that chooses AWS
module "tenant_abc_aws" {
  source            = "../../modules/aws-iot-tenant"
  tenant_id         = "t-abc123"
  platform_role_arn = var.platform_aws_role_arn
  region            = "us-east-1"
}
```

---

## 25.5 Terraform Workflow

```bash
# Initialize with remote state
terraform -chdir=deploy/terraform/environments/production init

# Preview changes before applying
terraform -chdir=deploy/terraform/environments/production plan \
  -var-file=production.tfvars

# Apply (requires MFA in production)
terraform -chdir=deploy/terraform/environments/production apply \
  -var-file=production.tfvars \
  -auto-approve  # Remove -auto-approve in production!

# Add a new tenant's AWS infrastructure
terraform apply \
  -target=module.tenant_xyz_aws \
  -var-file=production.tfvars

# Destroy a tenant's infrastructure (use with extreme caution)
terraform destroy \
  -target=module.tenant_xyz_aws \
  -var-file=production.tfvars
```
