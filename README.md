# Azure Local Cluster Tool — Web

[![CI](https://github.com/jasontayler/AzureLocal-ClusterTool-Web/actions/workflows/integration-tests.yml/badge.svg)](https://github.com/jasontayler/AzureLocal-ClusterTool-Web/actions/workflows/integration-tests.yml)
[![Release](https://github.com/jasontayler/AzureLocal-ClusterTool-Web/actions/workflows/release.yml/badge.svg)](https://github.com/jasontayler/AzureLocal-ClusterTool-Web/actions/workflows/release.yml)

> **v0.10.1** — Latest stable release. Please report bugs and
> feedback via [GitHub Issues](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues).

A **Blazor Server** web application for managing **Azure Stack HCI (Azure Local)** clusters and
Hyper-V hosts from any browser. Provides VM operations, node management, storage monitoring,
network (ATC intents), solution updates, alerting, Azure Arc status, and more — without needing
Remote Desktop or multiple PowerShell windows.

---

## Features

| Area | Capabilities |
|---|---|
| **Virtual Machines** | List, Start, Stop, Force Stop, Restart, Suspend, Live Migrate; VM detail (NICs, disks, integration services, snapshots); Configure CPU and memory; VM performance metrics (CPU%, memory, VHD IOPS/latency, network) |
| **Virtual Switches** | Hyper-V virtual switch inventory per node — type, SET, management OS adapters |
| **Cluster Nodes** | Live CPU, memory and uptime stats; OS build + display version; Pause/Drain, Resume, Failback |
| **Cluster Roles** | List, Start, Stop, Move (failover to node) |
| **Cluster Info** | Quorum mode and witness, S2D status, health faults, Cluster Shared Volumes |
| **Storage** | Storage pools, virtual disks, physical disks; Storage QoS volumes with read/write IOPS and latency; **Disk Replacement Wizard** — guided 3-step modal with pre-flight checks, retire command, and live repair job polling (feature-flag gated) |
| **Network** | Physical adapters with driver info; ATC intents + live status; cluster networks; logical networks (ARM); SMB health |
| **Events** | Cluster event log viewer with CSV export |
| **Remote Log Viewer** | Browse and tail log files directly from cluster nodes; auto-refresh with line interval picker |
| **Solution Updates** | List available updates, view runs, start an update, live streaming progress monitor; ARM catalog source when configured |
| **Azure Arc** | Registration and portal properties; Arc Machines; Arc Extensions (with upgrade detection); Cluster Extensions; Arc Resource Bridge appliance and Azure Local Sites; Custom Locations |
| **Agent Services** | View and control HCI agent services (wssdagent / mochostagent) |
| **Alerting** | Rules engine with configurable thresholds and cooldown; Teams webhook and SMTP email delivery; maintenance windows to suppress alerts during planned work; alert history with acknowledgement; VM name glob pattern filtering for VM stop alerts |
| **Fleet Status** | Default home page — multi-cluster dashboard with interactive filter pills (VMs Running, Nodes Up, Health Faults, Updates); user Favourites (pin clusters); personal named views (explicit / wildcard / regex); admin-created global and group-scoped shared views; zero WinRM calls (DB reads only) |
| **Fleet VM Status** | Fleet-wide VM table across all clusters — state, memory, uptime, node; interactive state-filter pills and name search; snapshot data, no live WinRM |
| **Snapshot Freshness** | Dedicated report page — per-data-type freshness grid with stale-cluster warning banner and exact timestamps on hover |
| **Admin — Clusters** | Multi-cluster CRUD management with audit trail |
| **Admin — Shared Views** | Create and manage global/group-scoped named cluster views for Fleet Status (explicit, wildcard, regex pattern types) |
| **Admin — Alerts** | Alert rule management — create, edit, enable/disable rules; alert history viewer |
| **Admin — Audit Log** | Full audit trail of all mutating operations, CSV export, configurable retention purge |
| **Admin — Settings** | Encrypted app-level settings in DB (ARM auth mode, SPN credentials, SMTP, webhook) — grouped collapsible UI |
| **Admin — Roles (RBAC)** | Fine-grained per-resource-type access control via Entra security groups; role badges in top bar |
| **Admin — Perf Debug** | Live call log of the last 500 WinRM/CIM/ARM/Local operations with duration and transport type badge |
| **Background Collector** | Automatic background polling with per-type schedules and exponential back-off; circuit breaker per cluster; pages load instantly from cache; amber stale banner when data is old |

---

## Technology Stack

| Item | Value |
|---|---|
| Framework | .NET 10.0 Blazor Server |
| Authentication | Microsoft Entra ID SSO (`Microsoft.Identity.Web`) or Windows Authentication (second site) |
| Authorisation | Entra security groups + fine-grained custom RBAC |
| PowerShell | `Microsoft.PowerShell.SDK` 7.5.4 — local RSAT + CIM API + WinRM where required |
| Database | EF Core 9 — PostgreSQL (recommended), SQLite, or SQL Server |
| Hosting | IIS (InProcess) with gMSA service account |

---

## Prerequisites

- **Windows Server** (2022 or later) with IIS installed
- **No separate .NET runtime required** — the release ZIP is a self-contained win-x64 build
- A **group Managed Service Account (gMSA)** — or a standard Windows service account — with WinRM access to the cluster nodes
- RSAT tools on the app server (installed by `scripts\Setup-Prerequisites.ps1`): `RSAT-Hyper-V-Tools` and `RSAT-Clustering-PowerShell`
- An **Entra ID app registration** with `groupMembershipClaims: SecurityGroup` and HTTPS redirect URIs — OR Windows Authentication if using the WinAuth site
- WinRM access from the app server to each cluster node (port 5985 HTTP or 5986 HTTPS)

See [docs/QUICK-START.md](docs/QUICK-START.md) for a step-by-step guide from zero to a running portal.

---

## Getting Started

### Quickest path

1. Download the latest release ZIP from the [Releases](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/releases) page
2. Extract and edit `appsettings.json` (Entra credentials, group Object IDs, database connection string)
3. On the app server, run `scripts\Setup-Prerequisites.ps1` then `scripts\Setup-IIS.ps1`
4. Browse to `https://<your-server>` and sign in

See [docs/QUICK-START.md](docs/QUICK-START.md) for full step-by-step instructions.  
See [docs/IIS-Deployment.md](docs/IIS-Deployment.md) for gMSA setup, HTTPS binding, Entra registration, and troubleshooting.

### Windows Authentication (second site)

An optional second IIS site (`HCIPortalWinAuth`) can be configured for internal users on the domain,
bypassing Entra sign-in entirely. Run `scripts\Setup-IIS-WinAuth.ps1` after the primary site is working.
Browser SSO applies — domain users are signed in automatically with their Windows credentials.

### Upgrading

Download the new release ZIP, extract, and from the `scripts` folder on your dev machine:

```powershell
.\Deploy-ToIIS.ps1 -Credential (Get-Credential)
```

**No manual database migration steps are needed.** The app applies all schema changes
automatically on startup using safe, idempotent `ADD COLUMN IF NOT EXISTS` migrations — new columns
are added without touching existing data. Simply deploy and recycle the app pool.

---

## Configuration

Key settings in `appsettings.json` on the server:

```json
{
  "AzureAd": {
    "TenantId":     "<your-entra-tenant-id>",
    "ClientId":     "<your-app-registration-client-id>",
    "ClientSecret": "<client-secret>"
  },
  "Groups": {
    "HciRead":    "<entra-group-object-id>",
    "HciOperate": "<entra-group-object-id>",
    "HciAdmin":   "<entra-group-object-id>"
  },
  "Database": {
    "Provider":         "PostgreSQL",
    "ConnectionString": "Host=127.0.0.1;Database=hciportal;Username=hci_app;Password=CHANGE_ME"
  },
  "DataProtection": {
    "KeyPath": "C:\\apps\\hci-portal-data\\dp-keys"
  }
}
```

Additional settings (ARM integration, SMTP, Teams webhook, Key Vault) are managed through
**Admin → Settings** in the UI after first sign-in and are stored encrypted in the database.

---

## Database Providers

| Provider | When to use | Notes |
|---|---|---|
| **PostgreSQL** (recommended) | Production; multi-server | No row-count limits; free and open source. Run `scripts\Setup-PostgreSQL.ps1` or configure manually. Set `DataProtection:KeyPath` to a persistent folder. |
| **SQLite** | Single-server / low-traffic | Simple; single file. Connection string: `Data Source=C:\apps\hci-portal-data\app.db`. Key path is derived from the DB file location automatically. |
| **SQL Server** | Enterprise / Azure SQL | Standard SQL Server connection string. Set `DataProtection:KeyPath` explicitly. |

---

## Authentication Modes

| Mode | Setup | Use case |
|---|---|---|
| **Entra ID SSO** (primary site) | `Setup-IIS.ps1`; requires Entra app registration and HTTPS redirect URIs | External / multi-tenant access; browser sign-in via Microsoft account |
| **Windows Authentication** (second site) | `Setup-IIS-WinAuth.ps1`; requires domain-joined browser | Internal LAN / domain users; transparent SSO — no sign-in prompt |

Both sites connect to the same database and cluster list.

---

## Access Levels

Three Entra security groups control base access:

| Group | Access |
|---|---|
| **HciRead** | View all data |
| **HciOperate** | View + perform VM/node/role operations |
| **HciAdmin** | Full access including cluster admin, audit log, settings, RBAC management, and alerting |

Fine-grained RBAC — per resource type, per named resource (glob pattern), per operation — is
configured via **Admin → Roles** after signing in as HciAdmin. When no role assignments exist
the app runs in pass-through mode and only the three group policies apply.

---

## Fleet Status — Saved Views

The Fleet Status Board (`/`) supports three layers of cluster filtering that persist across
sessions and can be shared across teams.

### User Favourites

Any signed-in user can **pin clusters** by clicking the pin icon on a cluster row. Pinned
clusters are saved to the database per user. When at least one cluster is pinned, a
**Favourites** pill appears in the filter bar — clicking it shows only your pinned clusters.
Unpin by clicking the pin icon again.

### Personal Named Views

Click **+ View** in the filter bar to create a personal named cluster filter. Three match
types are available:

| Match type | Example | Behaviour |
|---|---|---|
| **Explicit** | *(checkbox list)* | Hand-pick clusters by name — exact match only |
| **Wildcard** | `PROD-*` | Glob pattern — `*` matches any sequence, `?` matches one character |
| **Regex** | `^(PROD\|DR)-` | .NET regular expression matched case-insensitively |

A live **match count** preview is shown while typing a wildcard or regex pattern. Once saved,
the view appears as a labelled pill. Click to filter; click again (or click All Clusters) to
clear. Delete a personal view by clicking `x` on its pill.

Personal views are private — only the creating user can see them.

### Admin — Shared Views (`/admin/views`)

HciAdmin users (and operators with `SharedViews / Configure` RBAC permission) can create
**shared views** that appear as view pills for other users:

| Scope | Who sees the pill |
|---|---|
| **Global** | All signed-in users |
| **Group** | Only members of a specific Entra security group (matched by Object ID) |

Shared views support the same three pattern types as personal views. They are managed via
**Admin → Shared Views** and take effect immediately — no app restart or cache flush required.

**Typical use cases:**

- Create a `Production` global view with `PROD-*` so every operator sees a pre-filtered
  production-only fleet from the moment they sign in.
- Create per-team group views (`Sydney DC`, `DR Sites`) mapped to each team's Entra group OID
  so each team sees only their clusters by default.
- Admins can layer personal views on top of shared ones — the filter bar shows shared views,
  Favourites, and personal views all in one row.

**RBAC for Shared Views:**

| Permission | Required for |
|---|---|
| `SharedViews → View` | Seeing the `/admin/views` page |
| `SharedViews → Configure` | Creating, editing, and deleting shared views |

HciAdmin users always have both. Grant `SharedViews / Configure` to HciOperate users who
should manage views without needing full admin access.

---

## Building and Testing

```powershell
dotnet build AzureLocal.ClusterTool.Web.csproj --configuration Release
dotnet test Tests/AzureLocal.ClusterTool.Web.Tests.csproj
```

318 tests — unit (services + models), bUnit component tests, and `WebApplicationFactory` HTTP pipeline integration tests. No cluster connection required.

---

## Roadmap

The full roadmap — including planned features, community-requested ideas, and items under investigation — lives in **[ROADMAP.md](ROADMAP.md)**.

No fixed timelines. Items move from *Planned* → *In Progress* → shipped in the [Changelog](CHANGELOG.md).  
For detailed implementation notes and investigation logs see [`.github/BACKLOG.md`](.github/BACKLOG.md).

> **Have an idea?** [Open a feature request →](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/new?template=feature_request.yml)

---

## Known Gaps

- **VM creation** is not supported — the tool manages existing VMs only; use Windows Admin Center or PowerShell to provision new VMs
- **Live Update Monitor** has not been fully validated against an active in-progress update run

---

## Documentation

- [Quick Start](docs/QUICK-START.md) — zero to running in ~30 minutes
- [User Guide](docs/USER-GUIDE.md) — feature walkthroughs for operators and administrators
- [IIS Deployment Guide](docs/IIS-Deployment.md) — full server setup, gMSA, HTTPS, Entra registration, troubleshooting

---

## License

Proprietary — all rights reserved.