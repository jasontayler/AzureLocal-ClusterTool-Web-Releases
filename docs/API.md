# Azure Local Management — REST API

The app exposes a small REST API for managing cluster registrations programmatically.
This is useful when clusters are provisioned or deprovisioned by automation tooling
(Terraform, ARM templates, CI/CD pipelines, scripts) without needing a browser login.

---

## Base URL

```text
https://azlmgmt.yourdomain.com/api/v1
```

> **Versioning:** `/api/v1/...` is the primary, supported base URL. The unversioned
> `/api/...` routes (e.g. `/api/clusters`) still work identically and are kept as
> deprecated aliases for backward compatibility — they will not be removed earlier
> than one major version after 1.0.0. New integrations should use `/api/v1/...`.

---

## Authentication

All `/api/` routes require an API key. The key is sent in the `Authorization` header:

```text
Authorization: ApiKey <your-token>
```

If the header is missing or the token is wrong, the API returns `401 Unauthorized`.
If no API key has been configured yet, it returns `503 Service Unavailable`.

Privileged mutation endpoints require an API key with **Admin** access level.
This includes cluster registration updates, maintenance window/rule mutations,
and specific Ops control actions. These endpoints use the same header format:

```text
Authorization: ApiKey <your-admin-token>
```

If a privileged mutation endpoint is called with a Read-level key, the API
returns `403 Forbidden`.

### Generating an API key

1. Sign in as **HciAdmin**
2. Navigate to **Admin → Settings**
3. Scroll to the **API Access** section
4. Enter a **Name**, choose an **Access Level** and optional **RBAC Role**, then click **Generate Key**
5. Copy the key immediately — it is only shown once

> The plaintext token is never stored and cannot be recovered later. The app stores
> only a hash of each key plus metadata such as name, scope, and timestamps. Reusing
> the same name rotates that key in place and invalidates the old token immediately.

### API key access model

API keys use a single permission selector that resolves to one of three effective modes:

- **Admin** = privileged override for Ops endpoints
- **Read** = standard non-admin access
- **Custom RBAC role name** = limited automation scoped to that role

The role name itself is the permission scope for limited automation; it is not prefixed with `Read -`. A custom role can grant specific operations such as VM start/stop or cluster health actions, without requiring a separate `Read` label in the selector.

This keeps the UX simple:

- `Admin` is the override mode
- `Read` is the default non-admin mode
- `<custom-role-name>` is the limited-scope mode, with `Read` implied by the fact that it is a non-admin custom role

### API access on the Windows Authentication site

The WinAuth site (`azlmgmt-win.yourdomain.com`) has IIS Anonymous Authentication
disabled globally, which blocks API key requests before they reach the app.

**One-time server fix** — run these two commands once in an elevated prompt
on the IIS server, then recycle the WinAuth app pool:

```powershell
$appcmd = "$env:WINDIR\system32\inetsrv\appcmd.exe"
& $appcmd unlock config /section:system.webServer/security/authentication/anonymousAuthentication
& $appcmd unlock config /section:system.webServer/security/authentication/windowsAuthentication
Restart-WebAppPool -Name "AZLManagementWinPool"
```

This unlocks the IIS authentication sections so the
`<location path="api">` block in
`web.config` can re-enable Anonymous Auth for the `/api` path only.
The API key requirement
is still enforced by the app — anonymous access is not granted without a valid key.

> `Setup-IIS-WinAuth.ps1` performs this unlock automatically on new installations.
> Existing installations set up before v0.9.12 require the manual commands above.

---

## Rate Limiting

The API enforces a **fixed-window rate limit of 60 requests per minute** per
API key for authenticated requests. Before authentication (or when key identity
is unavailable), rate limiting falls back to client IP.

When the limit is exceeded, the request is rejected immediately with:

- **Status:** `429 Too Many Requests`
- **Header:** `Retry-After: 60`
- **Body:** `{ "error": "Rate limit exceeded. Please retry after 60 seconds." }`

Clients should honor the `Retry-After` header and not retry before the
indicated delay has elapsed. Automated tooling should use exponential back-off
if multiple sequential requests are required.

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
    "healthMonitorEnabled":  true
  }
]
```

> **Note:** AKS/Arc-connected Kubernetes access uses an Azure AD
> Proof-of-Possession (PoP)
> token flow via Arc and does not require any per-cluster API endpoint or token
> configuration. Older releases had `aksApiEndpoint`/`aksApiToken` fields for a
> now-retired manual token flow — these are no longer part of the API.

---

### `GET /api/clusters/{name}`

Returns a single cluster by name.

**Response** `200 OK` — cluster object (same shape as above)  
**Response** `404 Not Found` — `{ "error": "Cluster 'name' not found." }`

---

### `POST /api/clusters`

Registers a new cluster.

Requires an **Admin** API key.

#### Request body (create cluster)

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
  "healthMonitorEnabled":  true
}
```

