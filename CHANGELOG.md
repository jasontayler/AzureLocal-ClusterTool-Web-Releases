# Changelog

All notable changes to the Azure Local Cluster Tool — Web are documented here.


## v0.10.13 — 2026-04-30

### Bug Fixes

- **Fleet VM Status — VM row click now navigates to the VM directly** — when searching for a
  VM on the Fleet VM Status page (`/status/vms`) and clicking a result row, the app was
  navigating to the full cluster VM list (`/clusters/{name}/vms`) instead of the individual
  VM. The link on each VM row now targets `/clusters/{name}/vms/{vmName}` directly so the
  correct VM detail page opens immediately.

---

## v0.10.2 — 2026-04-15

### Setup & Deployment

**Setup-Prerequisites.ps1 — Server 2025 compatibility**
- IIS features now installed via DISM — faster and more reliable on Server 2025 than `Install-WindowsFeature`
- Already-installed IIS features are skipped on re-run (idempotent)
- PowerShell 7 now downloaded as a direct MSI instead of via winget — `--scope machine` is unsupported on Windows Server
- ASP.NET Core Hosting Bundle now downloaded directly instead of via winget for the same reason
- Clarified offline install path when Hosting Bundle download fails
- Fixed kubelogin version check — `$Matches` was undefined when `--version` writes to stderr
- `--source winget` added to remaining winget calls to avoid source-selection prompts
- Printed connection strings simplified — redundant Npgsql parameters removed from output
- Now detects when running on the app server itself and skips the loopback `Invoke-Command` path that caused credential errors

**Setup-IIS.ps1 / Setup-IIS-WinAuth.ps1 — ANCM detection fix**
- ASP.NET Core Module V2 check now uses `Get-ItemProperty` on the correct registry key and checks the `Install` DWORD value — `Test-Path` returns `$false` on some Server 2025 builds even when the Hosting Bundle is installed, causing a false "not installed" error

**Setup-IIS.ps1 — gMSA setup**
- Added `Install-ADServiceAccount` + `Test-ADServiceAccount` before setting the app pool identity — without this step WAS cannot retrieve the managed password and disables the pool with HTTP 503
- gMSA is now automatically added to the local Administrators group — required for WinRM connections to cluster nodes
- `-serviceAccountType` parameter now defaults to empty and requires an explicit value of `gMSA` or `Standard` — previously defaulted to `"gMSA"`, silently accepting a garbage value if the account name was passed in the wrong parameter

**Install-PostgreSQL.ps1**
- SQL identifiers (database name, username) are now double-quoted in DDL/DCL statements — prevents parse errors when names contain hyphens or other special characters
- Password prompts are now deferred until needed — re-runs where the DB/user already exist no longer prompt for a password unnecessarily

**New-AppServiceAccount.ps1**
- `DNSHostName` now passed to `New-ADServiceAccount` to avoid an interactive prompt on some DC configurations
- Removed empty `PrincipalsAllowedToRetrieveManagedPassword` from the initial call that was causing errors when no nodes were specified

### Documentation
- `QUICK-START.md` restructured — IIS setup (Step 6) now comes before JSON configuration (Step 7); added app pool stop/restart at end of Step 7; `AllowedHosts` added to the `appsettings.Production.json` snippet; App Proxy domain services how-to link added
- `appsettings.template.json` — fixed `DataProtection:KeyPath` to use the correct default data path


## v0.10.1 — 2026-04-14

### Performance

- **VM Detail load time reduced by ~70%** — `GetVMDetailAsync` previously opened 6 separate
  PowerShell calls per VM expand (properties, VHDs, NICs, firmware, integration services,
  snapshots). These are now consolidated into a single call with an internal discriminator
  tag, reducing pool slot consumption from 6 to 1 and cutting VM detail expand time
  significantly.

- **VM Detail migrated to local RSAT + cached CimSession** — `GetVMDetailAsync` now uses the
  local RSAT pool with per-node cached CimSession instead of per-call WinRM runspaces,
  eliminating repeated WinRM handshakes on every VM expand.

- **Collector performance — page-load refresh removed** — cluster data pages previously
  triggered a live background collector refresh on every page load, adding latency and
  competing with the scheduled poller for WinRM connections. Pages now read from the
  snapshot store directly; the background collector runs on its own schedule without
  interference from user navigation.

### Bug Fixes

- **IIS memory leak — PS7 module compatibility with FailoverClusters** — the FailoverClusters
  and Hyper-V modules do not declare `CompatiblePSEditions = Core`. When loaded in PS 7,
  they run via the Windows PowerShell compatibility shim which spawns a `powershell.exe`
  host process. Each evict-and-reconnect cycle spawned a new shim process that was never
  cleaned up. Over a long uptime this accumulated orphaned `powershell.exe` processes,
  exhausted thread pool threads, and eventually caused IIS to stop serving requests.
  The FailoverClusters and Hyper-V cluster mutations have been moved to the WinRM pool
  (running PS 5.1 on the cluster directly) so the shim is never invoked from the
  local PS7 pool.

- **Remote Log Viewer — per-node browsing** — file browsing, reading, and live tail now
  target the selected cluster node directly via `Invoke-Command -ComputerName` rather than
  routing through the cluster VIP. Fixes missing log content when logs are node-local.

- **Remote Log Viewer — node picker defaults to first node** — the node selector now always
  defaults to the first node alphabetically and no longer offers a Cluster VIP option that
  returned inconsistent results depending on which node the VIP resolved to.

- **Virtual Switches — removed from main navigation** — Virtual Switches was incorrectly
  appearing as a top-level nav item. It is a sub-tab within the Virtual Machines page.

- **RBAC — Fleet Status cluster filtering** — the Fleet Status page now applies name-pattern
  filtering so users with cluster-scoped role assignments only see their permitted clusters.

- **RBAC — Fleet VM Status access denial** — denied users previously saw a silent empty VM
  table with no explanation. The page now shows the standard access-denied message.

- **RBAC — Fleet VM Status nav guard** — the Fleet VM Status nav link is now hidden for
  users without VM View access, consistent with all other nav entries.

- **RBAC — Home page policy** — the cluster picker now enforces the HciRead group policy
  instead of bare authentication, consistent with all other pages in the app.

- **Install.ps1 -Upgrade overwrites appsettings.json and appsettings.WinAuth.json** — the
  upgrade robocopy excluded only `appsettings.Production.json` and `clusters.json`. Both
  `appsettings.json` and `appsettings.WinAuth.json` are now also excluded from the upgrade
  copy, preserving any server-side customisations (AllowedHosts, group SIDs, etc.) across
  upgrades.

---

## v0.10.1-rc1 — 2026-04-13

### New Features

- **Maintenance Windows — Edit and Delete** — maintenance windows can now be edited (change
  cluster, times, or reason) or permanently deleted directly from the table. An Edit modal
  pre-populates with the current values; Delete prompts for confirmation. Deactivate is now
  only shown for currently active or scheduled windows — expired windows no longer show it.

- **Maintenance Windows — local time display** — all times on the Maintenance Windows page
  (table and Add / Edit forms) now show the app server's local time. Column headers say
  "Local" rather than "UTC". The `datetime-local` input defaults to current server time.

- **Background Collector Health — local time display** — circuit-breaker and last-success
  times in the Admin Diagnostics panel now show server local time instead of UTC.

### Bug Fixes / Improvements

- **kubelogin minimum version enforced** — `AzureArmService` now checks the installed kubelogin
  version before calling `get-token`. Versions below v0.2.0 contain a nil pointer bug in the
  PoP token cache (MSAL Go v1.4.2) that causes a Go runtime panic (exit code 2) on every call.
  The app now surfaces a clear upgrade message instead of the raw panic stack trace.
  `Setup-Prerequisites.ps1` also checks the version and auto-upgrades via winget if needed.
  Minimum supported version is **v0.2.0**.

