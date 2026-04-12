# Azure Local Management — REST API

The app exposes a small REST API for managing cluster registrations programmatically.
This is useful when clusters are provisioned or deprovisioned by automation tooling
(Terraform, ARM templates, CI/CD pipelines, scripts) without needing a browser login.

---

## Base URL

```
https://azlmgmt.yourdomain.com/api
```

---

## Authentication

All `/api/` routes require an API key. The key is sent in the `Authorization` header:

```
Authorization: ApiKey <your-token>
```

If the header is missing or the token is wrong, the API returns `401 Unauthorized`.
If no API key has been configured yet, it returns `503 Service Unavailable`.

### Generating an API key

1. Sign in as **HciAdmin**
2. Navigate to **Admin → Settings**
3. Scroll to the **API Access** section
4. Click **Generate New API Key**
5. Copy the key immediately — it is only shown once

> The key is stored encrypted in the database. If you lose it, generate a new one.
> Rotating the key invalidates all callers using the old key immediately.

### API access on the Windows Authentication site

The WinAuth site (`azlmgmt-win.yourdomain.com`) has IIS Anonymous Authentication
disabled globally, which blocks API key requests before they reach the app.

**One-time server fix** — run these two commands once in an elevated prompt on the IIS server,
then recycle the WinAuth app pool:

```powershell
$appcmd = "$env:WINDIR\system32\inetsrv\appcmd.exe"
& $appcmd unlock config /section:system.webServer/security/authentication/anonymousAuthentication
& $appcmd unlock config /section:system.webServer/security/authentication/windowsAuthentication
Restart-WebAppPool -Name "AZLManagementWinPool"
```

This unlocks the IIS authentication sections so the `<location path="api">` block in
`web.config` can re-enable Anonymous Auth for the `/api` path only. The API key requirement
is still enforced by the app — anonymous access is not granted without a valid key.

> `Setup-IIS-WinAuth.ps1` performs this unlock automatically on new installations.
> Existing installations set up before v0.9.12 require the manual commands above.

---

## Endpoints

### `GET /api/clusters`

Returns all registered clusters.

**Response** `200 OK`
```json
[
  {
    "name":                  "MY-CLUSTER-01",
    "address":               "MY-CLUSTER-01.domain.local",
    "connectionType":        "HciCluster",
    "credentialSource":      "gMSA",
    "useHttps":              false,
    "skipCertValidation":    true,
    "isDisabled":            false,
    "solutionUpdatesEnabled": true,
    "arcExtensionsEnabled":  true,
    "atcIntentsEnabled":     true,
    "aksEnabled":            true,
    "healthMonitorEnabled":  true,
    "aksApiEndpoint":        "",
    "hasAksApiToken":        false
  }
]
```

---

### `GET /api/clusters/{name}`

Returns a single cluster by name.

**Response** `200 OK` — cluster object (same shape as above; `aksApiToken` is never returned — use `hasAksApiToken: true/false` to check if a token is stored)  
**Response** `404 Not Found` — `{ "error": "Cluster 'name' not found." }`

---

### `POST /api/clusters`

Registers a new cluster.

**Request body**
```json
{
  "name":                  "MY-CLUSTER-02",
  "address":               "MY-CLUSTER-02.domain.local",
  "connectionType":        "HciCluster",
  "credentialSource":      "gMSA",
  "useHttps":              false,
  "skipCertValidation":    true,
  "isDisabled":            false,
  "solutionUpdatesEnabled": true,
  "arcExtensionsEnabled":  true,
  "atcIntentsEnabled":     true,
  "aksEnabled":            true,
  "healthMonitorEnabled":  true,
  "aksApiEndpoint":        "",
  "aksApiToken":           "eyJ..."
}
```

| Field | Required | Default | Notes |
|---|---|---|---|
| `name` | Yes | — | max 256 chars; must be unique |
| `address` | Yes | — | max 512 chars; FQDN or IP |
| `connectionType` | No | `HciCluster` | `HciCluster` or `HyperVHost` |
| `credentialSource` | No | `gMSA` | `gMSA` or `KeyVault` |
| `useHttps` | No | `false` | Use WinRM HTTPS (port 5986) |
| `skipCertValidation` | No | `true` | Skip TLS cert check for HTTPS |
| `isDisabled` | No | `false` | Excludes cluster from picker; stops background polling |
| `solutionUpdatesEnabled` | No | `true` | Set `false` to hide the Solution Updates page for this cluster |
| `arcExtensionsEnabled` | No | `true` | Set `false` to hide Azure Arc extension pages |
| `atcIntentsEnabled` | No | `true` | Set `false` to hide the Network ATC Intents tab |
| `aksEnabled` | No | `true` | Set `false` to hide the AKS page |
| `healthMonitorEnabled` | No | `true` | Set `false` to disable background health polling |
| `aksApiEndpoint` | No | `""` | Kubernetes API server, e.g. `10.10.10.26:6443` |
| `aksApiToken` | No | — | Bearer token for AKS API. **Write-only** — never returned in responses. Omit or set `null` to keep existing value on PUT. |

