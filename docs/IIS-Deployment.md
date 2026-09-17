# Deployment Guide — Azure Local Cluster Tool

## Overview

The app runs on **IIS with ASP.NET Core in-process hosting**. The IIS worker process (`w3wp.exe`) is the app process — ASP.NET Core runs inside it rather than as a separate Kestrel process. The gMSA identity is set on the Application Pool, so all PowerShell/WinRM operations run as that identity automatically.

```
Browser → HTTPS → IIS (w3wp.exe running as gMSA) → WinRM → Cluster nodes
```

---

## App Server Requirements

| Item | Minimum |
|---|---|
| OS | Windows Server 2019 or 2022 |
| RAM | 4 GB (2 GB free for the app) |
| Disk | 2 GB free in C:\apps\azlmgmt |
| Network | LAN access to cluster nodes on port 5985 (or 5986 for HTTPS) |
| Domain | Must be domain-joined (required for Kerberos WinRM auth to cluster nodes; applies to both gMSA and standard service account deployments) |
| WinRM | WinRM client must be enabled on the app server |

---

## Features and Components Installed by Setup-Prerequisites.ps1

> All features and components listed in this section are installed automatically by `Setup-Prerequisites.ps1`.
> This section is a reference for verifying an existing setup or performing manual installation when the script cannot be run.

### Web Server (IIS) — `Web-Server`

| Feature | Role Service | Why needed |
|---|---|---|
| Web Server | `Web-WebServer` | Core web server role |
| Common HTTP Features | `Web-Common-Http` | Group container |
| → Default Document | `Web-Default-Doc` | Serves index on root request |
| → Static Content | `Web-Static-Content` | Blazor static assets (JS, CSS, wasm) |
| → HTTP Errors | `Web-Http-Errors` | Error pages |
| Health and Diagnostics | | Group |
| → HTTP Logging | `Web-Http-Logging` | Request logs to inetpub\logs |
| → Request Monitor | `Web-Request-Monitor` | Worker process health in IIS Manager |
| Security | | Group |
| → Request Filtering | `Web-Filtering` | Block malformed requests (security baseline) |
| Management Tools | `Web-Mgmt-Tools` | Group |
| → IIS Management Console | `Web-Mgmt-Console` | IIS Manager snap-in (MMC) |
| → IIS Management Scripts | `Web-Scripting-Tools` | Enables WebAdministration PowerShell module |
| Application Development | | Group |
| → WebSocket Protocol | `Web-WebSockets` | **Required for Blazor Server** — without this, SignalR falls back to Long Polling and the app degrades significantly |

### .NET Framework Features

| Feature | Why needed |
|---|---|
| `NET-Framework-45-ASPNET` | Required by some IIS modules even for .NET 10 apps |
| `NET-WCF-HTTP-Activation45` | HTTP activation for .NET services |

### RSAT Management Tools

Required so the app can call Hyper-V and FailoverClusters cmdlets **locally on the app server** via CimSession/`-Cluster` parameters, avoiding the double-hop WinRM penalty.

| Feature | Why needed |
|---|---|
| `RSAT-Hyper-V-Tools` | Enables `Get-VM`, `Start-VM`, `Get-VMSwitch`, `Get-VMNetworkAdapter`, snapshot cmdlets, CPU/memory config — used with `-CimSession` per node |
| `RSAT-Clustering-PowerShell` | Enables `Get-ClusterNode`, `Get-ClusterGroup`, `Move-ClusterGroup`, `Suspend-ClusterNode`, `Resume-ClusterNode` etc — used with `-Cluster <address>` |
| `RSAT-AD-PowerShell` | Enables `Get-ADServiceAccount` — required for gMSA SID resolution during NTFS permission setup |

### ASP.NET Core Hosting Bundle (separate installer)

**This is NOT a Windows Feature — it is a separate download.**

The Hosting Bundle installs:
- **ASP.NET Core Module v2 (ANCM)** — the IIS module that forwards requests into the .NET process
- The .NET runtime (not needed when deploying self-contained, but installed anyway)

Download URL: **https://dot.net** → Downloads → .NET 10 → **"Hosting Bundle"** (not the SDK or Runtime)

The bundle registers `AspNetCoreModuleV2` in `%windir%\system32\inetsrv\config\schema`. Without it, IIS cannot host the app and will return HTTP 500.

Run `iisreset` after installing the bundle before creating sites.

---

## Application Pool Configuration

| Setting | Value | Why |
|---|---|---|
| Name | `AZLManagementPool` | Logical name |
| .NET CLR Version | No Managed Code | ASP.NET Core doesn't use the CLR app pool |
| Pipeline Mode | Integrated | Required for ANCM in-process |
| Identity | `DOMAIN\azlmgmt-svc$` (gMSA) or `DOMAIN\svc-azlmgmt` (standard account) | This is what WinRM authenticates as |
| Identity password | Empty for gMSA; set for standard account | AD manages gMSA passwords automatically; standard accounts require manual rotation |
| 32-bit | False | App is 64-bit |
| Start Automatically | True | |