- **Maintenance window accumulation** — when a cluster remained offline across multiple
  circuit-breaker cycles, a new maintenance window was created every 10 minutes. The poller
  now extends the existing poller-created window's end time rather than creating a new row.
  One window per cluster, rolling forward.

- **Collector health circuit-open time was UTC** — `StatusLabel` and `LastSuccessDisplay`
  computed properties on `CollectorHealth` now call `.ToLocalTime()` so times appear in
  server local time everywhere they are displayed.

- **`GetStorageNetworkHealthAsync` latency** — two sequential `Get-NetAdapterAdvancedProperty`
  calls (one for Jumbo frames, one for Flow Control) were merged into a single call, reducing
  the SMB health poll time by roughly half.

- **Poller memory leak — unbounded cluster connections** — `ClusterPollerService` previously
  held one `WebHyperVService` open indefinitely per registered cluster. Each instance owns
  two RunspacePools, a CimSession, and a per-node runspace cache. With a large cluster estate
  this exhausted virtual memory and caused the host OS and PostgreSQL to crash overnight.
  After each poll, if the cluster's next scheduled visit is more than 2 minutes away the
  connection is disposed immediately. `GetOrCreatePollerConnectionAsync` reconnects
  transparently on the next cycle. At steady state only `MaxConcurrentPolls` (3) connections
  are live regardless of total cluster count.

- **Poller RunspacePool right-sized** — the poller-tier RunspacePool `maxRunspaces` was
  reduced from 4 to 1. The poller executes collector types sequentially per cluster;
  a single runspace is sufficient and removes 3 idle slots per pool per cluster.

- **Module load race on concurrent connect** — `WebHyperVService._localPool.Open()` triggers
  an `InitialSessionState` startup script that imports Hyper-V, FailoverClusters, and
  NetworkATC. All three modules write to the process-wide PS module load table. When multiple
  instances opened concurrently (poller + page request at startup) this produced:
  `Collection was modified; enumeration operation may not execute`.
  A static `SemaphoreSlim(1,1)` now serialises `_localPool.Open()` across all instances so
  only one module import runs at a time.

- **Admin Diagnostics — poller connection count** — the WinRM tile on Admin > Perf Debug now
  shows two rows: page-tier connections (N / 50 LRU cap) and current poller-tier connections
  (idle-evicted), making the effectiveness of the memory fix directly observable.

- **NetworkATC intents moved from local PS7 pool to WinRM pool** — `GetNetworkIntentsAsync`
  now runs via the WinRM `_pool` instead of `_localPool`. The `Hyper-V` and `FailoverClusters`
  modules use PS7's implicit Windows-PowerShell compatibility shim when their manifests do
  not declare `CompatiblePSEditions = Core`. Every time `_localPool` was evicted and
  recreated, the shim spawned a new `powershell.exe` process that was never cleaned up,
  causing process accumulation proportional to evict+reconnect cycles.

- **Local pool is now a static process-wide singleton** — `WebHyperVService._localPool` now
  points to a single `RunspacePool` that is created once on first `ConnectAsync` and never
  disposed. Previously, every evict+reconnect cycle disposed the old pool and opened a new
  one, re-triggering Windows-PowerShell compatibility-shim initialisation (orphaned
  `powershell.exe` processes) and accumulating unloadable PS type-system state in the
  AppDomain (.NET cannot unload module assemblies at runtime). With the static singleton:
  module imports and any shim processes happen once per app start; the pool is never closed
  so "Closing" state races from `RecreateLocalPool` cannot occur; and all cluster connections
  share up to 20 runspaces so per-connection pool overhead is eliminated entirely.
  
---

## v0.10.0-rc1 — 2026-04-11

### Prerequisites

- **NetworkATC management tools now required on the app server** — the app server must have
  the NetworkATC RSAT tools installed so that `Get-NetIntent` and `Get-NetIntentStatus` can
  be resolved via WinRM. Run once on the app server:
  ```powershell
  Install-WindowsFeature -Name NetworkATC -IncludeManagementTools
  ```

### New Features

- **AKS on Azure Local** — full AKS management section added under each cluster:
  - **Overview** — cluster list with Kubernetes version, upgrade availability, node pool
    summary, OIDC issuer, Azure Monitor status, and an expandable all-pods list.
  - **Workloads** — Deployments, StatefulSets, and DaemonSets with namespace filter, replica
    counts, and container image details.
  - **Resources** — Infrastructure (node pools, control plane), Storage (PVCs/PVs), and
    Networking (services, ingresses) tabs in a single consolidated page.
  - **Pod drilldown** — click any pod to view container details, resource requests/limits,
    readiness/liveness probes, volumes, and environment variables.
  - **Pod logs** — stream live container logs in a scrollable viewer directly in the browser.
  - **Pod restart** — restart individual pods with a single click and audit log entry.
  - **GitOps** — Flux kustomizations and config objects with compliance status, last-applied
    timestamps, expandable detail panels, and a Force Resync action.
  - All AKS live data is fetched via Arc cluster-connect using kubelogin; 5-minute
    stale-while-revalidate cache for all AKS pages.

- **Overview page — CSV and Network tiles** — the cluster Overview now shows:
  - **Cluster Shared Volumes** tiles (short name, owner node, free space) with a Storage link.
  - **Cluster Networks** tiles (role, subnet) with a Network link.
  - **% free space** aggregate sub-line on the existing Cluster Shared Volumes health tile.

- **Cluster Roles — Resource sub-panel** — clicking a role row expands an inline Resources
  panel showing all cluster resources for that role, with per-resource Start / Stop actions
  and audit logging.

- **Agent Services — bulk operations** — multi-select with Select All, floating bulk action
  bar (Start / Restart / Stop Selected), and a confirmation modal listing affected services
  by node.

- **Snapshot collection expanded** — background poller now collects Network Adapters,
  Cluster Networks, and SMB Network Health in addition to existing types; data is available
  immediately from the DB cache on page load.

- **Navigation and workflow improvements** — the sidebar navigation has been significantly
  reorganised to reduce clicks and improve day-to-day operational flow:
  - Cluster navigation is now a flat list of direct links (Disks, Cluster Volumes,
    Performance, Network, etc.) rather than collapsed trees — less clicking to get where
    you need to go.
  - Storage and Network sub-sections use in-page tab bars (SubNav) so you can switch
    between Disks / Volumes / Performance or Adapters / Intents / Networks without leaving
    the page.
  - **Admin** section now groups Clusters, Settings, Audit Log, and Perf Debug together —
    all admin tasks in one place.
  - **Monitoring** section groups Alerts and Maintenance Windows separately from Admin,
    making it clearer which menu items affect operational monitoring vs. configuration.
  - The "All Clusters" link has been removed from the cluster-scoped nav to reduce clutter;
    the cluster name at the top of the sidebar acts as the home/back navigation.
  - The Reports section is hidden when a cluster is selected to keep the nav focused on the
    cluster you are working with.

### Bug Fixes

- **Network ATC intents fail to load** — `Get-NetIntent` / `Get-NetIntentStatus` are now
  executed via WinRM on the cluster node where the NetworkATC DLLs are installed, eliminating
  the "module could not be loaded" error that occurred when running locally on the app server.

- **Volume Performance empty on first load** — `Get-Volume` requires the `Storage` module
  which was not auto-loading in the WinRM runspace. Added explicit
  `Import-Module Storage` before the CSVFS pipeline, so volume performance data populates
  on the first visit without needing a manual Refresh.