| Field | Required | Default | Notes |
| --- | --- | --- | --- |
| `name` | Yes | — | max 256 chars; must be unique |
| `address` | Yes | — | max 512 chars; DNS host or IP, optional `:port` |
| `connectionType` | No | `HciCluster` | `HciCluster` or `HyperVHost` |
| `credentialSource` | No | `gMSA` | `gMSA` or `KeyVault` |
| `useHttps` | No | `false` | Use WinRM HTTPS (port 5986) |
| `skipCertValidation` | No | `true` | Skip TLS cert check for HTTPS |
| `isDisabled` | No | `false` | Excludes cluster from picker; stops polling |
| `solutionUpdatesEnabled` | No | `true` | Hide page when `false` |
| `arcExtensionsEnabled` | No | `true` | Hide page when `false` |
| `atcIntentsEnabled` | No | `true` | Set `false` to hide ATC Intents tab |
| `aksEnabled` | No | `true` | Set `false` to hide the AKS page |
| `healthMonitorEnabled` | No | `true` | Set `false` to disable health polling |

Address validation notes:

- URL formats are rejected (for example `https://cluster.local`).
- Supported forms are `host`, `host:port`, `IP`, or `[IPv6]:port`.
- Port, when provided, must be `1-65535`.
- Loopback/unspecified/multicast IPs are rejected.

**Response** `201 Created`

```json
{ "name": "MY-CLUSTER-02", "id": 6 }
```

**Response** `409 Conflict` — cluster with that name already exists

---

### `PUT /api/clusters/{name}`

Updates an existing cluster's connection settings.

Requires an **Admin** API key.

#### Request body (update cluster)

Same shape as POST. For optional fields: omit or set `null` to keep the
existing value (`connectionType`, `credentialSource`).
All feature-enabled booleans replace the existing value when provided.
Omit a field to keep the existing value.

**Response** `204 No Content` — update applied  
**Response** `404 Not Found` — cluster does not exist

---

### `DELETE /api/clusters/{name}`

Removes a cluster registration. The cluster's cached connection is evicted immediately.

Requires an **Admin** API key.

**Response** `204 No Content` — deleted  
**Response** `404 Not Found` — cluster does not exist

---

## PowerShell examples

```powershell
$baseUrl = "https://azlmgmt.yourdomain.com/api/v1"
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

# Add a cluster in brownfield mode
# (Solution Updates and ATC not available on this cluster)
$body = @{
    name                  = "MY-CLUSTER-03"
    address               = "MY-CLUSTER-03.domain.local"
    solutionUpdatesEnabled = $false
    atcIntentsEnabled     = $false
    aksEnabled            = $true
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
Invoke-RestMethod -Uri "$baseUrl/clusters/MY-CLUSTER-02" `
  -Method Put -Headers $headers `
    -Body $body -ContentType "application/json"

# Delete a cluster
Invoke-RestMethod -Uri "$baseUrl/clusters/MY-CLUSTER-02" `
  -Method Delete -Headers $headers
```

## curl examples

```bash
BASE="https://azlocalmgmt.jase.org/api/v1"
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

Maintenance windows suppress alerting for the specified cluster
(or all clusters when `clusterName = "*"`).
Windows can be created as one-time or as recurring rules that auto-generate
occurrences 90 days ahead.

---

### `GET /api/maintenance/windows`

Returns all active and upcoming one-time and recurring-occurrence windows
(windows that have not yet ended and are not deactivated).

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

Requires an **Admin** API key.

#### Request body (create maintenance window)

```json
{
  "clusterName": "AZ-NUC-CL01",
  "startsAt":    "2026-05-20T22:00:00Z",
  "endsAt":      "2026-05-21T02:00:00Z",
  "reason":      "Emergency patching"
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `clusterName` | Yes | Cluster name or `"*"` for all clusters |
| `startsAt` | Yes | ISO 8601 UTC datetime |
| `endsAt` | No | ISO 8601 UTC datetime. Omit or `null` for indefinite |
| `reason` | No | Free text, max 512 chars |

**Response** `201 Created` — `{ "id": 15 }`  
**Response** `400 Bad Request` — validation error

---

### `DELETE /api/maintenance/windows/{id}`

Hard-deletes a maintenance window.

Requires an **Admin** API key.

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

Creates a recurring rule and immediately materializes occurrences for the
next 90 days.

Requires an **Admin** API key.

#### Request body (create recurring rule)

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
| --- | --- | --- |
| `clusterName` | Yes | Cluster name or `"*"` for all clusters |
| `recurrenceType` | Yes | `Daily`/`Weekly`/`MonthlyByDay`/`MonthlyByOrdinal` |
| `daysOfWeek` | Weekly/MonthlyByOrdinal | Valid: `Sunday`..`Saturday` |
| `weekOfMonth` | For MonthlyByOrdinal | 1=1st, 2=2nd, 3=3rd, 4=4th, 5=last |
| `dayOfMonth` | For MonthlyByDay | 1-31; shorter months produce no occurrence |
| `windowStart` | Yes | UTC time `HH:mm`, e.g. `"22:00"`, `"09:30"`, `"00:00"` |
| `windowDurationMinutes` | No | Default 120. Max 10080 (1 week) |
| `reason` | No | Free text, max 512 chars |

**Response** `201 Created` — `{ "id": 4 }`  
**Response** `400 Bad Request` — validation error

---

### `DELETE /api/maintenance/rules/{id}`

Hard-deletes a recurring rule and removes all future (not-yet-started) occurrences.
Currently in-progress and past occurrences are retained for history.

Requires an **Admin** API key.

**Response** `204 No Content`  
**Response** `404 Not Found`

---

### Maintenance PowerShell examples

```powershell
$baseUrl = "https://azlmgmt.yourdomain.com/api/v1"
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
Invoke-RestMethod -Uri "$baseUrl/maintenance/windows" `
  -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# Create an indefinite global mute (all clusters)
$body = @{
    clusterName = "*"
    startsAt    = (Get-Date).ToUniversalTime().ToString("o")
    reason      = "Incident investigation"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/maintenance/windows" `
  -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# Delete a window by ID
