# Azure Local Cluster Tool — Web

> **Beta v0.9.0** — This release is available for wider testing. Please report bugs and
> feedback via [GitHub Issues](https://github.com/jasontayler/AzureLocal-ClusterTool-Web/issues).

A **Blazor Server** web application for managing **Azure Stack HCI (Azure Local)** clusters and Hyper-V hosts from any browser. Provides VM operations, node management, storage monitoring, network (ATC intents), solution updates, Azure Arc status, and more — without needing Remote Desktop or multiple PowerShell windows.

> **Companion desktop app:** [AzureLocal_Cluster_ManagementTool](https://github.com/jasontayler/AzureLocal_Cluster_ManagementTool) — a standalone WPF version of the same tool.

---

## Features

| Area | Capabilities |
|---|---|
| **Virtual Machines** | List, Start, Stop, Force Stop, Restart, Suspend, Live Migrate |
| **Cluster Nodes** | Live stats, Pause/Drain, Resume, Failback |
| **Cluster Roles** | List, Start, Stop, Move (failover to node) |
| **AKS on Azure Local** | Provisioned cluster overview (ARM-sourced) |
| **Cluster Info** | Quorum, S2D status, health faults |
| **Storage** | Pools, virtual disks, physical disks, Storage QoS volumes |
| **Network** | Physical adapters, ATC intents + status, cluster networks, logical networks (ARM), SMB health |
| **Events** | Cluster event log with CSV export |
| **Remote Log Viewer** | Browse and read log files directly from cluster nodes |
| **Solution Updates** | List updates and runs, start an update, live streaming monitor |
| **Azure Arc** | Registration status, Arc Machines, Arc Extensions, Cluster Extensions |
| **Arc Resource Bridge** | Appliance status + Azure Local Sites |
| **Custom Locations** | ARM-sourced custom location inventory |
| **Agent Services** | View and control HCI agent services (wssdagent / mochostagent) |
| **Admin — Clusters** | Multi-cluster CRUD management |
| **Admin — Audit Log** | Full audit trail of all mutating operations, CSV export, purge |
| **Admin — Settings** | Encrypted app settings (ARM auth mode, SPN credentials) |
| **Admin — Roles (RBAC)** | Fine-grained per-resource-type access control via Entra security groups |
| **Background Collector** | Automatic background polling — VM, node, cluster state written to local DB; pages load instantly from cache; amber stale banner + force-refresh when collector falls behind |

---

## Technology Stack

| Item | Value |
|---|---|
| Framework | .NET 10.0 Blazor Server |
| Authentication | Microsoft Entra ID SSO (`Microsoft.Identity.Web`) |
| Authorisation | Entra security groups + fine-grained custom RBAC |
| PowerShell | `Microsoft.PowerShell.SDK` 7.5.4 — WinRM via `WSManConnectionInfo` |
| Database | EF Core 9 — PostgreSQL (recommended), SQLite, or SQL Server |
| Hosting | IIS (InProcess) with gMSA service account |

---

## Prerequisites

- Windows Server with IIS and the [ASP.NET Core Hosting Bundle](https://dotnet.microsoft.com/en-us/download/dotnet/10.0) (.NET 10)
- A [group Managed Service Account (gMSA)](docs/IIS-Deployment.md) — or a standard Windows service account
- An **Entra ID app registration** with `groupMembershipClaims: SecurityGroup` and redirect URIs configured
- WinRM access from the app server to the cluster nodes (HTTP 5985 or HTTPS 5986)

---

## Getting Started

1. **Download** the latest ZIP from the [Releases](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/releases) page and extract it to your IIS server
2. **Edit** `AzureLocal.ClusterTool.Web/appsettings.json` — fill in `AzureAd`, `Groups`, and `Database` settings
3. **Edit** `clusters.json.example` → rename to `clusters.json` and add your cluster(s)
4. **Run** the setup script and deploy — see [docs/QUICK-START.md](docs/QUICK-START.md) for a step-by-step guide

### Deploy to IIS

```powershell
# First-time server setup (run as Domain Admin on the IIS server)
.\scripts\Setup-IIS.ps1 -SiteName "HCIPortal" -AppPoolName "HCIPortalPool" `
    -PhysicalPath "C:\apps\hci-portal" -HostHeader "azlocalmgmt.yourdomain.com"

# Deploy from dev machine
cd scripts
.\Deploy-ToIIS.ps1 -Credential (Get-Credential)
```

See [docs/IIS-Deployment.md](docs/IIS-Deployment.md) for the full setup guide including gMSA configuration, HTTPS binding, and Entra app registration.

#### Upgrading an existing installation

Extract the new ZIP over your existing deployment folder, then recycle the app pool. The app uses `EnsureCreated` on startup — schema changes are applied automatically on a new database. For an existing database, the app logs any schema differences to the event log; see [docs/IIS-Deployment.md](docs/IIS-Deployment.md) for upgrade notes.

For PostgreSQL (the recommended provider), no manual schema steps are needed — `EnsureCreated` is safe to run against an existing database and only applies missing objects.

---

## Configuration

Key settings in `appsettings.json` on the server:

```json
{
  "AzureAd": {
    "TenantId":     "<your-tenant-id>",
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
    "ConnectionString": "Host=localhost;Database=hciportal;Username=hci_app;Password=CHANGE_ME"
  }
}
```

**PostgreSQL** is the recommended database. Install PostgreSQL on the app server (or any accessible server), create a database and user, then fill in the connection string above. The app creates all tables automatically on first startup — no migrations needed.

Supported providers: `PostgreSQL` (recommended), `SqlServer`, `Sqlite`.
```

---

## Access Levels

| Group | Access |
|---|---|
| **HciRead** | View all data |
| **HciOperate** | View + perform VM/node/role operations |
| **HciAdmin** | Full access including cluster admin, audit log, settings, and RBAC management |

Fine-grained RBAC (per resource type, per named resource, per operation) can be configured via **Admin → Roles** once signed in as HciAdmin.

## Beta Feedback

This is a beta release. Please use [GitHub Issues](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues) to report bugs or suggest improvements. Use the provided issue templates — they include an area selector and version field which helps track down issues quickly.

Known gaps in this release:
- VM creation is not supported — existing VMs are managed only
- SQL Server / Azure SQL providers have not been end-to-end tested against a live cluster
- SQLite is supported but not recommended for production — use PostgreSQL

---

## Documentation

- [User Guide](docs/USER-GUIDE.md) — feature walkthroughs for operators and administrators
- [IIS Deployment Guide](docs/IIS-Deployment.md) — full server setup, gMSA, HTTPS, troubleshooting

---

## License

Proprietary — all rights reserved.