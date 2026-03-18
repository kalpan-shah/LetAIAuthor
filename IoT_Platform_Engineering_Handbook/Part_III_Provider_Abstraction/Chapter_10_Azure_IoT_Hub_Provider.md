# Chapter 10: Azure IoT Hub Provider

> **Part III — The Provider Abstraction Layer**

The Azure adapter uses **Service Principal** authentication. The platform calls the Azure SDK to register device identities in IoT Hub, configure X.509 authentication, and retrieve connection endpoints.

---

## 10.1 Setup: Azure Service Principal

```bash
# Create a service principal with IoT Hub Data Contributor role
az ad sp create-for-rbac \
  --name 'iot-platform-sp' \
  --role 'IoT Hub Data Contributor' \
  --scopes /subscriptions/{SUB_ID}/resourceGroups/{RG}/providers/Microsoft.Devices/IotHubs/{HUB}

# Returns: { appId, password, tenant }
# Store in Vault — never in .env in production:
vault kv put secret/providers/{tenant_id}/azure \
  client_id=<appId> \
  client_secret=<password> \
  tenant_id=<tenant> \
  hub_hostname=<HUB>.azure-devices.net
```

---

## 10.2 Azure Provider Implementation

```go
// internal/providers/azure/provider.go
package azure

import (
	"bytes"
	"context"
	"crypto/sha1"
	"crypto/x509"
	"encoding/json"
	"encoding/pem"
	"fmt"
	"net/http"
	"time"

	"github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
	"github.com/Azure/azure-sdk-for-go/sdk/azidentity"
	"github.com/your-org/iot-platform/internal/providers"
)

type Provider struct{}

func New() *Provider { return &Provider{} }

func (p *Provider) ProviderType() providers.ProviderType { return providers.ProviderAzure }
func (p *Provider) Name() string                         { return "Azure IoT Hub" }

func (p *Provider) getBearerToken(ctx context.Context, creds providers.Credentials) (string, error) {
	cred, err := azidentity.NewClientSecretCredential(
		creds.AzureTenantID,
		creds.AzureClientID,
		creds.AzureClientSecret,
		nil,
	)
	if err != nil {
		return "", fmt.Errorf("azure credential: %w", err)
	}
	token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
		Scopes: []string{"https://iothubs.azure.net/.default"},
	})
	if err != nil {
		return "", fmt.Errorf("getting azure token: %w", err)
	}
	return token.Token, nil
}

// CreateDevice registers a device in Azure IoT Hub using X.509 cert thumbprint.
func (p *Provider) CreateDevice(ctx context.Context, req providers.CreateDeviceReq) (*providers.Device, error) {
	token, err := p.getBearerToken(ctx, req.Credentials)
	if err != nil {
		return nil, err
	}

	thumbprint, err := certThumbprint(req.CertPEM)
	if err != nil {
		return nil, fmt.Errorf("parsing cert: %w", err)
	}

	hub := req.Credentials.AzureHubHostname
	url := fmt.Sprintf("https://%s/devices/%s?api-version=2021-04-12", hub, req.DeviceID)

	payload := map[string]interface{}{
		"deviceId": req.DeviceID,
		"authentication": map[string]interface{}{
			"type": "selfSigned",
			"x509Thumbprint": map[string]string{
				"primaryThumbprint":   thumbprint,
				"secondaryThumbprint": thumbprint,
			},
		},
		"tags": map[string]string{"tenantId": req.TenantID},
	}
	body, _ := json.Marshal(payload)
	httpReq, _ := http.NewRequestWithContext(ctx, http.MethodPut, url, bytes.NewReader(body))
	httpReq.Header.Set("Authorization", "Bearer "+token)
	httpReq.Header.Set("Content-Type", "application/json")

	resp, err := http.DefaultClient.Do(httpReq)
	if err != nil {
		return nil, fmt.Errorf("azure IoT Hub PUT: %w", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 400 {
		return nil, fmt.Errorf("azure IoT Hub HTTP %d", resp.StatusCode)
	}
	return &providers.Device{ExternalID: req.DeviceID, HubName: hub, CreatedAt: time.Now()}, nil
}

// certThumbprint returns the SHA-1 thumbprint Azure uses to identify device certs.
func certThumbprint(certPEM string) (string, error) {
	block, _ := pem.Decode([]byte(certPEM))
	if block == nil {
		return "", fmt.Errorf("invalid PEM")
	}
	cert, err := x509.ParseCertificate(block.Bytes)
	if err != nil {
		return "", err
	}
	sum := sha1.Sum(cert.Raw)
	return fmt.Sprintf("%X", sum), nil
}

func (p *Provider) GetEndpoint(ctx context.Context, creds providers.Credentials) (string, error) {
	return creds.AzureHubHostname, nil
}

func (p *Provider) ValidateCredentials(ctx context.Context, creds providers.Credentials) error {
	_, err := p.getBearerToken(ctx, creds)
	return err
}
```