Invoke-RestMethod -Uri "$baseUrl/maintenance/windows/15" `
  -Method Delete -Headers $headers

# Create a weekly recurring rule (every Friday at 22:00 UTC, 2 hours, all clusters)
$body = @{
    clusterName           = "*"
    recurrenceType        = "Weekly"
    daysOfWeek            = "Friday"
    windowStart           = "22:00"
    windowDurationMinutes = 120
    reason                = "Weekly patching"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" `
  -Method Post -Headers $headers `
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
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" `
  -Method Post -Headers $headers `
    -Body $body -ContentType "application/json"

# List all active recurring rules
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules" -Headers $headers

# Delete a recurring rule (removes future occurrences)
Invoke-RestMethod -Uri "$baseUrl/maintenance/rules/3" -Method Delete -Headers $headers
```

#### Multi-cluster schedules

The UI groups rules that share the same schedule and reason as a single
"Reoccurring Schedule" entry, even though they are stored as separate rules,
one per cluster. To create the same recurring schedule across multiple
clusters, POST one rule per cluster using identical `recurrenceType`,
`daysOfWeek`, `windowStart`, `windowDurationMinutes`, and `reason` values.

```powershell
$baseUrl = "https://azlmgmt.yourdomain.com/api/v1"
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

> **Tip - use `"*"` to cover all clusters with one rule.**
> If every cluster in your fleet should share the same schedule,
> set `clusterName = "*"` and only a single rule is needed.
> The per-cluster approach above is for when different clusters need to be
> added or removed from the group independently.

---

## Ops API endpoints

The Ops API is intended for AI/SRE automation and operational tooling. The first
implementation slice exposes cluster collector health, recent PowerShell call
telemetry, and a pair of guarded background-collector actions.

### `GET /api/ops/clusters/{name}/health`

Returns the current background-collector health row for a cluster, plus a bounded
set of recent PowerShell/WinRM calls recorded for that cluster.

#### Query parameters (cluster health)

- `recentCalls` - optional, default `10`, min `1`, max `50`

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "lastSuccessAt": "2026-10-07T14:12:00Z",
  "lastFailureAt": null,
  "lastError": null,
  "lastPollDurationMs": 4820,
  "lastPollCallCount": 19,
  "consecutiveFails": 0,
  "isCircuitOpen": false,
  "circuitOpenUntil": null,
  "status": "OK",
  "statusCss": "state-green",
  "recentCalls": [
    {
      "timestamp": "2026-10-07T14:12:00Z",
      "cluster": "MY-CLUSTER-01",
      "label": "GetClusterInfoAsync",
      "durationMs": 381,
      "rowCount": 1,
      "hadErrors": false,
      "firstError": null,
      "outcome": "Succeeded"
    }
  ]
}
```

**Response** `404 Not Found` - `{ "error": "Cluster 'name' not found." }`

### `GET /api/ops/clusters/{name}/actions`

Returns cluster-level action availability for the caller. Use this endpoint to
discover which cluster mutation routes are currently allowed and why an action
may be blocked.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "status": "OK",
  "statusCss": "state-green",
  "isCircuitOpen": false,
  "circuitOpenUntil": null,
  "actions": [
    {
      "action": "ForcePoll",
      "routeSegment": "force-poll",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/force-poll",
      "method": "POST",
      "requiresAdmin": true,
      "allowed": false,
      "reason": "Admin API key required for this endpoint."
    },
    {
      "action": "ResetCircuit",
      "routeSegment": "reset-circuit",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/reset-circuit",
      "method": "POST",
      "requiresAdmin": true,
      "allowed": false,
      "reason": "Admin API key required for this endpoint."
    }
  ],
  "allowedActions": []
}
```

**Response** `404 Not Found` - cluster not found

### `GET /api/ops/clusters/{name}/nodes`

Returns the cached node snapshot for the specified cluster.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "nodes": [
    {
      "name": "NODE-01",
      "status": "Up",
      "state": "Up",
      "vmCount": 12
    }
  ]
}
```

