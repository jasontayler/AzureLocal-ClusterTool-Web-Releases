# Release Notes — Azure Local Cluster Tool (Web)

---

## v0.12.4 — 2026-05-15

### New — Maintenance Windows

A new **Admin &rarr; Maintenance Windows** page lets HciAdmin users configure windows that
suppress alerting for a cluster (or all clusters) during planned maintenance.

**Two window types:**

- **One-time windows** — start time with an optional end time. Leave the end blank for an
  indefinite mute. Useful for incidents or ad-hoc work outside the normal schedule.
- **Recurring rules** — four recurrence types:
  - `Daily` — fires every day
  - `Weekly` — fires on selected days of the week (e.g. every Friday)
  - `MonthlyByDay` — fires on a specific day number each month (e.g. the 1st)
  - `MonthlyByOrdinal` — fires on a specific weekday occurrence (e.g. the 2nd Tuesday)

  Rules generate occurrences 90 days ahead automatically. Each occurrence can be
  enabled/disabled individually; the rule itself can also be paused without deleting it.

**Grouped schedule view:**

Recurring rules that share the same pattern and reason are displayed as a single
**Reoccurring Schedule** row. Click the row to expand a fly-out table showing each cluster's
individual rule with Enable/Disable, Edit, and Delete controls. Expired multi-cluster windows
show a single group-level **Delete** button to remove the whole group in one action.

**REST API:**

