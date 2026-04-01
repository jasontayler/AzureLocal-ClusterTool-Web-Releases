# Changelog

All notable changes to the Azure Local Cluster Tool — Web are documented here.

---

## v0.9.6-beta — 2026-04-01

### Bug Fixes

- **Virtual Machines page missing VMs** — `GetVirtualMachinesAsync` previously ran `Get-VM`
  via the cluster RunspacePool, which executes on whichever single node owns the WinRM
  endpoint at the time. VMs hosted on other nodes were never returned. The method now
  performs a two-phase query: first `Get-ClusterNode` via RunspacePool to discover all node
  names, then parallel `Get-VM` calls to each node directly via `GetOrCreateCachedRunspace`.
  Results are merged and deduplicated by VM GUID. A failed node is logged as a warning and
  skipped — remaining nodes still return their VMs. Falls back to the cluster address for
  standalone Hyper-V hosts where `Get-ClusterNode` is unavailable.

- **SignalR disconnects every 30-90 seconds** — Blazor Server `KeepAliveInterval` was 15 s
  (default). Corporate proxies and load balancers commonly have a 30-60 s WebSocket idle cut.
  Reduced to 10 s pings with `ClientTimeoutInterval` extended to 60 s. Added
  `DisconnectedCircuitRetentionPeriod = 5 min` so brief disconnects restore page state rather
  than forcing a cold reload. IIS app pool `idleTimeout` set to 0 in `Setup-IIS.ps1` to
  prevent IIS killing the worker process during quiet periods (which dropped all circuits).

- **Per-node WinRM `PSRemotingTransportException` on node actions** — The per-node runspace
  cache TTL was 5 minutes. WinRM's default server-side shell idle timeout is 3 minutes, so
  cached runspaces frequently expired server-side while still appearing `Opened` client-side.
  TTL reduced to 90 s (safely below the 3-minute server kill). Added `PSRemotingTransportException`
  retry in `Nodes.razor` `RunNodeActionAsync`: evicts broken pool entry and retries once with a
  fresh connection, transparent to the user.

- **Empty node name crash** — `PauseClusterNodeAsync`, `ResumeClusterNodeAsync`, and
  `FailbackClusterNodeAsync` could be called with an empty node name when a `ClusterNode`
  snapshot entry had no `Name` value. All three now guard against empty names and return
  early with a warning log rather than firing `Suspend-ClusterNode ""` at the cluster.

- **PostgreSQL `FATAL: canceling authentication due to timeout`** — Npgsql connection pool
  was bulk-reconnecting after a prior transient failure invalidated idle connections.
  Added `Keepalive=60;Connection Idle Lifetime=300;Timeout=30;Command Timeout=60` to all
  PostgreSQL connection string defaults. `Keepalive=60` sends heartbeats every 60 s,
  preventing idle connections from expiring silently and triggering bulk reconnects.

- **PostgreSQL `Host=localhost` IPv6 fallback on Windows** — Windows resolves `localhost` to
  `::1` (IPv6) first. Changed all default connection strings to `Host=127.0.0.1` to force
  direct IPv4 and eliminate `EventId 20004` noise from IPv6 retry attempts.

---

## v0.9.5-beta — 2026-03-31

### Performance

- **Parallel background collector** — The snapshot collector now runs all due data types for a
  cluster concurrently instead of serially. A per-cluster visit previously took up to 40-50 s
  because ~10 WinRM calls executed one after another. They now fire simultaneously via
  `Task.WhenAll`, bounded by a `RunspacePool` of up to 4 concurrent connections, reducing a
  full visit to ~6-8 s (bounded by the slowest single call, typically `Get-SolutionUpdate`).

  Failure handling improved alongside the parallelisation:
  - A single flaky type (e.g. `Get-NetIntentStatus`) no longer blocks VMs, Nodes, and Storage
    from being collected on the same visit.
  - Partial success (some types land, some fail) resets `ConsecutiveFails` to 0 — the circuit
    breaker only opens when **all** types fail on a visit, same as before.
  - Admin > Collector Health now shows an **orange "Partial"** status (instead of a misleading
    green "OK") when at least one type failed on the last visit.

### Bug Fixes

- **AzureArmService — incorrect RP in `listUserKubeconfig` 403 error** — The error message and
  code comment cited `Microsoft.HybridContainerService` as the required resource provider
  permission. The correct action is
  `Microsoft.Kubernetes/connectedClusters/provisionedClusterInstances/listUserKubeconfig/action`.
  The message now accurately names the right RP and notes that the built-in
  "AKS Arc Cluster User Role" does **not** cover this action (it targets the HybridContainerService
  RP instead). A custom role with the `Microsoft.Kubernetes` action is required.