### `GET /api/ops/clusters/{name}/nodes/{nodeName}/actions`

Returns node-level action availability for the caller. This endpoint mirrors the
same state gating used by the UI:

- `Up` nodes allow `Pause` and `Drain`
- `Paused` nodes allow `Resume` and `Failback`

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "node": "NODE-02",
  "state": "Paused",
  "drainStatus": "InProgress",
  "actions": [
    {
      "action": "Pause",
      "routeSegment": "pause",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/nodes/NODE-02/pause",
      "method": "POST",
      "requiresAdmin": true,
      "allowed": false,
      "reason": "Node is already paused."
    },
    {
      "action": "Resume",
      "routeSegment": "resume",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/nodes/NODE-02/resume",
      "method": "POST",
      "requiresAdmin": true,
      "allowed": true,
      "reason": null
    }
  ],
  "allowedActions": ["Resume", "Failback"]
}
```

**Response** `404 Not Found` - node or cluster not found

### `GET /api/ops/clusters/{name}/cluster-info`

Returns the cached cluster information snapshot for the specified cluster.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "clusterInfo": {
    "name": "MY-CLUSTER-01",
    "domain": "contoso.local",
    "totalNodes": 4,
    "onlineNodes": 4,
    "quorumStatus": "Online",
    "criticalFaultCount": 0,
    "warningFaultCount": 0
  }
}
```

### `GET /api/ops/clusters/{name}/roles`

Returns the cached cluster role snapshot for the specified cluster.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "roles": [
    {
      "name": "Available Storage",
      "ownerNode": "NODE-01",
      "state": "Online",
      "resourceType": "Cluster Group"
    }
  ]
}
```

### `GET /api/ops/clusters/{name}/roles/{roleName}/actions`

Returns role-level action availability for the caller. This endpoint mirrors the
same state gating used by the UI:

- `Offline` and `Failed` roles allow `Start`
- `Online` roles allow `Stop` and `Move`

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "role": "Available Storage",
  "state": "Online",
  "ownerNode": "NODE-01",
  "actions": [
    {
      "action": "Start",
      "routeSegment": "start",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/roles/Available%20Storage/start",
      "method": "POST",
      "requiresAdmin": true,
      "allowed": false,
      "reason": "Role is already online."
    },
    {
      "action": "Move",
      "routeSegment": "move",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/roles/Available%20Storage/move?targetNode={nodeName}",
      "method": "POST",
      "requiresAdmin": true,
      "allowed": true,
      "reason": null
    }
  ],
  "allowedActions": ["Stop", "Move"]
}
```

**Response** `404 Not Found` - role or cluster not found

### `GET /api/ops/clusters/{name}/vms/summary`

Returns a summary of VM state counts from the cached VM snapshot for the cluster.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "summary": {
    "total": 24,
    "running": 20,
    "off": 3,
    "paused": 1,
    "other": 0,
    "unclustered": 0,
    "hasDda": 2,
    "hosts": 4
  }
}
```

### `GET /api/ops/clusters/{name}/vms`

Returns the cached VM list for the specified cluster (name + state + host metadata).
Use this endpoint to discover valid VM names before calling VM action endpoints.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "vms": [
    {
      "name": "VM-01",
      "state": "Running",
      "host": "NODE-01",
      "cpuUsage": 22,
      "memoryMB": 4096,
      "memoryDisplay": "4.0 GB",
      "uptime": "0d 01:15:30",
      "isUnclustered": false,
      "hasDda": true
    },
    {
      "name": "VM-02",
      "state": "Off",
      "host": "NODE-01",
      "cpuUsage": 0,
      "memoryMB": 2048,
      "memoryDisplay": "2.0 GB",
      "uptime": "",
      "isUnclustered": false,
      "hasDda": false
    }
  ]
}
```

### `GET /api/ops/clusters/{name}/vms/{vmName}`

Returns a single VM (from the cached VM snapshot) for the specified cluster.
Use this endpoint to check current state before invoking a VM action.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "vm": {
    "name": "VM-01",
    "state": "Running",
    "host": "NODE-01",
    "cpuUsage": 22,
    "memoryMB": 4096,
    "memoryDisplay": "4.0 GB",
    "uptime": "0d 01:15:30",
    "isUnclustered": false,
    "hasDda": true
  }
}
```

**Response** `404 Not Found` - VM name not found in that cluster snapshot

### `GET /api/ops/clusters/{name}/vms/{vmName}/actions`

Returns action availability for a VM based on its current cached state. Use this
to decide which VM mutation routes are valid before calling them.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "collectedAt": "2026-10-07T14:12:00Z",
  "isStale": false,
  "isLive": false,
  "isPendingFirstPoll": false,
  "vm": "VM-01",
  "state": "Running",
  "actions": [
    {
      "action": "Start",
      "routeSegment": "start",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/vms/VM-01/start",
      "method": "POST",
      "allowed": false,
      "reason": "VM is already running."
    },
    {
      "action": "Stop",
      "routeSegment": "stop",
      "path": "/api/v1/ops/clusters/MY-CLUSTER-01/vms/VM-01/stop",
      "method": "POST",
      "allowed": true,
      "reason": null
    }
  ],
  "allowedActions": ["Stop", "Restart", "Suspend", "ForceStop"]
}
```