- **Cluster Shared Volumes empty** — switched from `_localPool` + `-Cluster` (DCOM, 14-34s,
  unreliable) to `_pool` WinRM for `Get-ClusterSharedVolume`; CSVs now load consistently.

- **SMB Network health false-positive red status** — RDMA health was being evaluated against
  all adapters; corrected to only check adapters that belong to a Storage ATC intent.

- **SMB Network page HTML entity rendered as literal text** — diamond character was written
  as a raw Unicode escape; replaced with the correct HTML entity.

- **Security / Drift Detection** — fixed fault count (now failures-only), added collapsible
  test rows, added BitLocker columns, and hardened the 24-hour minimum detection window.

- **Overview — Arc and AKS cards** — both sections now render immediately from the DB
  snapshot with a loading indicator while ARM data fetches in the background, eliminating
  the blank-then-appear flash.

- **Cluster Info — CSV tiles overlap** — tile names now show the short label extracted from
  inside the parentheses (e.g. `Infrastructure_1`) instead of the full
  `Cluster Virtual Disk (...)` string, fitting within the standard tile width.

- **Cluster Storage — Free/Size column alignment** — the combined "X.X / Y.Y GB" cell has
  been split into separate Free and Size columns aligned consistently with text columns.

- **Nodes page** — VM count column alignment corrected.

- **VirtualSwitches nav link missing** — nav link was omitted; restored.

- **NetworkIntents scheduler** — intents were never polled by the background collector;
  added to the scheduler correctly.

- **Events page** — fixed loading/empty state display.

- **Fault history panel** — was loading in a collapsed state by default; now expanded.

---

## v0.9.13-rc3 — 2026-04-10

### Bug Fixes

- **Solution update health check shows "No failures" despite real failures existing** —
  `GetUpdateHealthCheckAsync` used `Get-SolutionUpdate -Id` to fetch the update, but the
  `-Id` parameter does not accept ARM-sourced ID formats where the cluster stores the update
  as `namespace/PackageName` (e.g. `redmond/Solution12.x`). The cmdlet silently returned
  null, producing an empty result. Fixed to use `Where-Object` with a `-like '*/name'`
  suffix pattern that matches both exact names and namespace-prefixed IDs. Same fix applied
  to the ID lookup in `StartSolutionUpdateAsync` and `StartSolutionUpdatePrepareOnlyAsync`.

- **NullReferenceException when clicking Prepare** — the `Start-SolutionUpdate -PrepareOnly`
  call failed internally because the ID-resolution lookup (`Where-Object $_.ResourceId -eq`)
  was an exact match and did not find the update when the cluster-native ResourceId has a
  namespace prefix (e.g. `redmond/Solution12.x`). Added a `-like '*/name'` fallback to
  all three update-action lookups so the native ID is resolved correctly before the cmdlet
  is called.

- **"Succeeded" health check items incorrectly shown in HealthCheck Errors panel** — the
  status filter used `StartsWith("Success")` which does not catch `"Succeeded"` (they
  diverge at position 5: `Succe**ss**` vs `Succe**ed**`). Updated all health check status
  filters in `GetUpdateHealthCheckAsync`, `GetSolutionUpdatesAsync`, and
  `GetSolutionUpdatesFromArmAsync` to use `StartsWith("SUCC")`, which correctly catches
  SUCCESS, Succeeded, Successful and any other success-variant status strings.

- **Prepare button description was incorrect** — the button tooltip and confirmation modal
  stated "without running health checks or installing" but `Start-SolutionUpdate -PrepareOnly`
  does run health checks. Corrected to "download, stage, and run health checks without
  installing".

### New Features

- **Cluster Resources panel on Cluster Roles page** — clicking a role row in the Cluster
  Roles table now expands an inline **Resources** sub-panel showing every cluster resource
  owned by that role:

  - Columns: State (colour dot), Name, Resource Type, Owner Node.
  - **Start** action (Offline / Failed resources) and **Stop** action (Online resources,
    with confirmation modal) per resource row.
  - The sub-panel refreshes automatically after a Start or Stop action succeeds; it closes
    when you click the same role row again or select a different role.
  - The selected role row is highlighted while the panel is open.
  - All Start / Stop actions are written to the Audit Log.

- **Bulk Service Operations on the Services page** — the Agent Services page now supports
  multi-select and bulk actions:

  - A **checkbox column** with a Select All toggle in the header.
  - A floating **Bulk actions bar** appears when one or more services are selected,
    offering **Start Selected**, **Restart Selected**, and **Stop Selected** buttons.
  - A **confirmation modal** lists all selected services grouped by node (with node counts)
    before executing.
  - Bulk actions run in parallel, with per-item entries
    written to the Audit Log.
  - Selection clears automatically on page refresh and after a bulk operation completes.

- **Cluster filter on PS/WinRM/ARM Call Log** (Admin → Perf Debug) — the call log toolbar
  now includes an **All clusters** dropdown that filters the table to a single cluster's
  calls. Populates automatically from the distinct cluster names currently in the log
  buffer; refreshes with the 5-second auto-refresh cycle.

- **Cluster Resources inline expansion and resource loading** — several issues fixed after
  the initial Cluster Resources panel was introduced:

  - Inline row expansion failed to toggle correctly on second click; now fixed.
  - `GetClusterResourcesAsync` was rewritten to use the C# CIM API against
    `MSCluster_Resource` (root/MSCluster namespace) instead of chained PowerShell cmdlets.
    This resolved intermittent empty-results issues caused by PS module availability,
    pipeline ordering, and `Where-Object` filtering differences across cluster versions.
  - The `OwnerGroup` filter now matches against the resource's `OwnerGroup` CIM property
    rather than a PS-side pipeline, fixing the case where sub-resources of the wrong group
    appeared in expanded panels.
  - `MSCluster_Resource` state enum values are now mapped correctly:
    `0=Unknown, 1=Online, 2=Offline, 3=PartialOnline, 4=Failed` — earlier code used
    incorrect mappings that showed "Offline" for healthy Online resources.
  - Health check ID stripping: action plan IDs embedded in resource health-check names
    are stripped before display so names remain readable.
  - Null-guard added for resources that return no `OwnerGroup` property.

- **Start/Stop Cluster Resource reliability** — `StartClusterResourceAsync` and
  `StopClusterResourceAsync` now execute on `_pool` (WinRM RunspacePool) with an explicit
  `Import-Module FailoverClusters` at the start of the command. Previously they ran on
  `_localPool` (local RSAT), which does not have the FailoverClusters module available in
  all execution contexts, causing "command not recognized" failures.

### Bug Fixes

- **Cluster role state dot (Cluster Roles page) now reflects resource states** — the
  coloured state indicator next to each cluster role is now derived from the states of its
  member resources (worst-state wins: Failed → red, Offline → orange, all Online →
  green) rather than relying solely on the `State` property of the cluster group itself,
  which could remain "Partially Online" without changing colour.

- **Snapshot freshness shows "N/A" for disabled feature types** — collector types that are
  disabled in the cluster configuration (e.g. AKS when the cluster has no AKS deployment)
  previously wrote a stale timestamp to the freshness grid immediately and kept the cell
  orange. They now display "N/A" so operators can distinguish "not configured" from
  "collection failed".

- **Solution Update** Fixes in dispaly and execution of update runs.  

---

## v0.9.13-rc2 — 2026-04-08

### New Features