---

## v0.9.4-beta — 2026-03-31

### Performance

- **Snapshot-first data loading across all cluster pages** — Four additional pages were calling
  live WinRM commands on every page load when the background snapshot poller already had
  current data available:

  | Page | Live call replaced | Estimated saving |
  |---|---|---|
  | Nodes | `Get-AzureStackHCI` (Arc portal URL) | ~3 s |
  | Cluster Info | `Get-SolutionUpdate` + `Get-AzureStackHCI` | ~5-15 s |
  | Arc Registration | `Get-AzureStackHCI` (full page data source) | ~3 s |
  | Cluster Storage | `Get-ClusterNode` (move-volume dropdown) | ~1-2 s |

  All four now read from the snapshot cache first. `ArcRegistration` falls back to a live
  WinRM call only when the snapshot is empty (e.g. first load before the poller has run).
  `ClusterInfoPage` adds both snapshot tasks to the `Task.WhenAll` block so DB reads race
  in parallel with the WinRM pool connection.

- **Solution Updates page** (`a8b04ea`) — Three bottlenecks fixed in the previous release:
  - `Get-SolutionUpdateEnvironment -FullHealthCheckDetails` (30-120 s) removed from
    auto-load; now on-demand via "Check Health" button only.
  - Arc info sourced from snapshot instead of live `Get-AzureStackHCI`.
  - `GetSolutionUpdateRunsAsync` reduced from two `Get-SolutionUpdate` PS calls to one.

---

## v0.9.2-beta — 2026-03-31

### Bug Fixes

- **Deploy script — `appsettings.WinAuth.json` overwrite** — The deployment script incorrectly
  replaced `appsettings.WinAuth.json` on the server whenever all three Entra group IDs were empty.
  Empty groups is a valid intentional configuration (ForcePassThrough mode — all authenticated
  domain users have access). The file is now only replaced when the server still holds the stale
  `BUILTIN\Administrators` default from an older install. All other server-side versions,
  including intentionally empty groups, are preserved as-is.

- **Deploy script — `appsettings.Production.json` not loaded** — ASP.NET Core only loads
  `appsettings.Production.json` when `ASPNETCORE_ENVIRONMENT=Production` is set on the IIS app
  pool. Without it, production settings (group IDs, DB connection string, ARM credentials) are
  silently ignored even when the file exists. A new idempotent step in `Deploy-ToIIS.ps1` (step 7a)
  checks and sets this environment variable on `HCIPortalPool` via PS remoting on every deploy.
  Existing installs that ran Setup-IIS.ps1 after v0.9.1-beta already have this set; the step is a
  no-op for them.

### New Features

- **Deleted Cluster Data Retention setting** — A new admin setting controls how long snapshot data
  (LatestSnapshot and SnapshotHistory rows) is kept after a cluster is removed from the admin page.
  - Default `0` — data is purged immediately when the cluster is deleted (keeps the database lean).
  - Values `1 / 3 / 7 / 14 / 30` — orphaned rows are cleaned up by the nightly prune after the
    configured number of days (useful if you want to retain recent history temporarily after
    decommissioning a cluster).
  - Configured at **Admin > Settings** under the "Background Collector" group.

---

## v0.9.1-beta — 2026-03-30

### Bug Fixes

- **Security / Drift Detection** — Drift detection (`Invoke-AzStackHciVSRDriftDetectionValidation`)
  is only available on clusters running solution update 2602 or later. The app now checks the
  installed release version before attempting a drift check. Clusters on older releases display a
  clear message instead of a raw PowerShell "cmdlet not found" error. The auto-run on page load
  is skipped entirely for clusters below 2602.

- **Windows Auth / AD Groups** — Fine-grained RBAC (custom roles with AD group SIDs) did not
  work on the Windows Auth site. IIS provides group membership as `groupsid` claims but all pages
  read `groups` claims. A middleware layer now maps `groupsid` values to `groups` on every
  authenticated request so RBAC checks and cluster visibility work correctly with AD group SIDs.

- **Setup-IIS.ps1** — The Entra ID site setup script now sets `ASPNETCORE_ENVIRONMENT=Production`
  on the IIS app pool so that `appsettings.Production.json` is loaded automatically.

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