**Response** `404 Not Found` - VM or cluster not found

### `GET /api/ops/ps-calls`

Returns recent PowerShell/WinRM call records from the in-memory diagnostics ring
buffer.

#### Query parameters

- `cluster` - optional cluster-name filter
- `limit` - optional, default `50`, min `1`, max `200`
- `outcome` - optional: `Succeeded`, `CompletedWithErrors`, or `Failed`

**Response** `200 OK` - array of PowerShell call objects
(same shape as `recentCalls` above)

### `GET /api/ops/clusters/{name}/status`

Returns the current observed cluster state. Optional `targetState` query parameter
allows clients to detect convergence for a wait loop without forcing a refresh.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "resourceType": "Cluster",
  "resourceName": "MY-CLUSTER-01",
  "state": "Online",
  "stateDisplay": "Online",
  "isConverged": true,
  "requestedState": "Online",
  "observedAtUtc": "2026-10-09T12:34:56Z"
}
```

### `POST /api/ops/clusters/{name}/wait?targetState=Online&timeoutSeconds=60&pollIntervalMs=2500`

Thin convenience wrapper around the cluster status check. Polls until the cluster
state matches `targetState`, then returns a single success result. If the timeout is
reached without convergence, it returns `status: "TimedOut"` and `converged: false`.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "resourceType": "Cluster",
  "resourceName": "MY-CLUSTER-01",
  "targetState": "Online",
  "finalState": "Online",
  "status": "Succeeded",
  "converged": true,
  "elapsedMs": 4380,
  "observedAtUtc": "2026-10-09T12:34:56Z"
}
```

### `POST /api/ops/clusters/{name}/force-poll`

Requires an **Admin** API key. Forces the background collector to immediately poll
all data types for the specified cluster, resetting any open circuit first.

**Response** `200 OK`

```json
{ "cluster": "MY-CLUSTER-01", "requested": true }
```

### `GET /api/ops/clusters/{name}/roles/{roleName}/status`

Returns the current observed role state. Clients that need to wait for a role to
become healthy or offline should poll this endpoint with `?targetState=Online` or
`?targetState=Offline` and stop when `isConverged` becomes `true`.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "resourceType": "ClusterRole",
  "resourceName": "Cluster Core Resources",
  "state": "Online",
  "stateDisplay": "Online",
  "isConverged": true,
  "requestedState": "Online",
  "observedAtUtc": "2026-10-09T12:34:56Z"
}
```

### `POST /api/ops/clusters/{name}/roles/{roleName}/wait?targetState=Online&timeoutSeconds=60&pollIntervalMs=2500`

Thin convenience wrapper around the role status check. Polls until the role state
matches `targetState`, then returns a single success result. If the timeout is
reached without convergence, it returns `status: "TimedOut"` and `converged: false`.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "resourceType": "ClusterRole",
  "resourceName": "Cluster Core Resources",
  "targetState": "Online",
  "finalState": "Online",
  "status": "Succeeded",
  "converged": true,
  "elapsedMs": 4380,
  "observedAtUtc": "2026-10-09T12:34:56Z"
}
```

### `GET /api/ops/clusters/{name}/vms/{vmName}/status`

Returns the current observed VM state; this is the clean contract for client-side
wait loops after a VM mutation request has been accepted.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "resourceType": "VirtualMachine",
  "resourceName": "VM-01",
  "state": "Running",
  "stateDisplay": "Running",
  "isConverged": true,
  "requestedState": "Running",
  "observedAtUtc": "2026-10-09T12:34:56Z"
}
```

### `POST /api/ops/clusters/{name}/vms/{vmName}/wait?targetState=Running&timeoutSeconds=60&pollIntervalMs=2500`

Thin convenience wrapper around the VM status check. Polls until the VM state
matches `targetState`, returning a single final result on success or timeout.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "resourceType": "VirtualMachine",
  "resourceName": "VM-01",
  "targetState": "Running",
  "finalState": "Running",
  "status": "Succeeded",
  "converged": true,
  "elapsedMs": 5120,
  "observedAtUtc": "2026-10-09T12:34:56Z"
}
```

### `POST /api/ops/clusters/{name}/reset-circuit`

Requires an **Admin** API key. Clears the collector circuit-breaker state for the
specified cluster so the next scheduled poll can proceed immediately.

**Response** `200 OK`

```json
{ "cluster": "MY-CLUSTER-01", "reset": true }
```

### `POST /api/ops/clusters/{name}/vms/{vmName}/start`