**Response** `201 Created`
```json
{ "name": "MY-CLUSTER-02", "id": 6 }
```
**Response** `409 Conflict` — cluster with that name already exists

---

### `PUT /api/clusters/{name}`

Updates an existing cluster's connection settings.

**Request body** — same shape as POST. For optional fields: omit or set `null` to keep the existing value (`connectionType`, `credentialSource`, `aksApiEndpoint`, `aksApiToken`). All feature-enabled booleans replace the existing value when provided (omit a field to keep existing).

**Response** `204 No Content` — update applied  
**Response** `404 Not Found` — cluster does not exist

---

### `DELETE /api/clusters/{name}`

Removes a cluster registration. The cluster's cached connection is evicted immediately.

**Response** `204 No Content` — deleted  
**Response** `404 Not Found` — cluster does not exist

---

## PowerShell examples

```powershell
$baseUrl = "https://azlmgmt.yourdomain.com/api"
$headers = @{ Authorization = "ApiKey YOUR_TOKEN_HERE" }

# List all clusters
Invoke-RestMethod -Uri "$baseUrl/clusters" -Headers $headers

# Get one cluster
Invoke-RestMethod -Uri "$baseUrl/clusters/MY-CLUSTER-01" -Headers $headers

# Add a cluster
$body = @{
    name                  = "MY-CLUSTER-02"
    address               = "MY-CLUSTER-02.domain.local"
    connectionType        = "HciCluster"
    credentialSource      = "gMSA"
    useHttps              = $false
    skipCertValidation    = $true
    isDisabled            = $false
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/clusters" -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# Add a cluster in brownfield mode (Solution Updates and ATC not available on this cluster)
$body = @{
    name                  = "MY-CLUSTER-03"
    address               = "MY-CLUSTER-03.domain.local"
    solutionUpdatesEnabled = $false
    atcIntentsEnabled     = $false
    aksEnabled            = $true
    aksApiEndpoint        = "10.10.10.26:6443"
    aksApiToken           = "eyJhbGciOiJSUzI1NiIsImtpZCI6..."
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/clusters" -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# Update a cluster address
$body = @{
    name               = "MY-CLUSTER-02"
    address            = "MY-CLUSTER-02-new.domain.local"
    useHttps           = $false
    skipCertValidation = $true
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/clusters/MY-CLUSTER-02" -Method Put -Headers $headers `
    -Body $body -ContentType "application/json"

# Delete a cluster
Invoke-RestMethod -Uri "$baseUrl/clusters/MY-CLUSTER-02" -Method Delete -Headers $headers
```

## curl examples

```bash
BASE="https://azlocalmgmt.jase.org/api"
KEY="YOUR_TOKEN_HERE"

# List all clusters
curl -s -H "Authorization: ApiKey $KEY" "$BASE/clusters" | jq .

# Add a cluster
curl -s -X POST "$BASE/clusters" \
  -H "Authorization: ApiKey $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "AZ-NUC-CL02",
    "address": "AZ-NUC-CL02.jase.org",
    "connectionType": "HciCluster",
    "credentialSource": "gMSA",
    "useHttps": false,
    "skipCertValidation": true,
    "isDisabled": false,
    "solutionUpdatesEnabled": true,
    "arcExtensionsEnabled": true,
    "atcIntentsEnabled": true,
    "aksEnabled": true,
    "healthMonitorEnabled": true
  }'

# Delete a cluster
curl -s -X DELETE "$BASE/clusters/AZ-NUC-CL02" \
  -H "Authorization: ApiKey $KEY"
```

---

## Audit logging

Every mutating API call (POST, PUT, DELETE) is written to the audit log
(visible at **Admin → Audit Log**) with:

- **User:** `api-key`
- **Action:** `AddCluster`, `UpdateCluster`, or `DeleteCluster`
- **Outcome:** `Success` or `Failure`

Read calls (GET) are not audited.

---

## Error responses

All errors return JSON:

```json
{ "error": "Human-readable description" }
```

| Status | Meaning |
|---|---|
| `400 Bad Request` | Validation failed — check required fields |
| `401 Unauthorized` | Missing or invalid API key |
| `404 Not Found` | Cluster name not found |
| `409 Conflict` | POST: cluster with that name already exists |
| `500 Internal Server Error` | Unexpected failure — check app logs |
| `503 Service Unavailable` | No API key has been configured |

---

## Future: Entra client credentials (Option B)

For enterprise environments where automation tooling already uses Entra identities,
an app-registration-based `client_credentials` grant will be added in a future release.
This will map to the `HciAdmin` policy automatically via an app role, removing the
shared-secret dependency. See BACKLOG-18 Option B.