- **Fleet Status as the default home page** — The Fleet Status Board (`FleetStatus.razor`)
  now serves both `/` (root) and `/status`, making it the first page every user sees after
  sign-in. The previous home page (cluster health cards) is still available at `/clusters`.
  The top-level navigation has been updated to match: Fleet Status is the first link,
  Fleet VM Status sits below it as a sub-link, and All Clusters follows.

  Summary pills on the Fleet Status Board are now **interactive toggle-filters**. Clicking
  a pill filters the cluster table to matching clusters:
  - **VMs Running** — clusters with any running VMs
  - **VMs Off** — clusters with any VMs off
  - **Nodes Up** — clusters where all nodes are Up
  - **Health Faults** — clusters with at least one critical or warning fault
  - **Updates** — clusters with ready, in-progress, or failed solution updates
  - **Clusters** pill — clears the active filter and shows all clusters (always active when
    no other filter is selected)

  Clicking the currently active pill a second time also clears the filter. A "Clear filter"
  hint bar appears while a filter is active and the Clusters pill count updates live to show
  the filtered count.

- **User Favourites and Saved Views on Fleet Status** — users can now personalise the Fleet
  Status Board without any admin access:

  - **Pin / Unpin** — click the pin icon on any cluster row to add it to your personal
    Favourites list. Pins are stored per user in the database.
  - **Favourites pill** — when you have pinned clusters, a Favourites filter pill appears
    above the table. Clicking it filters the table to show only your pinned clusters.
  - **Personal named views** — click **+ View** in the view bar to create a personal named
    cluster filter. Views support three match types:
    - **Explicit** — a hand-picked set of clusters selected by name from a checkbox list.
    - **Wildcard** — a glob pattern (`*`, `?`) matched against cluster names (e.g. `PROD-*`).
    - **Regex** — a full regular expression matched against cluster names.
  - A live match-count preview is shown while creating a wildcard or regex view.
  - Created views appear as labelled pills in the view bar. Click a view pill to filter the
    table; click it again or click Favourites / All Clusters to clear it.
  - **Delete a personal view** — click the `x` on the pill label, or remove it from the
    create-view form.

- **Admin — Shared Views** (`/admin/views`) — administrators and operators with the
  `SharedViews / Configure` RBAC permission can create **global** and **group-scoped** named
  views that are visible to all users (or to members of a specific Entra group) as view pills
  on the Fleet Status Board:

  - Shared views support the same Explicit / Wildcard / Regex match types as personal views,
    with a live match-count preview in the admin create/edit forms.
  - **Group views** appear only for users whose Entra group OID matches the view's group
    label — useful for giving different teams their own pre-filtered fleet view.
  - A new **Views** link has been added to the Admin section in the left navigation bar.
  - **RBAC:** a new `SharedViews` resource type has been added. Create/Edit/Delete operations
    require `Configure` permission on `SharedViews`. The `Admin — Roles` page now filters the
    available op checkboxes to only the ops that are valid for each resource type, preventing
    meaningless permission combinations.

- **Fleet VM Status page** (`/status/vms`) — a new fleet-wide VM table showing every VM
  across all registered clusters from background-collector snapshots (zero live WinRM calls).
  Columns: Cluster, VM Name, State, Memory Assigned, Uptime, Node, OS. Interactive filters
  above the table let operators quickly drill to VMs by state (Running / Off / Paused) or
  search by name. A Uptime sort makes it easy to spot recently restarted VMs.

- **Snapshot Freshness moved to dedicated Reports page** (`/reports/snapshot-freshness`) —
  the collapsible freshness grid that previously lived inside Fleet Status has been promoted
  to a full-page report with:
  - A per-column tooltip showing the exact collection timestamp for each data type.
  - A per-row stale-cluster warning banner listing all clusters with at least one stale type.
  - A colour legend at the top defining green / orange / red / grey thresholds.
  - RBAC guarded by the `Reports` resource type.
  A plain link to the new report replaces the old collapsible grid in the Fleet Status status
  bar. The **Snapshot Freshness** link is also available in the left navigation under Reports.

- **Disk Replacement Wizard** (Physical Disks page, Storage section) — a **Replace** button
  now appears on each physical disk row for operators with `Storage / Configure` permission
  (HciOperate or higher). Clicking it opens a three-step guided modal:
  1. **Pre-flight checks** — confirms the disk is not a Journal or Hot Spare tier, that the
     storage pool is healthy, and lists the virtual disks that use this disk for a last check
     before proceeding.
  2. **Retire & Replace** — submits the retire command (`Set-PhysicalDisk -Usage Retired`)
     and waits for the repair storage job to start, polling every 5 seconds.
  3. **Monitor repair** — shows live repair job progress (percentage, elapsed time). Once
     100% the page prompts the operator to physically swap the disk and refresh the page.
  The wizard is controlled by the `DiskReplacementWizard` feature flag in Admin → Settings
  and is only shown when the flag is enabled. The replace button is hidden entirely when the
  feature is disabled or when the user lacks `Storage / Configure` permission.

- **Cluster Health Settings page** (`/clusters/{n}/health-settings`) — new page in the
  Cluster sub-navigation for viewing and editing Health Service settings directly on the
  cluster via `Get-StorageHealthSetting` / `Set-StorageHealthSetting`.

  - **Volume Capacity Thresholds card** — edit the Warning (default 80%) and Critical
    (default 90%) percentage-full thresholds at which the Health Service raises a fault.
  - **Available Memory Threshold card** — edit the minimum free memory percentage
    (default 10%) before a health fault is raised on a node. The PS fractional value is
    computed and shown live as you type.
  - **Auto-pool New Disks card** — shows whether new physical disks are automatically
    added to the storage pool (`System.Storage.PhysicalDisk.AutoPool.Enabled`), plus
    whether the value is an active override or the system default.
  - **Active Overrides table** — lists every setting that is explicitly overriding its
    system default, with short name, category, raw/display values, and the default
    value for comparison. Long XML blob values are truncated in the table for readability.
  - **SDDC Management restart wizard** — after any save, a modal guides the operator
    through Stop → 10 s countdown → Start of the SDDC Management cluster role so the
    change takes effect without manual PowerShell.
  - **Info banner** — a static advisory note that default values are appropriate for most
    environments and changes should be made with care.
  - **Access control:** View requires Storage `View` permission; editing cards require
    Storage `Configure` permission (HciOperate or higher). Read-only users see the Active
    Overrides table and info banner but no editing controls.

### Bug Fixes

- **Diagnostics page returned HTTP 403 for users with View-only RBAC** — the page was
  incorrectly decorated with `[Authorize(Policy = GroupPolicies.Operate)]` instead of
  `GroupPolicies.Read`. Users in the HciRead group received a 403 when navigating to
  Diagnostics even though the page is a read-only view. Fixed to `GroupPolicies.Read`.

- **Solution Updates — health check panel improvements:**
  - **Succeeded results now filtered out by default** — the Health Check results panel
    previously showed all check results; passing checks cluttered the view when dozens of
    components return Succeeded. A "Show passed" toggle now hides Succeeded entries by
    default so only Warnings and Failures are shown.
  - **PrepareOnly button** — a dedicated **PrepareOnly** action button has been added to
    the Solution Updates page to run `Start-SolutionUpdate -PrepareOnly` separately from a
    full update, giving operators a way to stage update packages without committing to the
    full installation.
  - **Run Invoke-Precheck button moved to health strip** — the button to trigger
    `Invoke-SolutionUpdatePrecheck -SystemHealth` is now visible directly in the health
    status bar at the top of the page rather than being buried inside the health check
    panel. This makes it accessible whether the panel is expanded or collapsed.
  - **Success count in health check panel** — the panel header now shows the count of
    passed checks alongside the warning/failure counts.
  - **Null-safe health check fetch** — `GetUpdateHealthCheckAsync` is now null-safe and
    returns an empty list when the cluster response is missing a `properties` object,
    preventing a null-reference exception on clusters that haven't run a health check yet.
  - **`StartSolutionUpdateAsync` bails out cleanly on lookup failure** — if the service
    cannot resolve the action plan instance ID for the target update, the method now returns
    a clear error message and stops rather than throwing a null-reference exception that
    surfaced as an unhandled error on the page.