Requires an **Admin** API key. Starts a VM.

> Use `POST` for VM actions.
> Calling these routes with `GET` returns `405 Method Not Allowed`.

### `POST /api/ops/clusters/{name}/nodes/{nodeName}/pause`

Requires an **Admin** API key. Pauses a node without draining workloads.

### `POST /api/ops/clusters/{name}/nodes/{nodeName}/drain`

Requires an **Admin** API key. Pauses the node and drains workloads.

### `POST /api/ops/clusters/{name}/nodes/{nodeName}/resume`

Requires an **Admin** API key. Resumes a paused node.

### `POST /api/ops/clusters/{name}/nodes/{nodeName}/failback`

Requires an **Admin** API key. Moves preferred roles back after drain work.

### `POST /api/ops/clusters/{name}/roles/{roleName}/start`

Requires an **Admin** API key. Starts a cluster role.

### `POST /api/ops/clusters/{name}/roles/{roleName}/stop`

Requires an **Admin** API key. Stops a cluster role.

### `POST /api/ops/clusters/{name}/roles/{roleName}/move?targetNode={nodeName}`

Requires an **Admin** API key. Moves a cluster role to another node.
Returns `409 Conflict` when the role is already on the target node or the role
state does not allow a move.

### `POST /api/ops/clusters/{name}/vms/{vmName}/stop`

Requires an **Admin** API key. Requests a graceful stop for a VM.

### `POST /api/ops/clusters/{name}/vms/{vmName}/restart`

Requires an **Admin** API key. Restarts a VM.

### `POST /api/ops/clusters/{name}/vms/{vmName}/suspend`

Requires an **Admin** API key. Suspends a VM.

### `POST /api/ops/clusters/{name}/vms/{vmName}/turn-off`

Requires an **Admin** API key. Force-stops (power off) a VM.

**Response** `200 OK`

```json
{
  "cluster": "MY-CLUSTER-01",
  "vm": "VM-01",
  "action": "Start",
  "requested": true,
  "detail": null,
  "state": "Running",
  "converged": true
}
```

VM, role and node actions wait (bounded) for the live state to reach the state the action should produce, then
write the observed state into the cached snapshot so `GET .../vms`, `.../roles` and `.../nodes` agree
immediately instead of waiting for the next scheduled poll. `state` is the state actually observed and `converged`
says whether it reached the expected one (for example `Starting` / `false` if the VM was still booting when the
wait ended). Role responses also carry `ownerNode`; node responses carry `drainStatus`. The wait length is
`Ops:VmActionSettleSeconds` (default `20`, `0`-`60`; `0` = a single immediate check).

### Extended Ops endpoints

Same base route, same API-key protection. Reads come from the poller's cached snapshots where one exists
(`isLive: false`, with `collectedAt` / `isStale`) and from a live cluster query otherwise (`isLive: true`; a
connectivity failure returns `502` with a short `detail`). List responses are bounded: `limit` (1-500, default 100) and
`total` / `returned` / `truncated` tell you when you are looking at a partial result.

| Method and path (under `/api/v1/ops/clusters/{name}`) | Source | Returns |
|---|---|---|
| `GET storage/pools`, `storage/virtual-disks`, `storage/physical-disks`, `storage/csvs` | snapshot | storage inventory and health |
| `GET network/intents`, `network/adapters`, `network/cluster-networks`, `network/storage-health` | snapshot | network inventory and health |
| `GET updates`, `updates/settings`, `arc`, `drift`, `vm-replication`, `vm-checkpoints`, `node-processes` | snapshot | updates, Arc, drift, replication, checkpoints, top processes |
| `GET health-summary`, `events?max=&level=`, `updates/readiness`, `updates/runs`, `updates/precheck`, `storage/jobs` | live | cluster health, events, update readiness and history, storage jobs |
| `GET storage/health-settings` | live | Health Service volume capacity thresholds (defaults 80% / 90%), minimum free memory (default 10%), auto-pool setting and every active override, plus advisories when a value is unsafe or would already raise faults on current volumes |
| `GET storage/health-settings/evaluate?volumeWarning=&volumeCritical=&memoryPercent=` | live | dry run: verdict (`ok`, `ok_with_notes`, `needs_acknowledgement`, `blocked`, `invalid`), warnings and the changes a write would make; omit all values to judge the current settings |
| `GET vms/{vmName}/detail`, `performance/vms`, `performance/nodes?timeFrame=`, `performance/volumes?timeFrame=`, `vm-storage-alignment` | live | VM detail and performance |
| `GET services`, `security/features`, `security/wdac`, `security/bitlocker`, `affinity-rules`, `virtual-switches` | live | node services, security posture, rules, switches |
| `GET roles/{roleName}/resources` | live | resources inside a cluster role |
| `GET /api/v1/ops/fleet/summary` | database | health of every registered cluster |
| `GET /api/v1/ops/alerts?cluster=&severity=&unacknowledgedOnly=&hours=&limit=` | database | recent alerts |
| `GET /api/v1/ops/audit?cluster=&action=&outcome=&resourceType=&resourceName=&hours=&limit=` | database | recent audit entries - **Admin** key only |

