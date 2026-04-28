# Release Notes — Azure Local Cluster Tool (Web)

---

## v0.10.7

### Bug fix — continued wsmprovhost.exe accumulation (poller TTL race)

v0.10.6 fixed the `FetchNodeStats` leak but wsmprovhost processes continued to accumulate.
The root cause was a second related issue: the poller's per-node CimSession cache TTL was
set to 300 seconds, matching the configured Fast-tier poll interval exactly. Because the
cache check is `age < TTL`, the session is always stale at the moment of the next poll and
a new `wsmprovhost.exe` is spawned on each node every 300 seconds. With 14 clusters and
2 nodes each, that is 28 new processes every cycle. Slow-responding nodes cannot close
them cleanly, so they accumulate over hours.

The TTL is now 10 minutes, ensuring sessions are reused across multiple poll cycles
regardless of the configured base interval.

After deploying, kill any remaining orphans with
`Get-Process wsmprovhost | Stop-Process -Force` on each node, or wait for the 2-hour
WinRM idle timeout to clear them naturally.

---

## v0.10.6

### Bug fix — wsmprovhost.exe accumulation on cluster nodes

The Nodes page was leaving orphaned `wsmprovhost.exe` processes on cluster nodes. Each
time the page loaded with a stale snapshot, a new WinRM CIM session was opened per node
and not reliably closed — if the node was slow to respond, the server-side process
lingered for up to 2 hours. Over time this accumulated hundreds of orphaned processes.

The fix reuses the existing per-node CIM session cache (5-minute TTL) for these reads,
so a single `wsmprovhost` stays warm across page visits rather than a new one being
created on each load.

After deploying, recycle the IIS app pool once to force-close any existing orphaned
connections. Remaining orphans on the nodes will self-clean within 2 hours, or can be
cleared immediately with `Get-Process wsmprovhost | Stop-Process -Force` on each node.

---

## v0.10.5

### Fleet Schedules page

A new **Fleet Schedules** page (`/schedules`, linked from the nav alongside Fleet Update
Status) gives a single place to view and manage scheduled updates across every registered
cluster. Pending schedules are grouped by batch with expand/collapse, and a Schedule
History table shows the last 50 completed, cancelled, or failed rows.