- **SMTP alerts failing with port 25 unauthenticated relay** — `SmtpEmailSender` previously
  used `StartTls` (mandatory STARTTLS) for all ports other than 465. Internal SMTP relays
  that listen on port 25 and do not support STARTTLS caused `535 5.7.3 Authentication
  unsuccessful` or "does not support non-STARTTLS" errors, preventing any alert email from
  being delivered.
  Fixed by auto-selecting the TLS mode from the configured port:
  - Port 465 (or `SmtpUseSsl = true`) → `SslOnConnect`
  - Port 25 → `None` (plain unauthenticated relay, no TLS)
  - Port 587 / any other → `StartTls` (mandatory STARTTLS)
  No configuration change is required for existing deployments using port 587 or 465.
  Deployments using an internal port 25 relay only need to set `Smtp:Port = 25` in
  Admin → Settings.

- **Extension upgrade blocked by undiscoverable manual-mode extensions** — Arc machine
  extension upgrades and cluster extension upgrades now show a **warning modal** before
  proceeding whenever one or more selected extensions have `Upgrade Mode: Manual`. The
  modal lists the affected extension names and explains that manual-mode extensions were
  intentionally set to not auto-upgrade. The operator must explicitly confirm to continue.
  Applies to both the **Arc Extensions** page and the **Cluster Extensions** page.

- **Settings dropdowns always show the effective value** — Admin → Settings combo boxes
  previously showed a blank option when a setting had never been saved, rather than
  displaying the documented default. All dropdowns now default to the system default value
  and the blank/placeholder option has been removed, making it clear what value is active.

- **VM Uptime missing from Fleet VM Status** — the background poller was running with
  `lightMode = true`, which skipped the `Uptime` property when building VM snapshots.
  Disabled `lightMode` so `Uptime` is now included in every snapshot and available on the
  Fleet VM Status page.

- **Fleet Status pin buttons showing star/asterisk characters** — the Favourite pin icon
  previously rendered as a raw Unicode star character (`★`) in some browsers due to missing
  font fallback. The icon text is now set as a literal HTML entity so it renders correctly
  across all browsers.

- **Column alignment** — several numeric and metric columns that were right-aligned or
  inconsistently placed have been changed to left-align for readability:
  - VM Performance (CPU%, memory, VHD IOPS/latency/throughput, network)
  - Virtual Disks (Size, Allocated) and Storage Pools (Total Size, Allocated Size)
  - Fleet VM Status (Memory, CPU%) and Arc Machines (Cores, Memory)

### Security

- **API 500 responses no longer leak internal error details** — `ClustersApiController`
  Add, Update, and Delete actions previously returned `ex.Message` verbatim in the HTTP 500
  body. This could expose connection strings, server file paths, or internal class names to
  API callers. The HTTP response now returns a generic message
  (`"An internal error occurred. Check the audit log for details."`);
  `ex.Message` is retained in the audit log `Detail` field (server-side only).

- **`SkipCertValidation` defaults to `false`** — `ClusterConfig` and the `ClusterApiRequest`
  DTO previously defaulted `SkipCertValidation = true`, meaning all clusters added via the
  API without explicitly specifying the field skipped WinRM TLS certificate validation.
  Changed default to `false` (secure-by-default). Existing clusters stored in the database
  are not affected. Operators who use self-signed or internal-CA WinRM certificates must
  explicitly set `skipCertValidation: true` when adding clusters via the API.

---

## v0.9.12-rc1 — 2026-04-05

### Performance — WinRM to Local PS + CIM Migration

The primary focus of this release is eliminating `wsmprovhost.exe` (WinRM PS shell)
spin-up overhead from the hot-path read operations. Most cluster queries now run on the
**app server itself** using local RSAT modules with `-Cluster`/`-CimSession` parameters,
or via the C# CIM API (`Microsoft.Management.Infrastructure`) without PowerShell at all.

**Benchmark (cluster-level reads):** ~650ms average → ~36ms average (18x improvement).

#### Architecture change

| Transport | Before | After |
|---|---|---|
| `_pool` (WinRM RunspacePool) | All cluster reads + mutations | Mutations + ATC/ECE/Arc/Storage cmdlets with no CIM equivalent |
| `_cimSession` (CIM to cluster) | Storage only (3 methods) | Cluster nodes, roles, networks, CSVs + storage |
| `_localPool` (local RSAT PS) | Not present | VM reads, VM commands, FailoverClusters mutations, Hyper-V config ops, FetchNodeStats |
| `_nodeCimSessionCache` (CIM per node) | Not present | VM reads step 2, FetchNodeStats, network adapters fallback |
| `GetOrCreateCachedRunspace` (WinRM per-node) | Get-VM, FetchNodeStats, adapters | Get-NetAdapter only (NetAdapter module not on app server) |

`wsmprovhost.exe` is now only spawned for operations that have no local alternative:
WinRM RunspacePool for cluster mutations, direct runspaces for Get-NetAdapter.
All other paths use WmiPrvSE.exe (CIM provider) or no remote process at all.

#### Prerequisites

`Setup-Prerequisites.ps1` now installs two additional RSAT features required by
the app server:

- `RSAT-Hyper-V-Tools` — provides Hyper-V PS module (Get-VM, Start-VM, etc.) locally
- `RSAT-Clustering-PowerShell` — provides FailoverClusters PS module locally

Run `scripts\Setup-Prerequisites.ps1` on the app server before or after upgrading.

#### Methods migrated to `_localPool` (local RSAT + CimSession per node)

| Method | Module | Transport |
|---|---|---|
| `GetVirtualMachinesAsync` step 2 | Hyper-V | `_localPool` + `GetOrCreateNodeCimSession` |
| `ExecuteVMCommandAsync` | Hyper-V | `_localPool` + `GetOrCreateNodeCimSession` |
| `RemoveSnapshotAsync` | Hyper-V | `_localPool` + `GetOrCreateNodeCimSession` |
| `ResizeVHDAsync` | Hyper-V | `_localPool` + `GetOrCreateNodeCimSession` |
| `SetVMProcessorAsync` | Hyper-V | `_localPool` + `GetOrCreateNodeCimSession` |
| `SetVMMemoryAsync` | Hyper-V | `_localPool` + `GetOrCreateNodeCimSession` |
| `MoveClusterSharedVolumeAsync` | FailoverClusters | `_localPool -Cluster` |
| `StartClusterRoleAsync` | FailoverClusters | `_localPool -Cluster` |
| `StopClusterRoleAsync` | FailoverClusters | `_localPool -Cluster` |
| `PauseClusterNodeAsync` | FailoverClusters | `_localPool -Cluster` |
| `ResumeClusterNodeAsync` | FailoverClusters | `_localPool -Cluster` |
| `FailbackClusterNodeAsync` | FailoverClusters | `_localPool -Cluster` |
| `MoveVMAsync` | FailoverClusters | `_localPool -Cluster` |
| `FailoverClusterRoleAsync` | FailoverClusters | `_localPool -Cluster` |
| `FetchNodeStats` | Hyper-V + Win32 | `_localPool` + `GetOrCreateNodeCimSession` |
| `GetVirtualDisksAsync` (CSV sub-query) | FailoverClusters | `_localPool -Cluster` |

#### Methods migrated to `_cimSession` (C# CIM API, no PS)

