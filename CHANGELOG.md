# Changelog

All notable changes to the Azure Local Cluster Tool — Web are documented here.

---

## v0.9.0-beta — 2026-03-30

First public beta release. The application covers all major Azure Local operational
areas and is considered stable for day-to-day cluster management. This beta release
is intended to gather feedback from a wider group of users before the v1.0.0 general
availability release.

### Features

#### Virtual Machines
- Full VM inventory across all cluster nodes with live WinRM state
- Start, Stop, Force Stop, Restart, Suspend operations
- Live migration between nodes with node picker dialog
- Inline network adapter expansion (MAC, switch, VLAN, IP addresses)
- Configure CPU count and memory settings via modal dialog
- VM Performance page — per-VM CPU%, memory, VHD IOPS/latency/throughput, network in/out
  (sourced from `Get-ClusterPerf`)
- Virtual Switches — Hyper-V vSwitch inventory per node including SET teaming and adapter details

#### Cluster Nodes
- Node health, uptime, OS build, fault domain, dynamic weight
- Pause/Drain, Resume, and Failback operations with confirmation
- Combined state/drain display (e.g. "Paused (InProgress)")

#### Cluster Roles
- All cluster resource groups (VMs, roles, generic applications)
- Start, Stop, and Move (failover to selected node) operations

#### Cluster Info
- Quorum configuration and witness resource
- Storage Spaces Direct health and pool status
- Active health faults with severity and recommended actions
- Cluster Shared Volumes list and state

#### Storage
- Storage pools: health, size, allocated, provisioning type
- Virtual disks: health, size, footprint, resiliency, provisioning type
- Physical disks: media type, bus type, usage, health, capacity, firmware

#### Storage QoS & Performance
- Storage QoS volume metrics (IOPS, throughput, latency, policy status)
- Per-node performance (CPU%, memory used/total, network in/out)
- Per-CSV volume performance (read/write IOPS, latency, throughput)

#### Network
- Physical network adapters per node with MAC, speed, RDMA, VLAN
- Network ATC intents: traffic types, provisioning and health status, retry action
- Cluster networks: role, state, subnets
- SMB network health including SMB signing and encryption status
- Logical Networks (ARM-sourced via Azure Arc)

#### Events
- Windows cluster event log with level, source, message
- CSV export of filtered event list

#### Remote Log Viewer
- Browse directories and open log files on any cluster node
- Tail mode with auto-refresh and configurable interval
- Floating overlay panel for viewing logs while navigating the rest of the app

#### Solution Updates
- Available solution updates and update run history
- Start a solution update with confirmation
- Live update run monitor — streams action plan progress in real time

#### Azure Arc
- Arc registration status, billing model, hardware class, Windows Server Subscription toggle
- Arc Machines — hybrid compute machine inventory with hardware details
- Arc Extensions — per-machine extension list with version, provisioning state, and available upgrades
- Cluster Extensions — cluster-level Arc extension status and upgrade management

#### Arc Resource Bridge
- Arc appliance health and provisioning state
- Azure Local Sites list (ARM-sourced)

#### Custom Locations & AKS
- Azure Arc Custom Locations inventory (ARM-sourced)
- AKS on Azure Local — provisioned cluster overview including node pools, networking, RBAC,
  and OIDC settings (ARM-sourced)

#### Agent Services
- HCI agent service status per node (wssdagent, mochostagent, and related)
- Start, Stop, Restart operations on individual services

#### Fleet Status Board
- Multi-cluster dashboard showing VM count, node health, health faults, and update status
  across all registered clusters simultaneously — zero WinRM connections, DB-only reads

#### Background Collector
- Automatic background polling of VMs, Nodes, ClusterInfo, and ClusterRoles on a schedule
- Priority-queue poller with per-cluster circuit breaker and configurable poll intervals
- Pages load instantly from DB cache; force-refresh available for live data
- Stale-data banner when the collector falls behind
- Admin Diagnostics page showing collector health per cluster and full PS/WinRM call log

#### Authentication & Access Control
- Entra ID SSO — cookie-based OIDC session, no app passwords
- Three tier group policies: HciRead (view), HciOperate (view + mutate), HciAdmin (full)
- Fine-grained custom RBAC — named roles with per-resource-type, per-named-resource,
  per-operation permissions mapped to Entra security groups
- Custom RBAC works in pass-through mode by default (no configuration required)
- Role badge shown in top bar for signed-in user

#### Admin
- Multi-cluster CRUD management with encrypted credential storage (gMSA or Azure Key Vault)
- Full audit log of all mutating operations with CSV export and purge
- Encrypted application settings (ARM auth mode, SPN credentials) stored in the database
- Custom RBAC role management with Entra group assignments

#### Alerting (Phase 1)
- Rules engine — define alert rules per cluster, per rule type, with severity and thresholds
- Delivery: Teams webhook and SMTP email
- Alert history log with cooldown enforcement
- Maintenance windows — suppress alerts during planned maintenance periods

#### Deployment & Operations
- IIS InProcess hosting with gMSA service account (Kerberos, no password rotation)
- Optional Windows Authentication site for networks without Entra cloud access
- SQLite (default), PostgreSQL, or SQL Server database — switch via `appsettings.json`
- Self-contained `win-x64` publish — no .NET runtime required on server
- GitHub Actions CI (every push to main) and release workflow (tag-triggered)

### Known Limitations (beta)

| Limitation | Notes |
|---|---|
| VM creation not supported | The tool manages existing VMs only; use Windows Admin Center or scripts to create VMs |
| SQL Server / Azure SQL not end-to-end tested | SQLite and PostgreSQL are validated; SQL Server requires testing against a live cluster with an actual database |
| Live update monitor ECE streaming unvalidated | The streaming path (`Start-MonitoringActionplanInstanceToComplete`) has not been tested with an in-progress update; snapshot path is confirmed working |
| Security & Compliance page absent | BitLocker, WDAC, Credential Guard status planned for v1.0 |

### Upgrading from a Previous Preview Install

If upgrading from a pre-background-collector build:

1. Run the SQLite migration snippet from `README.md` to add the three collector tables
2. Run `Deploy-ToIIS.ps1` to publish the new build
3. Recycle the IIS app pool

No action needed for fresh installs — `EnsureCreated()` builds the full schema automatically.

---

*Previous versions were internal preview builds and are not documented here.*
