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

## Rate Limiting

The API enforces a **fixed-window rate limit of 60 requests per minute** per client IP address. When the limit is exceeded the request is rejected immediately with:

- **Status:** `429 Too Many Requests`
- **Header:** `Retry-After: 60`
- **Body:** `{ "error": "Rate limit exceeded. Please retry after 60 seconds." }`

Clients should honour the `Retry-After` header and not retry before the indicated delay has elapsed. Automated tooling should use exponential back-off if multiple sequential requests are required.

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

## Maintenance window endpoints

Maintenance windows suppress alerting for the specified cluster (or all clusters when `clusterName = "*"`). Windows can be created as one-time or as recurring rules that auto-generate occurrences 90 days ahead.

---

### `GET /api/maintenance/windows`

Returns all active and upcoming one-time and recurring-occurrence windows (windows that have not yet ended and are not deactivated).

**Response** `200 OK`
```json
[
  {
    "id":               12,
    "clusterName":      "AZ-NUC-CL01",
    "startsAt":         "2026-05-16T22:00:00Z",
    "endsAt":           "2026-05-17T00:00:00Z",
    "reason":           "Monthly patching",
    "isCurrentlyActive": false,
    "isRecurring":      true,
    "recurringRuleId":  3
  }
]
```

---

### `GET /api/maintenance/windows/{id}`

Returns a single maintenance window by ID.

**Response** `200 OK` — window object  
**Response** `404 Not Found`

---

### `POST /api/maintenance/windows`

Creates a one-time maintenance window.

**Request body**
```json
{
  "clusterName": "AZ-NUC-CL01",
  "startsAt":    "2026-05-20T22:00:00Z",
  "endsAt":      "2026-05-21T02:00:00Z",
  "reason":      "Emergency patching"
}
```

| Field | Required | Notes |
|---|---|---|
| `clusterName` | Yes | Cluster name or `"*"` for all clusters |
| `startsAt` | Yes | ISO 8601 UTC datetime |
| `endsAt` | No | ISO 8601 UTC datetime. Omit or `null` for indefinite |
| `reason` | No | Free text, max 512 chars |

**Response** `201 Created` — `{ "id": 15 }`  
**Response** `400 Bad Request` — validation error

---

### `DELETE /api/maintenance/windows/{id}`

Deactivates a maintenance window (soft-delete). Does not remove past or recurring-generated windows from history.

**Response** `204 No Content`  
**Response** `404 Not Found`

---

### `GET /api/maintenance/rules`

Returns all active recurring rules.

**Response** `200 OK`
```json
[
  {
    "id":                    3,
    "clusterName":           "*",
    "recurrenceType":        "Weekly",
    "daysOfWeek":            "Friday",
    "weekOfMonth":           null,
    "dayOfMonth":            null,
    "windowStart":           "22:00",
    "windowDurationMinutes": 120,
    "reason":                "Weekly patching",
    "recurrenceSummary":     "Every Friday at 22:00 UTC for 2h"
  }
]
```

---

### `GET /api/maintenance/rules/{id}`

Returns a single recurring rule by ID.

**Response** `200 OK` — rule object  
**Response** `404 Not Found`

---

### `POST /api/maintenance/rules`

Creates a recurring rule and immediately materializes occurrences for the next 90 days.

**Request body**
```json
{
  "clusterName":           "*",
  "recurrenceType":        "Weekly",
  "daysOfWeek":            "Friday",
  "windowStart":           "22:00",
  "windowDurationMinutes": 120,
  "reason":                "Weekly patching"
}
```

| Field | Required | Notes |
|---|---|---|
| `clusterName` | Yes | Cluster name or `"*"` for all clusters |
| `recurrenceType` | Yes | `Daily`, `Weekly`, `MonthlyByDay`, or `MonthlyByOrdinal` |
| `daysOfWeek` | For Weekly / MonthlyByOrdinal | Comma-separated day names: `"Friday"`, `"Monday,Wednesday,Friday"`. Valid names: `Sunday`, `Monday`, `Tuesday`, `Wednesday`, `Thursday`, `Friday`, `Saturday` |
| `weekOfMonth` | For MonthlyByOrdinal | 1=1st, 2=2nd, 3=3rd, 4=4th, 5=last |
| `dayOfMonth` | For MonthlyByDay | 1-31. Months shorter than this day produce no occurrence |
| `windowStart` | Yes | UTC start time in `HH:mm` format, e.g. `"22:00"`, `"09:30"`, `"00:00"` |
| `windowDurationMinutes` | No | Default 120. Max 10080 (1 week) |
| `reason` | No | Free text, max 512 chars |

**Response** `201 Created` — `{ "id": 4 }`  
**Response** `400 Bad Request` — validation error

---

### `DELETE /api/maintenance/rules/{id}`

Deactivates a recurring rule and cancels all future (not-yet-started) occurrences. Currently in-progress and past occurrences are retained for history.

**Response** `204 No Content`  
**Response** `404 Not Found`

---

### Maintenance PowerShell examples