| Method | CIM class | Namespace |
|---|---|---|
| `GetNodeNamesInternal` | `MSCluster_Node` | `root/MSCluster` |
| `GetVirtualMachinesAsync` step 1 | `MSCluster_Node` | `root/MSCluster` |
| `GetClusterNodesAsync` | `MSCluster_Node` | `root/MSCluster` |
| `GetClusterRolesAsync` | `MSCluster_ResourceGroup` | `root/MSCluster` |
| `GetClusterNetworksAsync` | `MSCluster_Network` | `root/MSCluster` |
| `GetClusterSharedVolumesAsync` | `MSCluster_ClusterSharedVolume` + owner association | `root/MSCluster` |
| `FetchNodeStats` (OS + CPU + registry) | `Win32_OperatingSystem`, `Win32_Processor`, `StdRegProv` | `root/cimv2` / `root/default` |

Note: `FetchNodeStats` uses the C# CIM API directly (not CimCmdlets PS module) because
PS7 Core cannot load `CimCmdlets` in some IIS/gMSA hosting configurations.

#### `GetClusterInfoAsync` reverted to WinRM pool

`GetClusterInfoAsync` was initially migrated to `_localPool` with `-Cluster` parameters.
In production this caused 14–34 second durations (previously 160–400ms) because
FailoverClusters `-Cluster <name>` from a non-member machine uses DCOM/RPC rather than
WsMan — each of the 5–6 cmdlets in the script opens its own DCOM connection (no session
reuse), resulting in much higher overhead than a single persistent WinRM session.
**Reverted** to run on `_pool` (WinRM runspace on a cluster node) where all FailoverClusters
cmdlets execute locally without `-Cluster`.

#### `GetNetworkAdaptersAsync` reverted to per-node WinRM runspace

`GetNetworkAdaptersAsync` was migrated to local PS + CimSession but then reverted because
`Get-NetAdapter` / `Get-NetIPInterface` / `Get-NetAdapterRdma` come from the `NetAdapter`
module which is not installed on the app server. The method was returned to
`GetOrCreateCachedRunspace` (per-node WinRM) where the module is available on each cluster
node. Driver properties (`DriverProvider`, `DriverVersionString`) added in the same step.

#### Diagnostics improvements

- **Perf Debug — Type column** — the `/admin/debug-perf` call log now shows a colour-coded
  transport badge per entry: **WinRM** (slate), **CIM** (teal), **ARM** (purple),
  **Local** (green). Local = `_localPool` invocations running on the app server.
- **`[Local]` tag auto-appended** — `InvokeLogged()` detects when `ps.RunspacePool` is
  the local pool and appends `[Local]` to the label automatically, making transport type
  visible in the call log without manual tagging.

#### Benchmark scripts

Four new scripts in `scripts/perf/` for measuring and comparing transport approaches:
- `Benchmark-A-RunspacePool.ps1` — times the WinRM RunspacePool pattern
- `Benchmark-B-ComputerName.ps1` — times local modules with `-ComputerName`/`-Cluster`
- `Benchmark-C-CimSession.ps1` — times CimSession (fresh per call vs. reused)
- `Compare-Approaches.ps1` — runs A, B, C and outputs a side-by-side timing table

### Features

- **Alerting — Alert acknowledgement with RBAC** — fired alert history entries can now be
  acknowledged by users who hold the `Op.Acknowledge` permission on the `AlertRules`
  resource type (the default `Cluster Admin` role now includes this operation).
  Acknowledging an alert marks it with the user's UPN, a timestamp, and an optional note.
  Cooldown behaviour respects acknowledgements: acking an alert resets the cooldown window
  so the next firing of the same rule sends a fresh notification rather than being
  suppressed by the previous event.  
  UI: acknowledged entries show a green check mark (hover for the note). An **Ack** button
  is shown in the history table when the signed-in user has `Op.Acknowledge`. Clicking
  opens a confirmation modal with an optional free-text note field.

- **Alerting — VMUnexpectedStop VM name glob filter** — `VMUnexpectedStop` alert rules now
  support an optional `VmNamePattern` field (shown in the rule editor when rule type is
  `VM Stop`). Supports `*` (any sequence of characters) and `?` (single character), case-
  insensitive. Leave blank to alert on all VMs. Examples: `PROD-*` alerts only on
  production VMs; `*-SQL-*` alerts on any VM with `SQL` in the name. The pattern is shown
  as a hint in the Cluster column of the rules table when set.

- **Solution Updates — Get WinRM Data button** — when ARM is configured, a secondary
  "Get WinRM Data" button appears in the toolbar allowing operators to fetch live
  solution update data directly from the cluster via WinRM, complementing the faster
  ARM-sourced primary refresh.

- **Solution Updates — ARM catalog source** — when ARM is configured and cluster
  coordinates are available, `SolutionUpdates` page loads the update catalog from
  ARM (~1 second) instead of WinRM (~20–30 seconds). Falls back to WinRM snapshot
  when ARM is not configured. Update runs remain WinRM.

- **Network Adapters — Driver info** — `DriverProvider` and `DriverVersionString`
  columns added to the Network Adapters page.

### Bug Fixes

- **Drift detection double WinRM hit** — the background poller was calling
  `Get-SolutionUpdate` (9-minute cmdlet) redundantly before `RunDriftDetectionAsync`.
  The poller now reads the cached installed release ID from DB via
  `GetCachedInstalledReleaseIdAsync`, skipping the WinRM call.

---

## v0.9.11-beta — 2026-04-04

### Testing

- **GitHub Actions integration test workflow** (`integration-tests.yml`) — two-job CI pipeline:
  - **Job 1 (ubuntu/xUnit):** builds on `ubuntu-latest`, runs all 294 xUnit unit and
    integration tests via `dotnet test`. Triggers on push to `feature-*` and
    `main` branches and on pull requests.
  - **Job 2 (self-hosted k6 + Playwright):** runs on `GITHUBRUN` (Windows Server
    self-hosted runner); executes k6 load test against the Entra ID site
    (`https://azlmgmt.yourdomain.com`), then Playwright E2E tests against the
    WinAuth site (`https://azlmgmt-win.yourdomain.com`). k6 uses
    `insecureSkipTLSVerify: true` because the server uses a private CA certificate.

- **Playwright E2E test suite** (`Tests/E2ETests/PageLoadTests.cs`) — 13 page-load
  smoke tests covering all major cluster pages. Each test navigates to the page and
  asserts that either a content element or an error box loads within 45 seconds.
  Tests pass on the WinAuth site using Windows Authentication (no Entra sign-in required
  for the CI runner host account).

- **ClusterInfo page test selector fix** — `ClusterInfo_LoadsSummary` was waiting for
  `.overview-card` but `ClusterInfoPage.razor` uses `info-card`/`info-grid` CSS classes
  (not `overview-card`). Changed selector to `.info-card, .alert-error` to correctly
  detect page load. All 13 tests now pass.

### Scripts

- **`scripts/New-AppServiceAccount.ps1`** — new script for automated AD service account
  setup supporting both gMSA and standard domain accounts.
  - `gMSA` mode (default): checks/creates the KDS root key, creates the gMSA with
    30-day password rotation, sets retrieval principals (by computer object or AD group),
    installs the gMSA on each app server via `Invoke-Command`, and adds the gMSA to
    local Administrators on each cluster node.
  - `Standard` mode: prompts for password with confirmation (or accepts `-Password`
    for automation), creates the AD user, optionally sets `PasswordNeverExpires`, and
    adds the account to local Administrators on each cluster node.
  - PS5.1 compatible: no `??` operator, no inline if-expressions, ASCII-only strings
    in values.
  - On completion, prints next-step instructions pointing to `Setup-Prerequisites.ps1`
    and `Setup-IIS.ps1`.

### Documentation