Full CRUD is available via the REST API at `/api/maintenance/windows` and
`/api/maintenance/rules`. The `windowStart` field uses a human-readable `HH:mm` format
(e.g. `"22:00"`) and `daysOfWeek` uses comma-separated day names. See
[API.md](https://github.com/jasontayler/AzureLocal-ClusterTool-Web/blob/main/docs/API.md)
for the full reference.

**`Manage-Maintenance.ps1`** is included in `scripts/` as a ready-to-run PowerShell helper
for the full maintenance window lifecycle via the API.

### Security

API layer, authentication, response headers, and audit subsystem hardening. No user-visible
changes.

### Documentation

- API.md updated with full maintenance window and recurring rule endpoint reference,
  field tables, PowerShell examples, and multi-cluster schedule patterns.

---

## v0.12.3 — 2026-05-09

### New — Storage Performance timeframe selector and sparklines

The Cluster Performance page now matches the Node Performance pattern. A timeframe dropdown
(Most Recent / Last Hour / Last Day / Last Week / Last Month) and an explicit **Load Performance**
button replace the previous auto-load behaviour. Sparklines appear on IOPS, Latency, and
Throughput columns when a historical timeframe is selected. An amber notice is shown for
historical timeframes to indicate that data comes from the collector's snapshot history.

### New — Node Performance added to sidebar navigation

The Node Performance page is now directly accessible in the **Cluster** section of the sidebar
(listed after Events). Previously it was only reachable via the tab bar on Cluster Info
sub-pages.

### Improved — Cluster Info page card styling

The four summary cards (Cluster Summary, Nodes, Quorum, VM Load Balancer) and the Health
section now use the same `overview-card` design as the Overview page. The Cluster Shared
Volumes and Cluster Networks sections at the bottom have also been updated to `overview-card`
panels with direct `Storage →` and `Network →` header links.

### Fixed — Storage Performance stale "No volume data" message

When `GetVolumePerfAsync` returned zero rows on first call (possible while performance history
initialises), the page would show a permanent dead-end message. Replaced with an inline
**Retry** button.

### Fixed — Cluster Info comment text visible in browser

Closing-brace comments left in Razor markup context were rendered as visible text in the
browser. Removed.

### Documentation

- USER-GUIDE.md updated with new sections: Health Settings, Security & Compliance, Node
  Performance, Diagnostic Logs, Platform Topology, Fleet VM Status, Fleet Update Status,
  Update Schedules, and Reports.
- API.md updated: 429 rate-limit response code added to the error table; new Rate Limiting
  section documents the 60 req/min fixed-window limit and `Retry-After` header.
- Access Levels section updated to reflect the current two-group model (HciAccess / HciAdmin).

---

## v0.12.2 — 2026-05-06

### New — Clone Custom Role

A **Clone** button on each role row in Admin &rarr; Custom Roles pre-populates the new-role
form with the source role's name (`"[Name] (Copy)"`), description, and all permission claims.
The cloned role is saved with a new ID — the original is unchanged. Group assignments are
intentionally not copied. The operation is recorded in the audit log.

### Breaking Change — New Entra Application Permission Required

> **Action required before upgrading if you use the group search in Admin &rarr; Custom Roles.**

The `GraphService` has been rewritten to use the **app-only** (client credentials) token flow
instead of a delegated (OBO) token. The permission type changes:

| | Before | After |
|---|---|---|
| Permission type | Delegated | **Application** |
| Permission name | `Group.Read.All` | **`Group.Read.All`** |

**Steps to grant the permission:**

1. Entra Portal &rarr; **App registrations** &rarr; select your app
2. **API permissions** &rarr; Add a permission &rarr; Microsoft Graph
3. Choose **Application permissions** (not Delegated)
4. Select **`Group.Read.All`** &rarr; Add permissions
5. Click **Grant admin consent for [tenant]**
6. The Client Secret must be set in **Admin &rarr; Settings &rarr; Entra App Client Secret**

If the Application permission is not granted, group search will show an error and fall back to
the manual Object ID input — existing role assignments are unaffected.

### Fixed — Entra group search returning no results

Group search was silently returning empty results even when the Client Secret was correctly
configured. Root cause: Blazor Server SignalR handlers run without an active `HttpContext`,
causing the OBO token acquisition to fail silently. `GraphService` now uses
`AcquireTokenForClient` (app-only), which works correctly from any Blazor event handler.

### Fixed — Group search dropdown rendered as a black box

The dropdown had a hardcoded dark background making results unreadable. Background, border,
shadow, and hover colours updated to use the app's light-theme CSS.

### Fixed — Search box pre-filled with browser URL history

The input inherited a monospace font which Chrome auto-filled with URL history. Fixed by
overriding `font-family: inherit` and switching to `autocomplete="new-password"`.

---

## v0.12.1

### Bug fixes and configuration improvements

This release focuses on authentication and RBAC configuration fixes surfaced through
customer testing.

### Fixed — Sign Out button on the Access Denied page

The Sign Out button on the `/access-denied` page did not work — clicking it appeared
to do nothing because Blazor enhanced navigation intercepted the click instead of
following the Entra sign-out redirect. Fixed with the same `data-enhance-nav="false"`
correction applied to the nav bar sign-out link in v0.12.0.

### Fixed — Access Denied page listed outdated group names

The Access Denied page still referenced the old three-group model (HCI Read / HCI
Operate / HCI Admin) from before v0.11. Updated to reflect the current two-group
model (HCI Access / HCI Admin).

### Fixed — Custom RBAC role assignments silently having no effect

With `groupMembershipClaims: "ApplicationGroup"` set in the Entra app manifest
(the recommended setting to avoid oversized tokens), a group's Object ID only
appears in a user's JWT token if that group is explicitly assigned to the
**Enterprise Application** under Entra Portal → Enterprise Applications →
[app] → Users and groups.

This requirement applies to **every group** used in the app — not only the login
gate groups (HciAccess / HciAdmin) but also every Entra group assigned to a
Custom Role. If a custom RBAC group is not listed in the Enterprise Application,
users in that group see no clusters and receive no role badge, even after signing
out and back in.

Two changes make this requirement visible at the point it matters:

- A persistent warning now appears in Admin → Custom Roles directly below the
  group assignment field.
- The Access Denied page (Step 2) now explicitly calls this out with the correct
  remediation steps.

### New — Entra App Client Secret configurable via Admin → Settings

The `AzureAd:ClientSecret` field is now visible in Admin → Settings under the
**Azure ARM Authentication** group, labelled **Entra App Client Secret**.

This secret enables two features:
- **Graph group name search** in Admin → Custom Roles — type a group name to find
  it instead of pasting an Object ID manually.
- **ARM OBO mode** for Azure Arc pages (if `ArmAuthMode` is set to `OBO`).

Storing the secret via Admin → Settings is the recommended approach — it is
DPAPI-encrypted in the database and never touches a config file on disk. An IIS
app pool recycle is required once after saving.

### Configuration clarifications

- `appsettings.json` and `appsettings.Production.json` now clearly document that
  `Groups:HciAccess` and `Groups:HciAdmin` hold **Entra group Object IDs**, not
  the App Role IDs visible in App registrations → App roles.
- `AzureAd:ClientSecret` has been removed from the settings file templates — the
  comment directs operators to Admin → Settings instead.
- Admins only need to be a member of the `HciAdmin` group — membership in
  `HciAccess` is not also required, as the access policy accepts either group.

---

## v0.12.0

### New — Unclustered VM detection and Add-to-Cluster

VMs that are running on a node but not registered as a cluster resource are now
highlighted on the Virtual Machines page with an orange **Unclustered** badge.

**What this means:**  
An unclustered VM has no automatic failover protection — if the host node goes offline,
the VM will not restart on another node. The badge makes this risk immediately visible
without needing to cross-reference Failover Cluster Manager.

**Add to Cluster:**  
An **Add to Cluster** button in the Actions column opens a confirmation modal and
registers the VM as a Highly Available cluster resource (`Add-ClusterVirtualMachineRole`).
The operation completes in seconds and the badge disappears on the next refresh.

**Exclusion patterns:**  
Set `NonClusteredVmPatterns` in Admin > Settings to a comma-separated list of name
patterns (e.g. `Mgmt-*,*-Utility`) to suppress the badge for VMs that are intentionally
unclustered.

### New — DDA VM detection

VMs with a PCI device directly assigned (Discrete Device Assignment) now show a **DDA**
badge. Live migration is automatically blocked for these VMs — the Live Migrate button is
disabled because DDA VMs cannot be moved while powered on.

### New — Global search (previously staged as v0.11.2)

A search box in the top bar lets you find clusters, VMs, and nodes instantly from any
page. Results come from database snapshots — no WinRM calls, instant response. Click a
result to navigate directly to that resource.

### Bug fixes

- **RBAC group search now works** — the Microsoft Graph `Group.Read.All` scope was
  missing, causing the group picker in Admin > Roles to return no results. Fixed.
- **ARM Auth settings** — the OBO mode option has been removed from Admin > Settings.
  SPN remains the only supported ARM authentication mode.

---

## v0.11.1

### New feature — Daily Health Digest

A new built-in scheduled service sends a once-per-day management-tool health summary
via email and/or Teams at a configurable UTC time (default 07:00).

**Each digest includes:**
- Server hostname, tool version, and process uptime
- Per-cluster status: healthy / unreachable / disabled with last-poll timestamp
- Alert rule summary: enabled/disabled counts and global alerting on/off status
- Recent errors: any cluster poll failures from the last 24 hours

**Delivery** uses the same SMTP and Teams channels as existing alerts. Both channels
fire independently based on what is configured. If neither is configured, the digest
is silently skipped.

**Configuration** via Admin > Settings > Daily Digest:

| Setting | Default | Description |
|---|---|---|
| `Digest:Enabled` | `false` | Set to `true` to activate |
| `Digest:SendTimeUtc` | `07:00` | UTC 24-hour send time (HH:mm) |
| `Digest:EmailTo` | _(global SMTP To)_ | Per-digest email override |
| `Digest:WebhookUrl` | _(global Teams URL)_ | Per-digest Teams webhook override |

Disabled by default. No emails are sent until `Digest:Enabled = true`.
Set `Digest:EmailTo = none` or `Digest:WebhookUrl = none` to suppress a specific
channel for the digest without affecting alert delivery on that channel.

---

## v0.11.0

> **BREAKING CHANGE — authentication configuration update required before upgrading.**
> See migration steps below.

### BREAKING CHANGE — Auth group configuration simplified (2-group model)

The three-group model (`HciRead` / `HciOperate` / `HciAdmin`) is replaced by two groups:
**`HciAccess`** and **`HciAdmin`**. All operational permissions (Start VM, Drain Node, etc.)
are now controlled entirely by custom Roles in Admin > Roles, not by group membership tiers.

**Migration steps (required before or immediately after upgrading):**

1. In `appsettings.Production.json` on the server, add the `HciAccess` key:
   ```json
   "Groups": {
     "HciAccess": "<same-OID-as-your-HciRead-group>",
     "HciAdmin":  "<same-OID-as-before>"
   }
   ```
2. `HciRead` and `HciOperate` entries can be removed, or left in place — they are ignored.
3. In **Entra Portal > App registrations > Manifest**, change:
   `"groupMembershipClaims": "SecurityGroup"` to `"groupMembershipClaims": "ApplicationGroup"`
   Then ensure HciAccess and HciAdmin are assigned under **Enterprise Applications > your app > Users and groups**.
4. Recycle the IIS app pool.
5. All previously HciRead and HciOperate members retain their current access — no user changes required.

**Backward compatibility:** If `Groups:HciAccess` is absent, the app automatically falls back
to reading `Groups:HciRead`, so the app continues to work before the config is updated.

**`ApplicationGroup` is strongly recommended** — it prevents the HTTP 400 "request headers too long"
error that occurs when `SecurityGroup` emits claims for every tenant group in the auth cookie.

### Bug Fixes

- **Fleet VM Status — VM link opens unfiltered list (#16)** — clicking a VM name on the Fleet VM
  Status page now opens the Virtual Machines page pre-filtered to that VM instead of showing all VMs.

- **VM Checkpoints tab disappears on navigation (#23)** — the Checkpoints sub-nav tab now persists
  correctly when navigating to VM Performance and Virtual Switches.

- **VM Checkpoints and VM Performance slow on large clusters (#25)** — both pages now wait for
  an explicit user action before connecting to the cluster. VM Checkpoints shows a description
  panel on load; VM Performance shows a VM selector before fetching counters.

- **Missing confirmation dialogs on destructive actions (#19)** — modal confirmations added to:
  VM Stop, Restart, Start, Resume, Delete Checkpoint; Stop Cluster Role; Delete Role,
  Remove Permission, Remove Group Assignment (Admin > Roles); Clear Setting, Generate API Key
  (Admin > Settings); Enable/Disable Cluster (Admin > Clusters).

- **Fleet Status: non-sortable columns + Azure Local and Hyper-V mixed (#20)** — Fleet Status
  column headers are now sortable. Standalone Hyper-V hosts appear in a separate section below
  the Azure Local clusters table.

---

## v0.10.12

### Bug fix — VM Checkpoints ObjectDisposedException in background poller

The VM Checkpoints background poller could corrupt the shared CimSession cache when running
concurrently with a live page load, producing `ObjectDisposedException: CimSession: hostname`
errors in the Perf Debug log and causing VM detail pages to return empty data.

The fix switches the poller from `-CimSession` (which caches the session object inside the
runspace) to `-ComputerName`. CDXML creates and manages its own transient session per call,
completely isolated from the shared node session cache.

---

## v0.10.11

### VM Checkpoints moved to sub-nav tab

VM Checkpoints is now a tab within the Virtual Machines sub-nav (alongside VM Performance
and Virtual Switches) rather than a separate sidebar navigation link, consistent with the
pattern used by Virtual Switches.

### Checkpoints count pill on Overview

The Virtual Machines workload card on the cluster Overview page now shows an orange
**Checkpoints** count pill when any checkpoints are present. The pill reads from the
existing background snapshot (no extra live query) and links directly to the Checkpoints tab.

### Bug fix — Network ATC alert false positives

The Network Intent Degraded alert rule was using an exclusion list of known-healthy status
strings. Any status not on the list — including `Completed` (returned by some Azure Local
builds) and transient states like `Provisioning`, `Retrying`, `Pending` — incorrectly
triggered the alert. Fixed by switching to an allow-list: the alert now fires only when
`Status == "Failed"` or `Status == "Error"`, eliminating false positives during ATC
convergence after node reboots or cluster updates.

---

## v0.10.10

### Bug fix — wsmprovhost.exe accumulation (PriorityQueue thread-safety)

This release fixes the root cause of all persistent `wsmprovhost.exe` accumulation.
Previous fixes (v0.10.6 through v0.10.9) addressed symptoms; this addresses the
underlying structural defect in the polling engine.

**Root cause:** `PriorityQueue<T,P>` in .NET is not thread-safe. The poller's
`Task.Run` poll tasks called `queue.Enqueue()` directly from thread pool threads
while the main loop simultaneously called `queue.TryDequeue()`, `queue.TryPeek()`,
and `ResyncQueue`'s full drain-and-rebuild on another thread. Concurrent heap
modifications silently corrupted the internal array, producing duplicate entries for
the same cluster. `ResyncQueue` (running every 30 s) faithfully re-enqueued all
copies without deduplication, making duplicates permanent once created.

Each duplicate entry produced an independent `PollClusterAsync` task per cycle, each
creating its own `WebHyperVService` instance with a separate `_cimSession` connection
to the cluster — one `wsmprovhost.exe` per task per cluster owner node. Clusters
with more activity (SV1405) were hit more often and accumulated more duplicates,
explaining why SV1405CH002 showed 12 processes while SV1407CH001 showed 1.

**Fix:**
- `_pendingRequeue` (`ConcurrentQueue`) added: Task.Run finallys post completed
  clusters here instead of writing to the PriorityQueue directly.
- The main loop drains `_pendingRequeue` at the top of every iteration, making it
  the sole writer to the PriorityQueue. No more concurrent access.
- `_inFlight.Remove` deferred to the drain so clusters stay protected until
  safely back in the queue.
- `ResyncQueue` now deduplicates by cluster name (earliest due time wins) on every
  rebuild — any pre-existing duplicates are cleaned up within 30 s of deployment.
- `lastRegistrySync` initialised to `UtcNow` to skip the redundant immediate
  `ResyncQueue` call on the very first loop iteration.

After deploying, kill any remaining orphans with
`Get-Process wsmprovhost | Stop-Process -Force` on each node.
Process counts should stabilise at 1-2 per node permanently.

---

## v0.10.9

### Bug fix — wsmprovhost.exe steady accumulation (per-node CimSession TTL)

v0.10.8 fixed the duplicate-poll race but wsmprovhost counts continued to grow steadily.
The remaining cause was the poller's per-node CimSession TTL of 10 minutes. A background
eviction sweep runs every 30 seconds and calls `Dispose()` on sessions older than the TTL.
When the remote node is slow or does not acknowledge the WSMan DELETE message, the
server-side `wsmprovhost.exe` process is not terminated and lingers for the full 2-hour
WinRM idle timeout. With a 10-minute evict cycle, approximately one new orphan is created
per node every 10 minutes. In practice this produced counts of 8-9 processes on the busier
nodes (SV8403, SV1418) after 100 minutes of uptime.

The fix sets the poller TTL to 4 hours, which is longer than the WinRM server-side idle
timeout. The eviction timer will never fire during normal operation. Each poller connection
now maintains exactly 1 long-lived wsmprovhost per cluster node. Sessions are only
closed when the cluster goes idle (poller idle-evict) or when an error forces a retry.

After deploying, kill any remaining orphans with
`Get-Process wsmprovhost | Stop-Process -Force` on each node, then observe counts
stabilise at 1-2 per node and remain flat.

---

## v0.10.8

### Bug fix — wsmprovhost.exe accumulation (poller duplicate-queue race)

v0.10.7 reduced the process count but did not stop accumulation entirely. The root cause
was a race condition in the polling engine: when a cluster was dequeued from the priority
queue, it entered a brief window where it was neither in the queue nor marked in-flight.
The background registry sync (`ResyncQueue`) runs every 30 seconds and treats any cluster
not in the queue or in-flight as newly registered — re-adding it to the queue. At startup,
when all clusters fire simultaneously and the concurrency semaphore is saturated, a cluster
could sit in this unprotected window for many seconds, allowing `ResyncQueue` to insert
2 or 3 duplicate entries. Each duplicate triggered an independent poll task which opened
its own CimSession to the cluster, spawning an additional `wsmprovhost.exe` per node.
This matched the observed pattern exactly: SV1405 (more activity) accumulated 23 processes;
SV1407 (less activity) stayed at 7-13.

The fix moves the in-flight registration to immediately after dequeue, before the semaphore
wait, so the cluster is protected for the entire time it is being processed.

After deploying, kill any remaining orphans with
`Get-Process wsmprovhost | Stop-Process -Force` on each node, or wait for the 2-hour
WinRM idle timeout.

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