```powershell
$baseUrl = "https://azlmgmt.yourdomain.com/api"
$headers = @{ Authorization = "ApiKey YOUR_TOKEN_HERE" }

# List all active windows
Invoke-RestMethod -Uri "$baseUrl/maintenance/windows" -Headers $headers

# Create a one-time window for a specific cluster
$body = @{
    clusterName = "AZ-NUC-CL01"
    startsAt    = "2026-05-20T22:00:00Z"
    endsAt      = "2026-05-21T02:00:00Z"
    reason      = "Emergency patching"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/maintenance/windows" -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# Create an indefinite global mute (all clusters)
$body = @{
    clusterName = "*"
    startsAt    = (Get-Date).ToUniversalTime().ToString("o")
    reason      = "Incident investigation"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/maintenance/windows" -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# Deactivate a window by ID
Invoke-RestMethod -Uri "$baseUrl/maintenance/windows/15" -Method Delete -Headers $headers

# Create a weekly recurring rule (every Friday at 22:00 UTC, 2 hours, all clusters)
$body = @{
    clusterName           = "*"
    recurrenceType        = "Weekly"
    daysOfWeek            = "Friday"
    windowStart           = "22:00"
    windowDurationMinutes = 120
    reason                = "Weekly patching"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# Create a monthly rule (2nd Tuesday, 23:00 UTC, 3 hours, specific cluster)
$body = @{
    clusterName           = "AZ-NUC-CL01"
    recurrenceType        = "MonthlyByOrdinal"
    daysOfWeek            = "Tuesday"
    weekOfMonth           = 2         # 2nd occurrence
    windowStart           = "23:00"
    windowDurationMinutes = 180
    reason                = "Monthly patching"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# List all active recurring rules
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" -Headers $headers

# Deactivate a recurring rule (cancels all future occurrences)
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules/3" -Method Delete -Headers $headers
```

#### Multi-cluster schedules

The UI groups rules that share the same schedule and reason together as a single "Reoccurring Schedule" entry (even though they are stored as separate rules, one per cluster). To create the same recurring schedule across multiple clusters, POST one rule per cluster using identical `recurrenceType`, `daysOfWeek`, `windowStart`, `windowDurationMinutes`, and `reason` values.

```powershell
$baseUrl = "https://azlmgmt.yourdomain.com/api"
$headers = @{ Authorization = "ApiKey YOUR_TOKEN_HERE" }

# Apply the same weekly schedule to multiple clusters
# The UI will display these as a single grouped schedule entry.
$clusters = @("AZ-NUC-CL01", "AZ-NUC-CL02", "AZ-NUC-CL03")

$scheduleBase = @{
    recurrenceType        = "Weekly"
    daysOfWeek            = "Friday"
    windowStart           = "22:00"
    windowDurationMinutes = 120       # 2 hours
    reason                = "Weekly patching"
}

foreach ($cluster in $clusters) {
    $body = ($scheduleBase + @{ clusterName = $cluster }) | ConvertTo-Json
    $result = Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" -Method Post `
        -Headers $headers -Body $body -ContentType "application/json"
    Write-Host "Created rule $($result.id) for $cluster"
}

# Apply a monthly schedule (1st day of month, 23:00 UTC, 4 hours) to two clusters
$clusters = @("AZ-NUC-CL01", "AZ-NUC-CL02")

foreach ($cluster in $clusters) {
    $body = @{
        clusterName           = $cluster
        recurrenceType        = "MonthlyByDay"
        dayOfMonth            = 1
        windowStart           = "23:00"
        windowDurationMinutes = 240       # 4 hours
        reason                = "Monthly patching"
    } | ConvertTo-Json
    $result = Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" -Method Post `
        -Headers $headers -Body $body -ContentType "application/json"
    Write-Host "Created rule $($result.id) for $cluster"
}

# Deactivate all rules in a multi-cluster schedule group
# First, find the rule IDs by listing rules and matching on reason + schedule
$rules = Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" -Headers $headers
$groupRules = $rules | Where-Object {
    $_.reason -eq "Weekly patching" -and $_.daysOfWeek -eq "Friday"
}

foreach ($rule in $groupRules) {
    Invoke-RestMethod -Uri "$baseUrl/maintenance/rules/$($rule.id)" `
        -Method Delete -Headers $headers
    Write-Host "Deactivated rule $($rule.id) for $($rule.clusterName)"
}
```

> **Tip — use `"*"` to cover all clusters with one rule.** If every cluster in your fleet should share the same schedule, set `clusterName = "*"` and only a single rule is needed. The per-cluster approach above is for when different clusters need to be added or removed from the group independently.


---

## Audit logging

Every mutating API call (POST, PUT, DELETE) is written to the audit log
(visible at **Admin → Audit Log**) with:

- **User:** `api-key`
- **Action:** `AddCluster`, `UpdateCluster`, `DeleteCluster`, `CreateMaintenanceWindow`, `DeactivateMaintenanceWindow`, `CreateRecurringRule`, or `DeactivateRecurringRule`
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
| `429 Too Many Requests` | Rate limit exceeded — retry after 60 seconds (check `Retry-After` header) |
| `500 Internal Server Error` | Unexpected failure — check app logs |
| `503 Service Unavailable` | No API key has been configured |

---

## Future: Entra client credentials (Option B)

For enterprise environments where automation tooling already uses Entra identities,
an app-registration-based `client_credentials` grant will be added in a future release.
This will map to the `HciAdmin` policy automatically via an app role, removing the
shared-secret dependency. See BACKLOG-18 Option B.