Actions (all audited, one at a time per resource, admin key or an RBAC-scoped key; `403` otherwise):

| Method and path (under `/api/v1/ops/clusters/{name}`) | Notes |
|---|---|
| `POST vms/{vmName}/move?targetNode={node}` | Live migration. The VM must be running and clustered, and the target node must be `Up` and different from the current host (`409` otherwise). The response includes `fromHost`, `targetHost`, the observed `host` and `converged`. Needs the `Migrate` operation on the VM. |
| `POST roles/{roleName}/resources/{resourceName}/start` | Starts one resource in a role. Already-online is a no-op; resources in a `*Pending` state return `409`. Needs the `Start` operation on the role. |
| `POST nodes/{nodeName}/services/{serviceName}/restart` | Restarts one of the monitored HCI agent services (`wssdagent`, `mochostagent`, `ArcProxy`, `ExtensionService`, `GCArcService`, `himds`, `LcmController`) on a node that is `Up`. Any other service name returns `400`. Needs the `Start` and `Stop` operations on Services. |
| `POST arc/sync` | Queues `Sync-AzureStackHCI`; Azure processes it asynchronously. **Admin** key only. |
| `POST storage/health-settings?volumeWarning=&volumeCritical=&memoryPercent=&acknowledgeRisk=` | Changes the Health Service volume capacity thresholds and/or minimum free memory percent. Only these three settings can be changed (never an arbitrary setting name). Values are validated (`400`); values far from the defaults (volume warning below 70 or above 90, critical below 80 or above 95, memory below 5 or above 20, or a volume that would immediately fault) return `409` with the warnings and are **not applied** unless `acknowledgeRisk=true`; extreme values (volume warning below 50, critical below 60, memory below 1 or above 30) return `422` and are refused even with `acknowledgeRisk`. Refusals are audited. Applied changes are verified, a failure part-way rolls back earlier changes, and the audit entry records the previous values. The change takes effect only after the `SDDC Management` role is restarted (`restartRequired: true`). Admin key or the `Configure` operation on Storage. |

---

## MCP endpoint (`/mcp`)

A Streamable HTTP [Model Context Protocol](https://modelcontextprotocol.io) endpoint for Azure SRE Agent, GitHub
Copilot, Copilot Studio and other agents. It is a thin layer over the Ops and Maintenance APIs above, so key level,
RBAC role, per-resource locks, state checks and audit logging all still apply. Authenticate with
`Authorization: Bearer <api-key>` (the `ApiKey <key>` scheme also works).

- **Tools:** 80 - reads (cached and live), maintenance windows, and confirm-gated actions. Action tools are hidden
  from read-level keys, need `confirm=true`, and should be set to **Ask** in the agent platform (`confirm` is a
  parameter the model fills in itself, not a human approval).
- **Errors:** tool failures return `isError: true`, and protocol errors a JSON-RPC `error`, always on HTTP `200`
  (MCP clients treat any 4xx as a transport failure). `GET /mcp` returns `405`.

### Tool profiles (per-key tool allowlists)

Every tool belongs to one group. A **profile** is a set of groups (plus optional extra tools and exclusions); each API
key is mapped to a profile by key **name**. A profile only narrows: the key's own level still applies on top, and a tool
outside the profile is neither listed nor callable (a call is answered exactly like an unknown tool and logged).

| Profile | Groups | Tools |
|---|---|---|
| `triage` | fleet, events, nodes, roles, vms | 30 (read only) |
| `diagnostics` | triage + performance, storage, network, updates, security, maintenance | 60 (read only) |
| `operator` | triage + maintenance + safe actions (`start_vm`, `start_role`, `resume_node`, `start_cluster_resource`) | 37 |
| `all` (aliases `admin`, `full`) | everything | 80 |

Groups: `fleet`, `events`, `audit`, `nodes`, `roles`, `vms`, `performance`, `storage`, `network`, `updates`,
`security`, `maintenance`, `actions-safe`, `actions-disruptive`, `actions-admin`.

```json
"Mcp": {
  "DefaultProfile": "all",
  "KeyProfiles": { "sre-triage": "triage", "sre-ops": "operator" },
  "Profiles": {
    "oncall": { "Extends": "triage", "Groups": ["storage"], "Tools": ["start_vm"], "Exclude": ["get_vm_detail"] }
  }
}
```

- Keys with no mapping get `DefaultProfile` (`all` by default, so existing connectors keep working).
- A key mapped to a profile that does not exist gets **no tools** (fail closed) and an error is logged. A `Profiles`
  entry with the same name as a built-in replaces it.
- The profile is chosen by the API key's **name** (Admin > Settings > API keys), not by its RBAC role name.
- Put the setting in the server's `appsettings.Production.json` (preserved across deploys). `appsettings.json` is
  replaced on every deploy and `appsettings.template.json` is only a template that the app never loads. Add it to
  `appsettings.WinAuth.json` too if agents also call the WinAuth site. Edits apply without restarting the app.
- To check it, call `tools/list` and look for the log line `MCP tools/list: key '<name>' -> profile '<profile>' -> N tools`.
  `(default, key not mapped)` in that line means the key name did not match any `KeyProfiles` entry.
- Use one key and one connector per profile; the Allow/Ask approval for each tool is still set per tool in the agent platform.

### Limits and caching

All settings are optional. Limits are per process (the main and WinAuth IIS sites each keep their own), so use the main
HTTPS site for agent traffic.

| Setting | Default | Meaning |
|---|---|---|
| `Mcp:RateLimit:TokensPerMinute` / `Burst` | 120 / same | Per-key budget. Cost: cached read 1, live read 5, expensive read 20, action 10, `tools/list` 1. |
| `Mcp:LiveConcurrencyPerCluster` | 2 | Concurrent live queries per cluster (1 expensive query at a time); extra calls are refused, not queued. |
| `Mcp:CacheEnabled` | true | Cache live reads and share one run between identical concurrent calls. |
| `Mcp:LiveCacheSeconds` / `ExpensiveCacheSeconds` | 30 / 120 | Cache lifetimes (the update pre-check uses 600). Status and `wait_for_*` tools are never cached; any action on a cluster clears its cache. |
| `Mcp:LiveTimeoutSeconds` / `ExpensiveTimeoutSeconds` | 30 / 60 | Stop waiting for a slow live query (it keeps its slot until it really ends). |
| `Mcp:MaxRequestBytes` | 65536 | Request body cap (`413` above it). |
| `Mcp:SessionIdleMinutes` / `MaxSessions` | 30 / 5000 | MCP session expiry and cap. |
| `Ops:VmActionSettleSeconds` | 20 | How long actions wait for the live state to settle (0-60). |

While a cluster's poll circuit breaker is open, live tools fail fast (`get_cluster_status` still runs so an agent can
see when the cluster is back). Failed live queries return a generic message to read keys; the detail is for Admin keys
and the application log.

