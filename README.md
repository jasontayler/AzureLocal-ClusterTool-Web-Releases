# Azure Local Cluster Tool — Web

Browser-based management for **Azure Stack HCI (Azure Local)** clusters and Hyper-V hosts.
Manage VMs, nodes, storage, networking, solution updates, Azure Arc, and more —
from any browser, without Remote Desktop or multiple PowerShell windows.

> **Beta v0.9.0** — available for wider testing.
> Please use [GitHub Issues](../../issues) to report bugs or share feedback.
> Bug report and feature request templates are provided — they include area/version fields which help track down issues quickly.

---

## Download

Get the latest release from the [Releases](../../releases) page.
Download the `AzureLocal.ClusterTool.Web_v*.zip` file and follow the Quick Start below.

---

## Quick Start

Full step-by-step guide: [docs/QUICK-START.md](docs/QUICK-START.md)

**Summary (30 minutes, first time):**

1. **Extract** the ZIP to a folder on your Windows Server
2. **Create an Entra ID app registration** — see Quick Start guide for the 5-minute setup
3. **Edit** `AzureLocal.ClusterTool.Web\appsettings.json` — fill in TenantId, ClientId, Groups, Database
4. **Rename** `clusters.json.example` → `clusters.json` and add your cluster addresses
5. **Run** `scripts\Setup-IIS.ps1` as Domain Admin (first time only)
6. **Browse** to `https://<your-server>` and sign in with your Entra ID account

**Upgrading an existing install:**
```powershell
cd scripts
.\Deploy-ToIIS.ps1 -Credential (Get-Credential)
```

---

## Features

| Area | Capabilities |
|---|---|
| **Virtual Machines** | List, Start, Stop, Force Stop, Restart, Suspend, Live Migrate, Configure CPU/Memory |
| **VM Performance** | Per-VM CPU%, memory, VHD IOPS/latency/throughput, network (from Get-ClusterPerf) |
| **Virtual Switches** | Hyper-V vSwitch inventory per node with SET teaming and adapter details |
| **Cluster Nodes** | Live stats, OS build, Pause/Drain, Resume, Failback |
| **Cluster Roles** | List, Start, Stop, Move (failover) |
| **Cluster Info** | Quorum, S2D health, health faults, Cluster Shared Volumes |
| **AKS on Azure Local** | Provisioned cluster overview — node pools, networking, RBAC, OIDC (ARM-sourced) |
| **Storage** | Pools, virtual disks, physical disks health and capacity |
| **Storage QoS & Perf** | Volume IOPS/throughput/latency, node CPU/memory/network, CSV volume perf |
| **Network** | Physical adapters, ATC intents + status, cluster networks, SMB health, logical networks |
| **Events** | Cluster event log with CSV export |
| **Remote Log Viewer** | Browse and read log files on cluster nodes; tail mode with auto-refresh |
| **Solution Updates** | List updates and runs, start an update, live streaming run monitor |
| **Azure Arc** | Registration status, Arc Machines, Arc Extensions, Cluster Extensions |
| **Arc Resource Bridge** | Appliance health and Azure Local Sites |
| **Custom Locations** | ARM-sourced custom location inventory |
| **Agent Services** | HCI agent service status and control (wssdagent, mochostagent) |
| **Fleet Status Board** | Multi-cluster dashboard — VM/node/health/updates across all clusters, zero WinRM |
| **Background Collector** | Automatic background polling — pages load instantly from DB cache |
| **Alerting** | Rules engine with Teams webhook + SMTP email delivery, history, maintenance windows |
| **Admin — Clusters** | Multi-cluster CRUD with encrypted credential storage |
| **Admin — Audit Log** | Full audit trail of all mutating operations, CSV export |
| **Admin — Settings** | Encrypted app settings (ARM auth mode, SPN credentials) |
| **Admin — RBAC** | Fine-grained per-resource-type, per-named-resource access control via Entra groups |

---

## Requirements

| Requirement | Notes |
|---|---|
| Windows Server 2022+ with IIS | |
| .NET runtime | **Not required** — self-contained build |
| Service account | gMSA (recommended) or standard Windows account with WinRM access to cluster nodes |
| WinRM | Port 5985 (HTTP) or 5986 (HTTPS) from app server to each cluster node |
| Entra ID app registration | Free — takes 5 minutes — see [Quick Start](docs/QUICK-START.md) |
| Database | SQLite (default, zero config) · PostgreSQL · SQL Server |

---

## Documentation

| Document | Description |
|---|---|
| [Quick Start](docs/QUICK-START.md) | Step-by-step first-time install guide (~30 min) |
| [User Guide](docs/USER-GUIDE.md) | Feature walkthroughs for operators and administrators |
| [IIS Deployment Guide](docs/IIS-Deployment.md) | Full server setup: gMSA, HTTPS, database, troubleshooting |
| [Changelog](CHANGELOG.md) | Release notes for each version |

---

## Beta Feedback

This is a beta release. Found a bug? Have a suggestion?

- [Report a bug](../../issues/new?template=bug_report.yml)
- [Request a feature or share feedback](../../issues/new?template=feature_request.yml)

Known gaps in this beta:
- VM creation is not supported — existing VMs are managed only
- SQL Server / Azure SQL not yet end-to-end tested against a live cluster
- Security & Compliance page (BitLocker, WDAC, Credential Guard) planned for v1.0

---

## License

Proprietary — all rights reserved.