A **New Schedule** modal lets operators schedule the same update across multiple clusters
in one step — cluster checkboxes show only those where the version is confirmed in the
snapshot, with a date/time picker, optional notes, maintenance window toggle, prepare-only
option, and a configurable stagger (minutes between each cluster's start time).

Both tables include a **Live State** column that shows the current install state from
each cluster's snapshot (Preparing, Downloading, Installing, Ready, Failed, etc.) so you
can see at a glance whether a triggered schedule is actively running. Batch group headers
aggregate live states across all member clusters. The whole page is a pure database read
— no WinRM connection required.

### Solution Updates — rendering fixes

Three rendering bugs in the Solution Updates schedule UI were fixed:

- The version number was missing from update entries in the schedule dropdown (`v` was
  displayed with no number following it).
- The multi-cluster schedule modal was showing clusters where the selected update version
  is not available in the snapshot — those clusters are now hidden from the list.
- The expand/collapse arrows on batch schedule rows were appearing as literal `&#x25BC;`
  text instead of the actual ▼ / ▶ characters.

---

## v0.10.4

### Platform Topology

A new **Platform Topology** page (Azure Arc > Platform Topology) shows every ARM resource
associated with the cluster's Custom Location in one place. Resources are grouped by role —
AKS Cluster, Gallery Image, Logical Network, Storage Path, Virtual Hard Disk, and more —
with traffic-light status indicators, direct Azure Portal links, and internal navigation
links to the relevant app page. Each role group expands a supplementary Details column
that shows role-specific properties (VHD size, NIC IP address, Gallery Image OS + version,
AKS Kubernetes version and node count). Results are cached after the first load for fast
subsequent views.

### Cluster Info — VM Load Balancer

The Cluster Info page now includes a **VM Load Balancer** card showing the cluster's
automatic balance settings (mode: Disabled / If Highly Unbalanced / Always, and
aggressiveness: Low / Medium / High). Resolves [#10](https://github.com/jasontayler/AzureLocal-ClusterTool-Web/issues/10).

### Cluster Storage improvements

- **Storage Path column** — the Cluster Shared Volumes table now shows which ARM storage
  container backs each volume, with a direct Azure Portal link.
- **Hide Primordial** — the Storage Pools section now hides system-managed primordial pools
  by default. A toolbar checkbox reveals them when needed.

---

## v0.10.3

### Solution Updates — scheduling and ARM integration

- **Schedule updates** — any update can be scheduled for a future date and time via a
  new Schedule button and date/time picker. Scheduled updates appear in a new Schedules tab;
  a background service fires the update automatically at the configured time.
- **Scheduling permissions** — Schedule and Cancel Schedule are independently controlled
  via custom RBAC, separate from the Start permission.
- **ARM-sourced run data** — update run history and the full nested action plan step tree
  are now loaded directly from ARM, removing the need for an active WinRM connection to
  view run history. A manual "Get WinRM Data" button still fetches supplementary local data
  on demand.
- **Error Details panel** — failed runs show an inline Error Details button that displays
  the ARM error payload without navigating away.
- **Live status improvements** — active run detection accounts for ARM catalog lag
  (the brief window where ARM still shows the previous state while the run is in progress).
  Action buttons are suppressed while a run is genuinely in flight to prevent duplicate
  triggers.

### Azure Arc — faster page loads

Arc-related pages (Arc Machines, Arc Extensions, Cluster Extensions, Arc Resource
Bridge) no longer make a live WinRM call on every page open. ARM identity coordinates
(subscription, resource group, tenant, region) are stored in the database after the first
successful poll and reused on all subsequent loads, reducing page open time noticeably
when WinRM is slow or temporarily unavailable.

### VM actions — live state tracking

After starting, stopping, restarting, or suspending a VM, the row updates in real time as
the VM moves through intermediate states (e.g. Starting → Running, Stopping → Off) without
refreshing the full VM list.

### Other improvements

- **Cluster Info** — a new VM Load Balancer card shows automatic balancing mode
  (Disabled / On Node Join / Always) and aggressiveness level.
- **Arc Machines** — the expanded machine row now shows the Arc Gateway name and endpoint,
  alongside a redesigned hardware and system detail layout.
- **Admin** — Disabled Features and ARM Identity Coordinates sections in the cluster edit
  form are collapsed by default for a cleaner edit experience.
- **Navigation** — the sidebar footer shows the running version and short commit hash
  (e.g. `v0.10.3+abc1234`).

---

## Quick Start

1. Extract the ZIP to your app server.
2. Run `scripts\Setup-Prerequisites.ps1` as Administrator.
3. Run `scripts\Install.ps1` as Administrator.
4. Edit `AzureLocal.ClusterTool.Web\appsettings.Production.json` with your Entra ID, database, and group settings.
5. Edit `AzureLocal.ClusterTool.Web\clusters.json` with your cluster addresses.
6. Run `scripts\Setup-IIS.ps1` as Domain Admin.

See [`docs/QUICK-START.md`](docs/QUICK-START.md) for a step-by-step guide or
[`docs/IIS-Deployment.md`](docs/IIS-Deployment.md) for full deployment detail.

## Requirements

- Windows Server 2022+ with IIS
- .NET runtime **not** required — self-contained build
- gMSA (or service account) with WinRM access to cluster nodes (HTTP 5985 / HTTPS 5986)
- Entra ID app registration with `groupMembershipClaims: SecurityGroup`

## Reporting issues

Use the **Issues** tab — templates are provided for bug reports and feature requests.