---

## Audit logging

Every mutating API call (POST, PUT, DELETE) is written to the audit log
(visible at **Admin → Audit Log**) with:

- **User:** `API (<key-name>)`
- **User UPN:** `api-key:<key-name>`
- **User Object Id:** API key ID from the key store
- **Action:** `AddCluster`, `UpdateCluster`, `DeleteCluster`,
  `CreateMaintenanceWindow`, `DeleteMaintenanceWindow`,
  `CreateRecurringRule`, `DeleteRecurringRule`, `ForcePoll`, `ResetCircuit`,
  `PauseNode`, `ResumeNode`, `FailbackNode`, `StartRole`, `StopRole`,
  `MoveRole`, `Start`, `Stop`, `Restart`, `Suspend`, or `ForceStop`
- **Outcome:** `Success` or `Failure`

Read calls (GET) are not audited.

### End-to-end node action audit example

Request:

```http
POST /api/v1/ops/clusters/MY-CLUSTER-01/nodes/NODE-02/resume HTTP/1.1
Host: azlmgmt.yourdomain.com
Authorization: ApiKey 1x2y3z...
```

Response:

```json
{
  "cluster": "MY-CLUSTER-01",
  "node": "NODE-02",
  "action": "Resume",
  "requested": true,
  "detail": null
}
```

Audit row written by this call:

```json
{
  "timestamp": "2026-10-09T08:42:33Z",
  "clusterName": "MY-CLUSTER-01",
  "resourceType": "Node",
  "resourceName": "NODE-02",
  "action": "ResumeNode",
  "outcome": "Success",
  "userDisplayName": "API (CI Pipeline)",
  "userUpn": "api-key:CI Pipeline",
  "userObjectId": "7f1bd2ea-0f3f-44fd-82e6-a8dd58ec2b13",
  "detail": "State before action: Paused (InProgress)"
}
```

---

## Error responses

All errors return JSON:

```json
{ "error": "Human-readable description" }
```

| Status | Meaning |
| --- | --- |
| `400 Bad Request` | Validation failed — check required fields |
| `401 Unauthorized` | Missing or invalid API key |
| `403 Forbidden` | Read-level API key used for an Admin API endpoint |
| `404 Not Found` | Cluster name not found |
| `409 Conflict` | POST: cluster with that name already exists |
| `429 Too Many Requests` | Rate limit exceeded; retry in 60s (`Retry-After`) |
| `500 Internal Server Error` | Unexpected failure — check app logs |
| `503 Service Unavailable` | No API key has been configured |

---

## Future: Entra client credentials (Option B)

For enterprise environments where automation tooling already uses Entra identities,
an app-registration-based `client_credentials` grant will be added in a future release.
This will map to the `HciAdmin` policy automatically via an app role, removing the
shared-secret dependency. See BACKLOG-18 Option B.
