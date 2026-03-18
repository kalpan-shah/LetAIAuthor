# Chapter 9: AWS IoT Core Provider

> **Part III — The Provider Abstraction Layer**

The AWS adapter translates the `IoTProvider` interface into AWS SDK calls using **IAM role assumption** — the platform never stores permanent AWS access keys. It assumes a per-tenant IAM role for each operation, receiving temporary credentials that expire in 15 minutes.

---

## 9.1 IAM Role Architecture

```
Platform AWS Account                  Tenant AWS Account
──────────────────                    ──────────────────
IAM Role: iot-platform                IAM Role: iot-platform-tenant-{id}
  (has permission to call             Trust Policy: allows platform role to
   sts:AssumeRole)                    assume it WITH ExternalID = tenant_id
         │                                      │
         └── sts:AssumeRole ──────────────────► │
                                      Temporary creds (15 min TTL)
                                      Used for: CreateThing, RegisterCertificate,
                                                CreatePolicy, AttachPolicy, etc.
```

> **🔒 SECURITY:** The `ExternalID` parameter prevents the "confused deputy" attack. A malicious actor in another AWS account cannot trick the platform into managing resources in your account without knowing the correct tenant ID.

---

## 9.2 Terraform: Tenant IAM Role

```hcl
# deploy/terraform/modules/aws-iot-tenant/main.tf

variable "platform_role_arn" { type = string }
variable "tenant_id"         { type = string }

resource "aws_iam_role" "iot_platform_tenant" {
  name = "iot-platform-tenant-${var.tenant_id}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = var.platform_role_arn }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = { "sts:ExternalId" = var.tenant_id }
      }
    }]
  })
}

resource "aws_iam_role_policy" "iot_management" {
  name = "iot-management"
  role = aws_iam_role.iot_platform_tenant.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "iot:CreateThing",         "iot:DeleteThing",
        "iot:RegisterCertificate", "iot:UpdateCertificate",
        "iot:AttachThingPrincipal","iot:DetachThingPrincipal",
        "iot:CreatePolicy",        "iot:AttachPolicy",
        "iot:DetachPolicy",        "iot:DeletePolicy",
        "iot:DescribeEndpoint",    "iot:CreateTopicRule",
        "iot:DeleteTopicRule",     "iot:UpdateTopicRule",
      ]
      Resource = "*"
    }]
  })
}

output "role_arn" { value = aws_iam_role.iot_platform_tenant.arn }
```

---

## 9.3 AWS Provider Implementation

```go
// internal/providers/aws/provider.go
package aws

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/credentials/stscreds"
	"github.com/aws/aws-sdk-go-v2/service/iot"
	"github.com/aws/aws-sdk-go-v2/service/iot/types"
	"github.com/aws/aws-sdk-go-v2/service/sts"
	"github.com/your-org/iot-platform/internal/policy"
	"github.com/your-org/iot-platform/internal/providers"
)

type Provider struct {
	region    string
	accountID string
	policyEng *policy.Engine
}

func New(region, accountID string) *Provider {
	return &Provider{region: region, accountID: accountID, policyEng: policy.NewEngine()}
}

func (p *Provider) ProviderType() providers.ProviderType { return providers.ProviderAWS }
func (p *Provider) Name() string                         { return "AWS IoT Core" }

// iotClient creates a per-request IoT client using assumed-role credentials.
func (p *Provider) iotClient(ctx context.Context, creds providers.Credentials) (*iot.Client, error) {
	base, err := config.LoadDefaultConfig(ctx, config.WithRegion(creds.AWSRegion))
	if err != nil {
		return nil, err
	}
	stsC := sts.NewFromConfig(base)
	roler := stscreds.NewAssumeRoleProvider(stsC, creds.AWSRoleARN, func(o *stscreds.AssumeRoleOptions) {
		o.ExternalID      = aws.String(creds.AWSExternalID)
		o.RoleSessionName = "iot-platform"
		o.Duration        = 15 * time.Minute
	})
	cfg2, err := config.LoadDefaultConfig(ctx,
		config.WithRegion(creds.AWSRegion),
		config.WithCredentialsProvider(roler),
	)
	if err != nil {
		return nil, err
	}
	return iot.NewFromConfig(cfg2), nil
}

// CreateDevice: Thing → RegisterCertificate → CreatePolicy → Attach → Attach
func (p *Provider) CreateDevice(ctx context.Context, req providers.CreateDeviceReq) (*providers.Device, error) {
	client, err := p.iotClient(ctx, req.Credentials)
	if err != nil {
		return nil, err
	}

	// ① Create IoT Thing
	thing, err := client.CreateThing(ctx, &iot.CreateThingInput{
		ThingName: aws.String(req.DeviceID),
		AttributePayload: &types.AttributePayload{
			Attributes: map[string]string{"tenant_id": req.TenantID},
		},
	})
	if err != nil {
		return nil, fmt.Errorf("CreateThing: %w", err)
	}

	// ② Register the Vault-issued certificate
	cert, err := client.RegisterCertificate(ctx, &iot.RegisterCertificateInput{
		CertificatePem: aws.String(req.CertPEM),
		Status:         types.CertificateStatusActive,
	})
	if err != nil {
		_, _ = client.DeleteThing(ctx, &iot.DeleteThingInput{ThingName: aws.String(req.DeviceID)})
		return nil, fmt.Errorf("RegisterCertificate: %w", err)
	}

	// ③ Generate topic-scoped IoT policy
	policyDoc, _ := json.Marshal(p.policyEng.ToAWSPolicy(req.PolicyRules, p.region, p.accountID))
	policyName := "device-" + req.DeviceID
	_, err = client.CreatePolicy(ctx, &iot.CreatePolicyInput{
		PolicyName:     aws.String(policyName),
		PolicyDocument: aws.String(string(policyDoc)),
	})
	if err != nil {
		return nil, fmt.Errorf("CreatePolicy: %w", err)
	}

	// ④ Attach policy to certificate
	_, err = client.AttachPolicy(ctx, &iot.AttachPolicyInput{
		PolicyName: aws.String(policyName),
		Target:     cert.CertificateArn,
	})
	if err != nil {
		return nil, fmt.Errorf("AttachPolicy: %w", err)
	}

	// ⑤ Attach certificate to Thing
	_, err = client.AttachThingPrincipal(ctx, &iot.AttachThingPrincipalInput{
		ThingName: aws.String(req.DeviceID),
		Principal: cert.CertificateArn,
	})
	if err != nil {
		return nil, fmt.Errorf("AttachThingPrincipal: %w", err)
	}

	return &providers.Device{
		ExternalID: *thing.ThingName,
		ARN:        *thing.ThingArn,
		CreatedAt:  time.Now(),
	}, nil
}
```
