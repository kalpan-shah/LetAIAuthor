# Chapter 17: Web Console & CLI Tooling

> **Part V — Developer Interfaces**

The web console provides a React-based management interface for operators. The CLI provides scriptable access to all API operations for automation workflows and GitOps pipelines.

---

## 17.1 Web Console Architecture

```
web/
├── src/
│   ├── api/         # Generated OpenAPI client (openapi-typescript-codegen)
│   ├── auth/        # JWT storage, refresh logic, protected routes
│   ├── components/
│   │   ├── DeviceTable.tsx
│   │   ├── CertStatusBadge.tsx
│   │   └── AuditLogViewer.tsx
│   ├── pages/
│   │   ├── Login.tsx
│   │   ├── TenantDashboard.tsx
│   │   ├── DeviceList.tsx
│   │   ├── DeviceDetail.tsx
│   │   └── AuditLogs.tsx
│   └── main.tsx
├── package.json
└── vite.config.ts
```

---

## 17.2 Device Provisioning React Component

```tsx
// web/src/components/ProvisionDeviceModal.tsx
import { useState } from 'react';
import { api } from '../api/client';

interface Props {
  tenantId: string;
  onSuccess: (result: ProvisionResult) => void;
}

export function ProvisionDeviceModal({ tenantId, onSuccess }: Props) {
  const [name, setName]       = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError]     = useState<string>();
  const [result, setResult]   = useState<ProvisionResult>();

  const handleProvision = async () => {
    setLoading(true); setError(undefined);
    try {
      const res = await api.devices.provision(tenantId, { name });
      setResult(res);
      onSuccess(res);
    } catch (e: any) {
      setError(e.message ?? 'Provisioning failed');
    } finally { setLoading(false); }
  };

  if (result) return (
    <div className="cert-delivery">
      <h3>Device Provisioned</h3>
      <p className="warning">
        This is the ONLY time the private key is shown.
        Download it now and store it securely on the device.
      </p>
      <pre>{result.certificate.private_key_pem}</pre>
      <button onClick={() => downloadPEM(result.certificate.private_key_pem, 'private.key')}>
        Download private.key
      </button>
    </div>
  );

  return (
    <div>
      <input value={name} onChange={e => setName(e.target.value)} placeholder="Device name" />
      {error && <p className="error">{error}</p>}
      <button onClick={handleProvision} disabled={loading || !name}>
        {loading ? 'Provisioning...' : 'Provision Device'}
      </button>
    </div>
  );
}
```

---

## 17.3 CLI Usage

```bash
# Provision a device and save the private key
iot-platform devices provision \
  --tenant t-abc123 \
  --name my-sensor-01 \
  --output json \
  | jq -r '.certificate.private_key_pem' > device.key

# List active devices
iot-platform devices list --tenant t-abc123 --status active

# Rotate a certificate
iot-platform devices rotate-cert \
  --tenant t-abc123 \
  --device d-xyz789

# View recent audit logs
iot-platform audit list \
  --tenant t-abc123 \
  --action device.provision \
  --since 24h

# Validate a provider integration
iot-platform integrations validate \
  --tenant t-abc123 \
  --provider aws
```

---

## 17.4 CLI Implementation (Cobra)

```go
// cli/cmd/devices.go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
	"github.com/your-org/iot-platform/cli/api"
)

var provisionCmd = &cobra.Command{
	Use:   "provision",
	Short: "Provision a new device",
	RunE: func(cmd *cobra.Command, args []string) error {
		tenantID, _ := cmd.Flags().GetString("tenant")
		name,     _ := cmd.Flags().GetString("name")

		client := api.NewClient(globalConfig())
		result, err := client.ProvisionDevice(cmd.Context(), tenantID, api.ProvisionReq{Name: name})
		if err != nil {
			return fmt.Errorf("provisioning failed: %w", err)
		}

		fmt.Fprintf(cmd.ErrOrStderr(), "\nSave the private key — it will not be shown again.\n\n")
		outputJSON(result)
		return nil
	},
}

func init() {
	devicesCmd.AddCommand(provisionCmd)
	provisionCmd.Flags().String("tenant", "", "Tenant ID (required)")
	provisionCmd.Flags().String("name",   "", "Device name (required)")
	provisionCmd.MarkFlagRequired("tenant")
	provisionCmd.MarkFlagRequired("name")
}
```