- **`docs/IIS-Deployment.md`** — gMSA Requirements and Standard Service Account sections
  now each include an "Automated (recommended)" block showing how to run
  `New-AppServiceAccount.ps1` before the existing manual steps. Manual steps retained
  for environments where the script cannot be run.

- **`docs/ARCHITECTURE.md`** — Service Dependency Map (section 5) updated to include
  the background services introduced in v0.9.10-beta: `ClusterPollerService`,
  `SnapshotService`, `AlertEngine`, `MaintenanceWindowService`, `CollectorTypeScheduler`,
  `CompositeAlertSender`, `SmtpEmailSender`, `TeamsWebhookSender`, and
  `DatabaseHealthCheck`.

---

## v0.9.10-beta — 2026-04-03

### Performance

- **Persistent poller WinRM connections** — `ClusterPollerService` previously created and
  disposed a `WebHyperVService` on every poll visit. This threw away the entire per-node
  runspace cache on every cycle, causing a cold WinRM connect penalty (~5s) before every
  VM and Node collection. The poller now holds a `_pollerConnections` dictionary of
  long-lived `WebHyperVService` instances per cluster. Connections are reused across poll
  visits and only recreated on connection failure (circuit-breaker pattern). Observed
  improvement: AZL-NUC-CL02 VM poll time dropped from 5,163ms to 24ms.

- **Extended per-node runspace TTL for poller (60s → 300s)** — The per-node runspace cache
  TTL was 60s, shorter than the VM poll interval (120s) and Node poll interval (240s).
  For single-node clusters this meant the cache always expired between polls — cold connect
  on every cycle even with persistent connections. The poller now passes `nodeCacheTtl: 300s`
  when constructing its `WebHyperVService` instances. Page-tier connections retain the 60s
  default (unchanged behaviour for interactive users). `wsMan.IdleTimeout` now scales
  automatically to `2 * nodeCacheTtl` to maintain adequate server-side margin.

### Bug Fixes

- **Nodes page missing CPU/memory/uptime data** — The poller was collecting
  `GetClusterNodesAsync(enrichStats: false)`, storing a lightweight snapshot (State/DrainStatus
  only) to the DB. `SnapshotService` served this snapshot to the Nodes page when fresh
  (within the stale threshold of ~12 minutes), resulting in empty CPU%, Memory, and Uptime
  columns with no error or visual indicator. Data only appeared after the snapshot went stale
  or the user forced a Refresh. Fixed by removing `enrichStats: false` from the poller —
  with warm 300s persistent runspaces the additional CIM query per node costs ~100ms against
  a 600s poll interval.

- **Exponential backoff for consistently-failing collection types** — When a collection type
  fails (e.g. `Get-SolutionUpdate` on a standalone Hyper-V host, `RunDriftDetection` on a
  non-HCI cluster), `MarkPolled` was never called, leaving `lastPoll = DateTime.MinValue`
  forever. Every subsequent visit saw the type as always-due, triggering repeated 9-minute
  hangs and 90-130 second DriftDetection timeouts on every poll cycle — monopolizing the
  poll slot and blocking all other collection types.
  Fixed by adding `MarkFailed(clusterName, dataType)` to `CollectorTypeScheduler` with
  exponential backoff: 1 → 2 → 4 → 8 → 16 → 32 → 60 min (capped). `GetDueTypes` skips
  types on backoff. `MarkPolled` on the next success clears the backoff. The per-type
  catch block in `ClusterPollerService` now calls `MarkFailed` on every collection exception.

### Diagnostics

- **Per-node Get-VM duration now visible in PsCallLog** — `GetVirtualMachinesAsync` step 2
  (the per-node `Get-VM` script via direct runspace) was using raw `ps.Invoke()`, making it
  invisible in Admin > Perf Debug. Only the step 1 `Get-ClusterNode` duration was shown.
  Switched to `InvokeLogged(ps)` so both steps appear separately, making it possible to
  distinguish slow cluster WMI response from slow Hyper-V enumeration on a specific node.

### Deployment

- **New `Install.ps1` script** — The release ZIP now includes `scripts\Install.ps1`, the
  missing step between extracting the ZIP and running `Setup-IIS.ps1`. It handles both
  first-time installs and upgrades:
  - **First time:** copies app binaries to the install path, places `appsettings.Production.json`
    and `clusters.json` from the bundled templates so both files are ready to edit.
  - **Upgrade** (`-Upgrade`): stops the configured app pool(s) by name, replaces binaries
    via robocopy `/MIR /XF`, preserves `appsettings.Production.json` and `clusters.json`,
    then restarts the pools. Config is never overwritten.
  - `Setup-Prerequisites.ps1` is also now included in the release ZIP.
- **Documentation updated** — `QUICK-START.md`, `IIS-Deployment.md`, and `USER-GUIDE.md`
  updated to reflect the new install flow. The upgrade section in the quick start now shows
  `Install.ps1 -Upgrade` as the primary path. `Deploy-ToIIS.ps1` is retained for developers
  building from source.

---

## v0.9.9-beta — 2026-04-02

### Bug Fixes

- **Blazor circuits still disconnecting after v0.9.8 deploy** — Evidence from Windows
  Application Event Log (Event 1000, Category `Microsoft.EntityFrameworkCore.Database.Command`,
  EventId 20102) showed `Failed executing DbCommand (34,002ms)` with SQL targeting
  `LatestSnapshots` and `SnapshotHistories`. The 34s duration is the exact signature of
  EF Core's retry policy (2 retries x 15s command timeout + delays). The source was
  `ClusterPollerService` — its three DB helper methods (`UpsertSnapshotAsync`,
  `GetOrCreateHealthAsync`, `SaveHealthAsync`) each create their own `DbContext` but never
  set a command timeout override, so they inherited the global 15s. With `MaxConcurrentPolls=3`
  and 4 parallel data types per cluster, up to 12 background write operations could each
  hold a DB connection for 34s simultaneously. This approached the `MaxPoolSize=20` ceiling,
  starving Blazor circuit threads waiting for a connection and causing them to exceed
  `ClientTimeoutInterval=60s`. The Event 1000 flood was not process crashes — the EventLog
  logging provider (`"EventLog": { "LogLevel": { "Default": "Error" } }`) routes all EF Core
  Error-level log entries to Windows Application Event Log, which looks alarming but is
  just structured logging.
  Fixed by adding `db.Database.SetCommandTimeout(5)` (guarded with `db.Database.IsRelational()`
  to protect in-memory tests) to all three poller DB helpers. A simple upsert to these tables
  should complete in <100ms; 5s is generous. If a genuine timeout occurs, the background
  service skips that write and retries on the next poll cycle — poller data is ephemeral.

- **`rapidFailProtection = True` on IIS app pools** — IIS rapid-fail protection monitors
  for crash bursts and disables the app pool if too many failures occur within a time window.
  The EF Core Error log flood described above could conceivably trigger this on some servers.
  More broadly, Blazor Server holds long-lived SignalR connections that IIS should not
  mistake for a crash loop under any load spike.
  Fixed by adding `Set-ItemProperty ... failure.rapidFailProtection $false` to step 7b of
  `Deploy-ToIIS.ps1` and to `Setup-IIS.ps1` / `Setup-IIS-WinAuth.ps1`, applied to all
  pools alongside `idleTimeout` and `periodicRestart.time`.