> **Critical:** The "No Managed Code" setting on the app pool does NOT break the app. It just means IIS doesn't load the classic .NET framework pipeline. ASP.NET Core uses its own module regardless of this setting.

> **SignalR settings:** `Setup-IIS.ps1` automatically disables idle timeout (`processModel.idleTimeout = 00:00:00`) and periodic worker process restart (`recycling.periodicRestart.time = 00:00:00`) on the app pool. Both are required for Blazor Server. If you create the app pool manually, apply these settings — see [Troubleshooting](#troubleshooting) for the PowerShell commands.

---

## Entra Application Proxy (recommended for external access)

Entra Application Proxy provides HTTPS access for external users without opening firewall ports or requiring a VPN. Install the connector on the app server or a dedicated connector server on the same network segment.

1. Follow the Microsoft guide: [Add an on-premises application — Entra Application Proxy](https://learn.microsoft.com/en-us/entra/identity/app-proxy/application-proxy-add-on-premises-application)
2. Set the **internal URL** to your IIS site (e.g. `https://azlmgmt.yourdomain.com`)
3. Add the **external URL** provided by Entra as a redirect URI in the app registration

The connector uses outbound HTTPS (port 443) only — no inbound firewall rules on the app server are needed.

---

## gMSA Requirements

> If you are using a **standard service account** instead of a gMSA, skip to [Running as a Standard Service Account](#running-as-a-standard-service-account-without-gmsa) below.

The gMSA must be configured in Active Directory **before** running `Setup-IIS.ps1`.

**Automated (recommended):** Use the included `scripts\New-AppServiceAccount.ps1` to create and configure the account in a single step. Run it on a Domain Controller or any machine with RSAT AD tools:

```powershell
# Creates azlmgmt-svc gMSA, grants azlmgmt computer account retrieval rights,
# installs on the app server, and grants local admin on each cluster node.
.\New-AppServiceAccount.ps1 `
    -AppServerName "azlmgmt" `
    -ClusterNodes  @("NODE1", "NODE2", "NODE3")
```

See `New-AppServiceAccount.ps1 -?` for all parameters including retrieval-group mode and standard account mode.

**Manual steps** (if you cannot run the script or need to troubleshoot):

### Check gMSA is installed and retrievable on the app server

```powershell
# Run on the app server — should return the gMSA object
Test-ADServiceAccount -Identity azlmgmt-svc
```

If this returns `False`, the app server's computer account hasn't been added to the gMSA's retrieval principals:

```powershell
# Run as Domain Admin
Set-ADServiceAccount -Identity azlmgmt-svc `
    -PrincipalsAllowedToRetrieveManagedPassword (
        Get-ADGroup "HCI-AppServers"  # or individual computer: Get-ADComputer "azlmgmt"
    )
```

### Grant gMSA access to cluster nodes

The gMSA needs to be a local Administrator on each cluster node (or on a JEA endpoint if configured):

```powershell
# Run on each cluster node, or deploy via GPO to the cluster OU
Add-LocalGroupMember -Group "Administrators" -Member "DOMAIN\azlmgmt-svc$"
```

### Verify WinRM works from the app server as the gMSA service account

From an elevated PS session on the app server, test connectivity using the same auth path the app will use:

```powershell
# This simulates what the app does — if this works, the app will work
$ws = New-Object System.Management.Automation.Runspaces.WSManConnectionInfo `
        ([Uri]"http://MY-CLUSTER-01.domain.local:5985/wsman")
$ws.AuthenticationMechanism = [System.Management.Automation.Runspaces.AuthenticationMechanism]::Default
$rs = [System.Management.Automation.Runspaces.RunspaceFactory]::CreateRunspace($ws)
$rs.Open()
Write-Host "Connection successful" -ForegroundColor Green
$rs.Close()
```

> Run this **as the gMSA** (e.g. by running the app pool process or using `runas`). Running it as your own admin account tests your credentials, not the gMSA's.

---

## Running as a Standard Service Account (without gMSA)

If your environment does not support gMSA (non-domain, workgroup, or AD gMSA capability not available), you can run the app as a regular domain service account instead. **No application code changes are required** — the credential logic is identity-agnostic.

**Automated (recommended):** use `scripts\New-AppServiceAccount.ps1 -AccountType Standard`:

```powershell
.\New-AppServiceAccount.ps1 `
    -AccountType "Standard" `
    -AccountName "svc-azlmgmt" `
    -ClusterNodes @("NODE1", "NODE2", "NODE3")
```

### How it works

`CredentialSource = "gMSA"` in `WebHyperVService` means "use the process identity, supply no explicit credential". The Kerberos ticket used for WinRM comes from whoever is running the IIS app pool — gMSA or standard account. The setting name is a misnomer; it applies to any process-identity approach.

### What changes compared to the gMSA setup

| Area | gMSA | Standard service account |
|---|---|---|
| App code / `CredentialSource` | `"gMSA"` | `"gMSA"` (same — no change) |
| `Setup-IIS.ps1` | `$serviceAccountType = "gMSA"`, `$serviceAccountPassword = ""` | `$serviceAccountType = "Standard"`, `$serviceAccountPassword = "<password>"` |
| IIS identity password | Blank (AD manages) | Must supply; re-run script or update in IIS Manager when rotated |
| NTFS permissions | SID-based `icacls` (needed because `$` in gMSA name confuses some tools) | Plain `icacls /grant DOMAIN\svc:(OI)(CI)RX` — script handles this automatically |
| AD pre-requisite | gMSA must exist and app server must be in retrieval principals | Just need a standard domain account with a password |
| Key Vault access (if used) | Assign `Key Vault Secrets User` to gMSA object ID | Assign `Key Vault Secrets User` to service account object ID |
| WinRM on cluster nodes | gMSA needs Administrators membership on nodes | Same — standard account needs Administrators membership on nodes |
| Password management | None — AD rotates every 30 days automatically | You must rotate and update the app pool identity when the password changes |

### Setup steps for a standard service account

**Automated (recommended):** use `scripts\New-AppServiceAccount.ps1 -AccountType Standard`:

```powershell
.\New-AppServiceAccount.ps1 `
    -AccountType "Standard" `
    -AccountName "svc-azlmgmt" `
    -ClusterNodes @("NODE1", "NODE2", "NODE3")
```

The script creates the AD user, grants Administrators membership on each cluster node, then prints next-step instructions for `Setup-IIS.ps1`.

**Manual steps** (if you cannot run the script):

1. **Create the account in Active Directory:**
   ```powershell
   # Run as Domain Admin
   New-ADUser -Name "svc-azlmgmt" -SamAccountName "svc-azlmgmt" `
       -AccountPassword (Read-Host -AsSecureString "Password") `
       -PasswordNeverExpires $true -Enabled $true
   ```
   > Set `PasswordNeverExpires` to avoid unexpected app pool failures when the password expires. Alternatively accept expiry and treat IIS reconfiguration as a recurring task.

2. **Grant the account local admin on each cluster node** (same as gMSA):
   ```powershell
   # Run on each cluster node, or deploy via GPO
   Add-LocalGroupMember -Group "Administrators" -Member "DOMAIN\svc-azlmgmt"
   ```

3. **Run `Setup-IIS.ps1` with the service account parameters:**
   ```powershell
   .\Setup-IIS.ps1 -serviceAccountType Standard -serviceAccount "DOMAIN\svc-azlmgmt" -serviceAccountPassword "YourPasswordHere"
   ```

5. **Leave `CredentialSource = "gMSA"`** in the cluster config in the app (or leave it blank to use the default). No changes in `appsettings.json` or the Admin Clusters UI.

### Password rotation procedure

When the service account password changes:

```powershell
# Option A: Update via IIS Manager on the server
# Application Pools -> AZLManagementPool -> Advanced Settings -> Identity -> Set password

# Option B: Re-run the setup script with the new password
.\Setup-IIS.ps1 -serviceAccountType Standard -serviceAccount "DOMAIN\svc-azlmgmt" -serviceAccountPassword "NewPassword"
# Script is idempotent -- it will update the processModel on the existing pool
```

After updating, recycle the app pool:
```powershell
Invoke-Command -ComputerName azlmgmt.yourdomain.com {
    Import-Module WebAdministration
    Restart-WebItem "IIS:\AppPools\AZLManagementPool"
}
```

---

## AKS on Azure Local — Additional Prerequisites (optional)

These steps are only required if you manage **Arc-connected AKS clusters** deployed on Azure Local. Skip this section if you do not use the AKS pages.

### kubelogin

The portal calls `kubelogin get-token` to acquire Proof-of-Possession tokens for Arc-connected Kubernetes clusters. The binary must be present on the app server in the **machine-level PATH** — user-specific directories (e.g. `%LOCALAPPDATA%\Microsoft\WinGet\...`) are not visible to the gMSA app pool process.

```powershell
# Install via winget
winget install Microsoft.Azure.Kubelogin
```

> **Minimum version: v0.2.0** — run `kubelogin --version` after install. Run
> `winget upgrade Microsoft.Azure.Kubelogin` if the version shown is below v0.2.0.
> Older versions contain a PoP token cache nil pointer bug that causes a Go runtime panic
> (exit code 2) on every `get-token` call.

```powershell
# Copy to System32 so it is on the machine-level PATH
Copy-Item (Get-Command kubelogin).Source "C:\Windows\System32\kubelogin.exe"

# Verify it is accessible to all processes
$env:PATH -split ';' | ForEach-Object { Join-Path $_ "kubelogin.exe" } |
    Where-Object { Test-Path $_ }
# Should return: C:\Windows\System32\kubelogin.exe
```

### Azure role assignments and Admin > Settings configuration

The ARM SPN configured in **Admin > Settings** requires four Azure role assignments on the connected cluster resource:

| Role | Purpose |
|---|---|
| Azure Arc Enabled Kubernetes Cluster User Role | Call `listClusterUserCredential` to get the kubeconfig from ARM |
| Azure Kubernetes Service Arc Cluster User Role | Get AKS credentials for Arc-hosted clusters specifically |
| **Azure Kubernetes Service Arc Cluster Admin Role** | List/read k8s resources (namespaces, pods) — most commonly missed |
| Flux Configurations Contributor | Read and write GitOps configurations (required for Force Resync) |

For the full setup procedure including PowerShell commands, Admin > Settings values, token flow explanation, and troubleshooting table, see **[docs/AKS-Arc-SPN-Setup.md](AKS-Arc-SPN-Setup.md)**.

---

## PostgreSQL Setup (recommended for production)

PostgreSQL is the recommended database for production deployments. It has no size limit, is free and open-source, and requires no licensing.

### Install PostgreSQL on Windows

```powershell
# Via winget (Windows 10 1709+ / Windows Server 2019+) — use the current stable release
winget install -e --id PostgreSQL.PostgreSQL.17

# Via Chocolatey
choco install postgresql --version 17
```

Or download the Windows installer from https://www.postgresql.org/download/windows/

### Create the database and user

After install, open `psql` as the `postgres` superuser:
```powershell
# winget adds psql to PATH — open a new shell after install
psql -U postgres
# Or by full path: & "C:\Program Files\PostgreSQL\17\bin\psql.exe" -U postgres
```

At the `postgres=#` prompt:

```sql
CREATE USER azlmgmt_app WITH PASSWORD 'your_strong_password';
CREATE DATABASE azlmgmt OWNER azlmgmt_app;
GRANT ALL PRIVILEGES ON DATABASE azlmgmt TO azlmgmt_app;
\q
```

> **Tip:** The password you choose here must match the `Password=` value in `appsettings.json`.

### Grant the app pool identity access (gMSA)

For the gMSA identity to authenticate to a local PostgreSQL instance, add it to `pg_hba.conf`:

```
# In pg_hba.conf (usually C:\Program Files\PostgreSQL\17\data\pg_hba.conf)
# Allow the azlmgmt_app user from localhost with scram-sha-256 (PG 17 default)
host    azlmgmt    azlmgmt_app    127.0.0.1/32    scram-sha-256
host    azlmgmt    azlmgmt_app    ::1/128          scram-sha-256
```

Reload PostgreSQL after editing: `pg_ctl reload` or restart the `postgresql-x64-17` service.

### Configure appsettings.json for PostgreSQL

```json
"Database": {
  "Provider": "PostgreSQL",
  "ConnectionString": "Host=127.0.0.1;Database=azlmgmt;Username=azlmgmt_app;Password=your_strong_password"
},
"DataProtection": {
  "KeyPath": "C:\\apps\\azlmgmt-data\\dp-keys"
},
```

> **Important:** `DataProtection:KeyPath` is required when using PostgreSQL (there is no DB file path to derive it from automatically). The directory must exist and be writable by the app pool identity. The `Setup-Prerequisites.ps1` script creates `C:\apps\azlmgmt-data` with the correct permissions.

### Migrate an existing SQLite database to PostgreSQL

Use `scripts\Migrate-SqliteToPgsql.ps1`. The script uses the app's own deployed DLLs
(Npgsql + Microsoft.Data.Sqlite) — no pgloader or additional tools required.

The script requires **PowerShell 7** (`pwsh.exe`):

```powershell
# Install PS7 if not already present (free, does not replace PS 5.1)
winget install Microsoft.PowerShell
```

Then run the migration:

```powershell
# Stop the app pool first to prevent writes during migration
Stop-WebAppPool -Name 'AZLManagementPool'

# Run from the scripts\ folder (prompts for PG password interactively)
pwsh .\Migrate-SqliteToPgsql.ps1

# Update appsettings.Production.json to use PostgreSQL (see above), then:
Start-WebAppPool -Name 'AZLManagementPool'
```

The script discovers all tables dynamically, handles SQLite-to-PostgreSQL type conversions
(booleans, timestamps), inserts in transactions, and verifies row counts at the end.

### Verify

After starting the app pool, browse to the app and check:
- All clusters appear in the cluster picker
- `/admin/clusters` shows the correct cluster list
- `/admin/audit` shows existing audit log entries
- The app log shows `[startup] Database initialised (PostgreSQL)` or similar

### Legacy Clusters schema compatibility (PostgreSQL)

Some upgraded environments may retain legacy `Clusters` columns (`AksApiEndpoint`, `AksApiToken`) from older builds.
Current versions do not write these fields on insert, so they must either be nullable or have defaults.

Read-only check:

```sql
SELECT ordinal_position, column_name, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public' AND table_name = 'Clusters'
ORDER BY ordinal_position;
```

If the legacy columns exist and are `NOT NULL` with no default, set defaults:

```sql
ALTER TABLE "Clusters" ALTER COLUMN "AksApiEndpoint" SET DEFAULT '';
ALTER TABLE "Clusters" ALTER COLUMN "AksApiToken" SET DEFAULT '';
```

This change is safe and idempotent. Newer builds also apply this compatibility guard at startup when those columns are present.

---

## Data Protection Key Backup

The app encrypts sensitive settings (ARM SPN credentials, SMTP password, Teams webhook URL, etc.)
using ASP.NET Core Data Protection, backed by DPAPI on Windows. The key ring is stored on disk at
`DataProtection:KeyPath` (default: `C:\apps\azlmgmt-data\dp-keys`).

**DPAPI keys are machine-bound.** They cannot be decrypted on a different machine. If the app
server is rebuilt without restoring the key ring, all encrypted DB values (`AppSettings` rows with
`IsSensitive = true`) become permanently unreadable. The operator must re-enter every secret after
the rebuild.

### What to back up

Back up the entire `DataProtection:KeyPath` directory. Each `.xml` file in it is an individual key,
plus the `key-{id}.xml` files contain the actual encryption material.

```
C:\apps\azlmgmt-data\dp-keys\
    key-{guid}.xml
    key-{guid}.xml
    ...
```

### How to back up

```powershell
# Run from a machine with access to the app server
# Back up to a network share (run as domain admin or gMSA identity):
robocopy "C:\apps\azlmgmt-data\dp-keys" "\\backup-server\share\azlmgmt\dp-keys" /MIR /LOG:dp-keys-backup.log

# Or copy locally and store offsite:
Compress-Archive -Path "C:\apps\azlmgmt-data\dp-keys\*" `
                 -DestinationPath "C:\backup\dp-keys-$(Get-Date -f yyyyMMdd).zip"
```

**When to back up:** after any secrets are added or changed via `/admin/settings`. A weekly
scheduled robocopy is sufficient for most environments.

### How to restore

1. Stop the app pool before restoring: `Stop-WebAppPool -Name 'AZLManagementPool'`
2. Copy the backed-up key files to `DataProtection:KeyPath` on the new server
3. Ensure the gMSA identity (`DOMAIN\account$`) has **Read** and **List** access to the directory
4. Start the app pool: `Start-WebAppPool -Name 'AZLManagementPool'`

The app discovers keys automatically — no configuration change is needed.

### gMSA identity permissions reminder

The `Setup-Prerequisites.ps1` script grants the gMSA `Full Control` on the data directory. On a
rebuilt server, re-run the script or grant permissions manually:

```powershell
$sid = (Get-ADServiceAccount -Identity 'hci-web-svc').SID.Value
icacls "C:\apps\azlmgmt-data\dp-keys" /grant "*${sid}:(OI)(CI)F" /T
```

---

## Performance Tuning

The application is designed to scale horizontally across many clusters with minimal resource
usage. The defaults are appropriate for most deployments. This section explains the key
settings and when to change them.

### RunspacePool size (`Clusters:RunspacePoolMax`)

Each cluster connection owns an independent PowerShell RunspacePool. The pool size controls
how many PS operations on the **same cluster** can execute in parallel. Operations that exceed
the pool size queue and wait — they do not fail.

**Default: 5.** This is the right value for the vast majority of deployments:

| Scenario | Behaviour with default (5) |
|---|---|
| 5 users click Refresh on the same cluster simultaneously | All 5 served in parallel — no queuing |
| 50 users each managing a different cluster | 50 independent pools, each with 5 slots — zero cross-cluster contention |
| 10 users all hit the same cluster at once | First 5 served immediately; next 5 queue briefly (~1-3 s) |

Each runspace corresponds to one `wsmprovhost.exe` process on the cluster nodes. With
`RunspacePoolMax=5` and the ClusterConnectionPool LRU cap of 50 active clusters, the worst-
case total `wsmprovhost.exe` count per node is **50 x 5 = 250** — well within Windows WinRM
limits.

**To change**, add or edit in `appsettings.Production.json`:

```json
"Clusters": {
  "RunspacePoolMax": 5
}
```

| Value | When to use |
|---|---|
| `3` | Very low traffic; single admin; want minimal footprint on cluster nodes |
| `5` | Default — covers up to ~8-10 simultaneous active users per cluster comfortably |
| `8` | 10+ users regularly on the same cluster seeing slow page loads |
| `10` | High-traffic shared cluster (monitor `wsmprovhost.exe` count on nodes after changing) |

### PostgreSQL connection pool (`Maximum Pool Size`)

Npgsql's default pool size of 100 collides with PostgreSQL's default `max_connections=100`,
causing `FATAL: sorry, too many clients already` under any concurrency. The app enforces
`Maximum Pool Size=20` automatically in code regardless of the connection string setting.

For a Blazor Server app doing simple cluster-config reads and audit-log writes, 20 connections
provides thousands of DB operations per second — far more than any realistic load. Do not
increase this unless PostgreSQL's `max_connections` has been raised and you have a measured
reason to do so.

### Active cluster LRU cap (`ClusterConnectionPool.MaxActiveConnections`)

The pool keeps at most **50 clusters** connected simultaneously. When a 51st cluster is
accessed, the least-recently-used cluster is disconnected. Subsequent access to an evicted
cluster reconnects transparently.

This is a compile-time constant (`const int MaxActiveConnections = 50` in
`ClusterConnectionPool.cs`). For deployments managing more than 50 frequently-accessed
clusters simultaneously, increase this value and rebuild. For most organisations, 50 is
far more than enough — only clusters being actively viewed hold a connection.

### Multi-instance deployments (web farm / high availability)

> **This application is designed as a single-instance deployment.** The defaults and
> architecture assume one IIS worker process. Most organisations do not need to change this.

If you need high-availability or load-balanced deployments, two requirements apply:

**1. Sticky sessions (required)**

Blazor Server uses persistent SignalR circuits — each user's circuit is bound to the server
instance that created it. Without sticky sessions, a load balancer routing requests to a
different instance will break the circuit and disconnect the user.

Configure **Application Request Routing (ARR) affinity** on your load balancer so all
requests from a given user always reach the same server instance:

```powershell
# IIS ARR — enable session affinity on the server farm
Add-WebConfiguration -Filter "webFarms/webFarm[@name='MyFarm']/applicationRequestRouting/affinitySettings" `
    -Value @{ affinityCookieName = "ARRAffinity"; }
```

**2. SignalR backplane (optional, recommended for 2+ instances)**

Without a backplane, real-time updates (poller-driven page refreshes) only reach users on the
same instance as the poller. Add the Azure SignalR Service backplane to share circuit state
across instances:

```csharp
// Program.cs — replace AddSignalR() with:
builder.Services.AddSignalR()
    .AddAzureSignalR(builder.Configuration["Azure:SignalRConnectionString"]);
```

For single-instance deployments (the recommended topology), no changes are needed.

---

## Deployment Workflow

### First time

**Default paths:** binaries at `C:\apps\azlmgmt`, runtime data at `C:\apps\azlmgmt-data`.
To use different paths, pass `-AppPath` and `-DataPath` to `Setup-Prerequisites.ps1`:
```powershell
.\Setup-Prerequisites.ps1 -ServiceAccount "DOMAIN\azlmgmt-svc$" `
    -AppPath  "D:\apps\azlmgmt" `
    -DataPath "D:\apps\azlmgmt-data"
```
Pass the same paths to `Setup-IIS.ps1 -physicalPath` and `Install.ps1 -AppPath` if changed.

1. **On the app server** — copy and run `scripts\Setup-Prerequisites.ps1`:
   ```powershell
   # Copy scripts to server then run
   \\azlmgmt.yourdomain.com\c$\scripts\Setup-Prerequisites.ps1
   # Restart if prompted
   \\azlmgmt.yourdomain.com\c$\scripts\Setup-IIS.ps1
   # Or: Copy-Item scripts\Setup-*.ps1 \\azlmgmt.yourdomain.com\c$\temp\ then RDP and run
   ```

2. **Install the ASP.NET Core Hosting Bundle** — handled by `Setup-Prerequisites.ps1`. If ANCM is missing, the setup script will prompt for manual install or try winget.

3. **On the app server** - run `scripts\Add-HttpsBinding.ps1` to create the TLS certificate and HTTPS binding:
   ```powershell
   .\Add-HttpsBinding.ps1 -HostHeader azlmgmt.yourdomain.com
   ```
   This creates a self-signed cert valid for 3 years and adds port 443 binding to the IIS site.

4. **In Entra Portal** - add redirect URIs to the app registration (Authentication blade):
   - `https://azlmgmt.yourdomain.com/signin-oidc`
   - `https://azlmgmt.yourdomain.com/signout-callback-oidc`

5. **Deploy the app binaries** — choose one:

   **Release ZIP** (recommended for most deployments):
   ```powershell
   # From the extracted ZIP's scripts\ folder on the app server
   .\Install.ps1
   ```
   This copies app files to `C:\apps\azlmgmt` and places `appsettings.Production.json` and `clusters.json` from the templates. Edit those files, then run Setup-IIS.ps1.

   **From source (developer workflow):**
   ```powershell
   # From your dev machine with .NET SDK installed
   .\scripts\Deploy-ToIIS.ps1 -targetServer azlmgmt.yourdomain.com -Credential (Get-Credential)
   ```
   This publishes a fresh build and mirrors it to the server in one step.

6. **Browse to** `https://azlmgmt.yourdomain.com` - you should be redirected to Entra ID sign-in.

### Every subsequent deploy (code changes)

**Release ZIP upgrade** (on the app server, from the extracted ZIP's `scripts\` folder):
```powershell
.\Install.ps1 -Upgrade
# If your pool has a non-default name:
.\Install.ps1 -Upgrade -AppPoolName "AZLManagementPool" -WinAuthPoolName "AZLManagementWinPool"
```

**Developer workflow** (from dev machine with .NET SDK, building from source):
```powershell
cd F:\github\AzureLocal-ClusterTool-Web
.\scripts\Deploy-ToIIS.ps1 -targetServer azlmgmt.yourdomain.com
```

Both approaches stop the app pool, replace the files, and restart the pool. `Install.ps1 -Upgrade`
preserves `appsettings.Production.json` and `clusters.json` automatically. Takes ~30-60 seconds.

### Updating clusters.json on the server (add/remove clusters)

```powershell
# Edit the file directly on the server
notepad \\azlmgmt.yourdomain.com\c$\apps\azlmgmt\clusters.json

# Then recycle the app pool for changes to take effect (ClusterRegistry loads at startup)
Invoke-Command -ComputerName azlmgmt.yourdomain.com {
    Import-Module WebAdministration
    Restart-WebItem "IIS:\AppPools\AZLManagementPool"
}
```

---

## HTTPS Setup (when ready for production)

1. Obtain a certificate (domain cert from AD CS, Let's Encrypt via win-acme, or import a PFX):
   ```powershell
   # Import a PFX
   $cert = Import-PfxCertificate -FilePath "azlmgmt.pfx" `
               -CertStoreLocation Cert:\LocalMachine\My `
               -Password (Read-Host -AsSecureString "PFX password")
   ```

2. Add the HTTPS binding in IIS:
   ```powershell
   Import-Module WebAdministration
   New-WebBinding -Name "AZLManagement" -Protocol https -Port 443 `
       -HostHeader "azlmgmt.yourdomain.com" -SslFlags 1

   # Bind the cert (requires netsh for SNI bindings)
   netsh http add sslcert hostnameport="azlmgmt.yourdomain.com:443" `
       certhash=$($cert.Thumbprint) appid="{$(New-Guid)}" certstorename=MY
   ```

3. Remove the HTTP binding (optional — or keep for internal redirect):
   ```powershell
   Remove-WebBinding -Name "AZLManagement" -Protocol http -Port 80
   ```

---

## Troubleshooting

| Symptom | Check |
|---|---|
| **HTTP 503 Service Unavailable** | App pool is stopped — see 503 diagnosis section below |
| HTTP 500.19 on first browse | ASP.NET Core Hosting Bundle not installed — install it and run `iisreset` |
| HTTP 500.30 (app failed to start) | Check Windows Event Log → Application for `IIS AspNetCore Module` errors |
| HTTP 403 Forbidden | Request Filtering or authentication misconfigured |
| Cluster page loads but VM list is empty | gMSA not in Administrators on cluster node; WinRM not enabled on nodes |
| Sign-in redirects back to sign-in | Entra app registration missing redirect URI `https://azlmgmt.yourdomain.com/signin-oidc` |
| App pool stops immediately after start | gMSA not retrievable on app server (`Test-ADServiceAccount` returns False) |
| Robocopy fails during upgrade | Ensure the app pool is stopped and your account has write rights to the app folder |
| Blazor disconnects every ~29 hours; browser console shows "connection could not be found on server" then falls back to Long Polling | IIS periodic restart (`recycling.periodicRestart.time`) not disabled — apply the fix below |
| Blazor disconnects every ~20 min when site is idle (no active users) | IIS idle timeout (`processModel.idleTimeout`) not disabled — apply the fix below |
| Blazor disconnects frequently on one network but not another | Corporate proxy intercepting WebSocket traffic — see Proxy Bypass below |

### SignalR app pool settings (if not set by Setup-IIS.ps1)

Blazor Server requires two app pool settings that IIS does not apply by default. `Setup-IIS.ps1` sets these automatically. If you created the app pool manually or are troubleshooting disconnects, apply them:

| Setting | Required value | Default | Effect if not set |
|---|---|---|---|
| `processModel.idleTimeout` | `00:00:00` (never) | 20 min | IIS kills the worker process after 20 min idle — drops all active SignalR circuits |
| `recycling.periodicRestart.time` | `00:00:00` (disabled) | 29 h | Worker restarts on a timer — all clients get "connection could not be found on server" and fall back to Long Polling |

```powershell
Import-Module WebAdministration
$pool = "AZLManagementPool"

Set-ItemProperty "IIS:\AppPools\$pool" -Name processModel.idleTimeout       -Value "00:00:00"
Set-ItemProperty "IIS:\AppPools\$pool" -Name recycling.periodicRestart.time -Value "00:00:00"

# Verify (both must show 00:00:00)
(Get-ItemProperty "IIS:\AppPools\$pool").processModel.idleTimeout
(Get-ItemProperty "IIS:\AppPools\$pool").recycling.periodicRestart.time
```

### HTTP 503 — App Pool Stopped: Diagnosis Steps

503 means IIS has stopped the app pool. Run these on the server in order:

**Step 1 — Check app pool state**
```powershell
Import-Module WebAdministration
Get-WebAppPool -Name AZLManagementPool | Select-Object Name, State, ManagedRuntimeVersion
```
If State is `Stopped`, try to start it manually and watch if it stops again immediately:
```powershell
Start-WebAppPool -Name AZLManagementPool
Start-Sleep -Seconds 3
(Get-WebAppPool -Name AZLManagementPool).State   # should be "Started"
```

**Step 2 — Check the Windows Event Log for the crash reason**
```powershell
Get-EventLog -LogName Application -Newest 20 |
    Where-Object { $_.Source -match "IIS|ANCM|AspNetCore|W3SVC" } |
    Format-List TimeGenerated, Source, Message
```
Also check:
```powershell
Get-EventLog -LogName System -Source "Microsoft-Windows-WAS" -Newest 10 |
    Format-List TimeGenerated, Message
```
`WAS` (Windows Process Activation Service) logs app pool state changes with a reason code.

**Step 3 — Check ANCM is installed**
```powershell
Get-WebConfiguration "system.webServer/globalModules/*" |
    Where-Object { $_.name -like "*AspNetCore*" } |
    Select-Object name, image
```
If nothing is returned: **the ASP.NET Core Hosting Bundle is not installed.** Download from https://dot.net → .NET 10 → Hosting Bundle, install, then `iisreset`.

**Step 4 — Check the gMSA is retrievable**
```powershell
Test-ADServiceAccount -Identity azlmgmt-svc
# Must return True. False means app pool cannot obtain a Kerberos ticket.
```
If False, add the app server computer account to the gMSA retrieval principals (see gMSA section above).

**Step 5 — Enable stdout logging and check the startup log**

Edit `C:\apps\azlmgmt\web.config`, set `stdoutLogEnabled="true"`, recycle the pool, browse again, then check:
```powershell
Get-ChildItem C:\apps\azlmgmt\logs\stdout_*.log | Sort-Object LastWriteTime -Descending | Select-Object -First 1 | Get-Content
```
This shows the exact .NET exception if the app is crashing before it can serve requests.

### Event log locations

```powershell
# ASP.NET Core Module startup errors
Get-EventLog -LogName Application -Source "IIS AspNetCore Module V2" -Newest 10

# General IIS errors
Get-EventLog -LogName System -Source "Microsoft-Windows-IIS*" -Newest 10

# App stdout logs (if enabled in web.config)
# C:\apps\azlmgmt\logs\stdout_*.log
```

### Enable stdout logging temporarily (for startup crash diagnosis)

Edit `C:\apps\azlmgmt\web.config` on the server:
```xml
<aspNetCore processPath=".\AzureLocal.ClusterTool.Web.exe"
            arguments=""
            stdoutLogEnabled="true"        <!-- change to true -->
            stdoutLogFile=".\logs\stdout"
            hostingModel="inprocess">
```

Recycle the app pool, reproduce the error, check `C:\apps\azlmgmt\logs\stdout_*.log`. Set back to `false` after diagnosing — stdout logging has a performance cost.

### Proxy Bypass — Blazor WebSocket disconnects on specific networks

Blazor Server uses a persistent WebSocket (`wss://`). If a corporate proxy intercepts it, the proxy's own session lifetime (typically 30–60 min) terminates the connection regardless of keepalive settings.

**Diagnose — run on an affected client machine:**
```powershell
# If this returns the proxy address instead of the original URL, traffic is going through the proxy
[System.Net.WebRequest]::GetSystemWebProxy().GetProxy("http://azlmgmnt.yourdomain.net.au")
```

**Common cause:** Windows proxy bypass lists use `*.domain.net.au` which only matches **one level deep**. A server at `http://azlmgmnt.yourdomain.net.au` has four levels and falls through to the proxy despite the bypass rule existing.

**Fix — add sub-domain levels to the bypass list:**

PAC file (add before the catch-all `return "PROXY ..."`):
```javascript
if (shExpMatch(host, "*.domain.net.au"))           { return "DIRECT"; }
if (shExpMatch(host, "*.ad.domain.net.au"))        { return "DIRECT"; }

```

GPO / registry `ProxyOverride` value:
```
*.domain.net.au;*.ad.domain.net.au;<local>
```

After the PAC/GPO change, verify on the client:
```powershell
# Must return the original URL (not the proxy) to confirm bypass is working
[System.Net.WebRequest]::GetSystemWebProxy().GetProxy("http://azlmgmnt.yourdomain.net.au")
```

> The app-side `serverTimeout` (120 s) and `KeepAliveInterval` (10 s) settings tolerate brief proxy delays but cannot overcome a proxy hard-terminating sessions. The bypass is the correct fix for corporate LAN-to-LAN traffic.