- **`wsmprovhost.exe` accumulation from background poller** — `ClusterPollerService`
  calls `GetClusterNodesAsync()` on every poll visit. That method fans out to each
  cluster node via `GetOrCreateCachedRunspace` (one WinRM connection per node, one
  `wsmprovhost.exe` per node on the cluster) to collect CPU/memory/uptime via
  `FetchNodeStats`. The poller snapshot only needs `State` and `DrainStatus` from
  `Get-ClusterNode` (which runs via the RunspacePool cluster connection, no direct
  per-node connection); the per-node enrichment is unused in the snapshot and was
  creating N extra `wsmprovhost.exe` connections per Nodes poll cycle.
  Fixed by adding an `enrichStats` parameter (default `true`) to `GetClusterNodesAsync`.
  The poller now calls `GetClusterNodesAsync(enrichStats: false)`. Pages that call the
  method directly get full enrichment as before.

---

## v0.9.8-beta — 2026-04-02

### New

- **`Clusters:RunspacePoolMax` configurable** — The PowerShell RunspacePool size per cluster
  is now configurable via `appsettings.json` (`"Clusters": { "RunspacePoolMax": 5 }`).
  Previously hardcoded to 5. The default remains 5, which is the right value for most
  deployments (up to ~8-10 simultaneous active users per cluster without queuing, with
  zero cross-cluster contention). The `appsettings.json` comment block explains the
  capacity model in detail and gives guidance for when to increase or decrease the value.
  A full performance tuning section has been added to `docs/IIS-Deployment.md`.

### Bug Fixes

- **Blazor disconnects still occurring after v0.9.7 deploy** — `Deploy-ToIIS.ps1` copies
  application files and recycles the app pool but never applied the critical IIS settings
  (`idleTimeout=0`, `periodicRestart.time=0`). Those settings were only written by
  `Setup-IIS.ps1`, which is only run once during initial server setup. Any deployment using
  `Deploy-ToIIS.ps1` on a server where those settings had drifted or were never applied
  left the pool in its default state (29-hour periodic recycle, 20-minute idle timeout).
  Fixed by adding step 7b to `Deploy-ToIIS.ps1`: on every deploy it now applies both
  `processModel.idleTimeout=00:00:00` and `recycling.periodicRestart.time=00:00:00` to
  all app pools on the target server via PS remoting. This is idempotent — re-applying an
  already-correct setting produces no change.

- **"An error occurred using the connection to database"** — PostgreSQL's default
  `max_connections=100` was being exhausted by Npgsql's default `Maximum Pool Size=100`.
  When the app fully utilised its connection pool it consumed all available PostgreSQL
  connections, causing every subsequent request to fail with
  `FATAL: sorry, too many clients already`. Fixed by capping `Maximum Pool Size=20` via
  `NpgsqlConnectionStringBuilder` in `Program.cs` (applied regardless of what is in the
  server's `appsettings.Production.json`) and adding the parameter to the connection string
  template. Additionally, `Keepalive=60` and `ConnectionIdleLifetime=300` are also
  injected programmatically to prevent idle connections being silently dropped by
  firewalls or NAT devices.

- **`wsmprovhost.exe` accumulation on cluster nodes** — WinRM provider-host processes were
  accumulating on cluster nodes (observed growing from 117 to 291 rapidly) due to several
  compounding issues in `WebHyperVService`:
  (1) `GetWindowsServicesAsync`, `ServiceOperationAsync`, `GetBitLockerStatusAsync`, and
  the `GetSecurityFeaturesAsync` fallback strategy each created fresh per-node WinRM
  connections on every call instead of reusing the per-node runspace cache, producing N new
  `wsmprovhost.exe` processes per page load.
  (2) `GetBitLockerStatusAsync` called `rs.Close(); rs.Dispose()` in a `finally` block
  after pulling the runspace from the cache, destroying the cached entry and forcing a new
  WinRM connection on every subsequent call.
  (3) The per-node runspace cache had no proactive cleanup — entries could only be evicted
  on the next cache access, meaning connections accumulated indefinitely when the site was
  idle or a user navigated away.
  (4) `IdleTimeout` was 300 s (5 min), so any abandoned connection from an unclean shutdown
  lingered on the cluster node for 5 minutes before the server cleaned it up.
  Fixed by: converting all four methods to use `GetOrCreateCachedRunspace` (reusing
  existing connections); removing the destructive `finally` in `GetBitLockerStatusAsync`;
  adding a 30-second `PeriodicTimer` background task in `ConnectAsync` that proactively
  sweeps and closes stale (TTL-expired or broken-state) cache entries; reducing
  `RunspaceCacheTtl` from 90 s to 60 s and `IdleTimeout` from 300,000 ms to 120,000 ms
  (maintaining the required 2x safety margin between the two).

- **Blazor circuits disconnecting on PostgreSQL DB blips** — EF Core's retry policy
  (3 retries x 30 s command timeout + 5 s delays) could block a Blazor hub thread for up
  to ~100 s on any transient DB hiccup. SignalR's `ClientTimeoutInterval` is 60 s, so any
  block longer than that killed the circuit with "Server returned an error on close",
  causing the browser log cascade of SignalR errors followed by long-polling timeouts.
  `RbacService.LoadSnapshot()` compounded this as a synchronous `_loadLock.Wait()` + EF
  query on the circuit thread with the same full-timeout exposure.
  Fixed by: reducing EF Core command timeout from 30 s to 15 s and max retries from 3 to 2
  (worst case ~35 s — well within the 60 s SignalR window); adding a 5 s command timeout
  override in `RbacService.LoadSnapshot()` so a DB blip falls through to pass-through mode
  in under 13 s rather than up to 90 s; fixing the `MaxPoolSize` enrichment guard to also
  handle Npgsql's `0` default (unlimited), and setting `MinPoolSize=1` to keep a warm
  connection after idle periods. Connection string template updated accordingly.

---

## v0.9.7-beta — 2026-04-01

### Bug Fixes

- **Blazor disconnects every ~29 hours** — IIS app pool `recycling.periodicRestart.time`
  defaults to 1740 minutes (29 hours). When the worker process recycles on this timer, all
  active SignalR circuit IDs are discarded. Clients attempting to reconnect receive "connection
  could not be found on the server" and fall back to slow long polling. Fixed by setting
  `periodicRestart.time = 00:00:00` (disabled) in `Setup-IIS.ps1` and `Setup-IIS-WinAuth.ps1`.
  To apply immediately on an existing server without redeploying, run on the IIS machine:
  `Set-ItemProperty "IIS:\AppPools\<PoolName>" -Name recycling.periodicRestart.time -Value "00:00:00"`

- **Blazor client-side serverTimeout too short** — Blazor JS auto-derives `serverTimeout`
  as `2 x KeepAliveInterval`. With `KeepAliveInterval = 10 s` (set in v0.9.6), the client
  used a 20 s timeout. A single delayed WebSocket ping from a corporate proxy or busy server
  then caused an immediate disconnect. Fixed by configuring `Blazor.start()` in `App.razor`
  with an explicit `serverTimeout: 120000` (120 s) and `keepAliveInterval: 10000` (10 s),
  using `autostart="false"` on `blazor.web.js` so the circuit config applies before the
  hub connection is established.

### Documentation and Scripts

- **`Setup-IIS-WinAuth.ps1`** — added `processModel.idleTimeout = 00:00:00` and
  `recycling.periodicRestart.time = 00:00:00`, which were present in `Setup-IIS.ps1`
  but missing from the WinAuth variant.

- **`IIS-Deployment.md`** — added a "Critical SignalR settings" subsection under App Pool
  Configuration with a table explaining `idleTimeout` and `periodicRestart.time`, the
  symptoms if they are not set, and manual fix commands. Added three new troubleshooting
  table rows (periodic restart disconnect, idle disconnect, proxy disconnect) and a new
  "Proxy Bypass" section explaining the multi-level wildcard bypass problem with diagnosis
  commands and PAC/GPO fix examples.

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
  checks and sets this environment variable on `AZLManagementPool` via PS remoting on every deploy.
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
