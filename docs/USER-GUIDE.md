# Azure Local Management (ALM) — User Guide

> **Audience:** Cluster operators and administrators.  
> **App URL:** Set by your administrator during IIS setup — typically `https://azlmgmt.<your-domain>/`  
> **Authentication:** Microsoft Entra ID (Azure AD) single sign-on — no username/password prompt inside the app.

---

## Contents

1. [Signing In](#1-signing-in)
2. [Cluster Selection — Home Page](#2-cluster-selection--home-page)
3. [Navigation](#3-navigation)
4. [Status Indicators](#4-status-indicators)
5. [Virtual Machines](#5-virtual-machines)
   - [5a. VM Performance](#5a-vm-performance)
   - [5b. Virtual Switches](#5b-virtual-switches)
   - [5c. VM Checkpoints](#5c-vm-checkpoints)
6. [Cluster Nodes](#6-cluster-nodes)
7. [Cluster Roles](#7-cluster-roles)
8. [AKS on Azure Local](#8-aks-on-azure-local)
9. [Cluster Info](#9-cluster-info)
   - [9a. Security & Compliance](#9a-security--compliance)
10. [Storage](#10-storage)
11. [Storage QoS](#11-storage-qos)
12. [Network](#12-network)
13. [Events](#13-events)
14. [Remote Log Viewer](#14-remote-log-viewer)
15. [Solution Updates](#15-solution-updates)
16. [Azure Arc](#16-azure-arc)
17. [Arc Resource Bridge](#17-arc-resource-bridge)
18. [Custom Locations](#18-custom-locations)
19. [Agent Services](#19-agent-services)
20. [Admin — Clusters](#20-admin--clusters)
21. [Admin — Audit Log](#21-admin--audit-log)
22. [Admin — Settings](#22-admin--settings)
23. [Admin — Custom Roles (RBAC)](#23-admin--custom-roles-rbac)
24. [Access Levels](#24-access-levels)
25. [Frequently Asked Questions](#25-frequently-asked-questions)
26. [Windows Authentication Deployment](#26-windows-authentication-deployment)
27. [Admin — Diagnostics & Background Collector](#27-admin--diagnostics--background-collector)
28. [Fleet Status Board](#28-fleet-status-board)
   - [28a. Fleet Schedules](#28a-fleet-schedules)
29. [Admin — Alerts](#29-admin--alerts)

---

## 1. Signing In

Navigate to the app URL in any modern browser. You will be redirected to the **Microsoft Entra ID** sign-in page automatically. Use your normal domain or Microsoft 365 credentials.

After sign-in you are returned to the app home page. No separate app password is required.

**If access is denied:** Your account is not a member of a permitted Entra security group. Contact your administrator to be added to the appropriate group (see [Access Levels](#24-access-levels)).

**Sign out:** Click your name in the top-right corner and select **Sign out**, or navigate directly to `/signout`.

**Your role badge:** After signing in, your assigned custom RBAC role name (e.g. `VM Admin`) is displayed as a small badge next to your name in the top bar. HciAdmin members see an `Admin` badge. In pass-through mode (no custom roles configured) no badge is shown.

---

## 2. Cluster Selection — Home Page

The home page lists every cluster you have access to. Click any cluster card to connect and begin managing it. The selected cluster name appears in the left navigation bar for the rest of your session.

Use the **Filter clusters...** search box to narrow the list by cluster name or address — useful when many clusters are registered.

If no clusters are listed, either no clusters are registered (an Administrator must add them via [Admin → Clusters](#20-admin--clusters)) or your custom RBAC role does not include **View** access to any clusters — contact your administrator to update your role's `Clusters` permissions.

---

## 3. Navigation

The left sidebar has two sections:

**Cluster section** (appears after selecting a cluster)

The sidebar groups cluster links into labelled sections:

| Link | Page |
|---|---|
| 🏠 All Clusters | Return to the cluster picker home page |
| 📋 Fleet Status | Fleet-wide VM/node/health/updates dashboard — all clusters, zero WinRM, DB-only |
| � Fleet Schedules | Pending and historical update schedules across all clusters |
| �📊 Overview | Summary cards — VM, role health, and Azure Arc status at a glance |
| **Compute** | |
| 🖥️ Virtual Machines | Full VM list with actions |
| &nbsp;&nbsp;↳ 📈 VM Performance | Per-VM CPU/memory/disk/network charts (sub-link) |
| &nbsp;&nbsp;↳ 🌐 Virtual Switches | Hyper-V virtual switches per node (sub-link) |
| &nbsp;&nbsp;↳ 📋 VM Checkpoints | All VM checkpoints/snapshots across the cluster (sub-link) |
| 🖧 Cluster | Cluster-wide summary — health faults, quorum, CSV volumes |
| &nbsp;&nbsp;↳ 💻 Cluster Nodes | Node health and drain operations (sub-link) |
| &nbsp;&nbsp;↳ 🔧 Cluster Roles | Cluster resource groups (sub-link) |
| &nbsp;&nbsp;↳ 💾 Cluster Storage | CSV volumes and storage pools (sub-link) |
| &nbsp;&nbsp;↳ 📊 Cluster Performance | Storage QoS and node performance metrics (sub-link) |
| &nbsp;&nbsp;↳ 📡 Cluster Network | Cluster network health (sub-link) |
| &nbsp;&nbsp;↳ 📋 Cluster Events | Windows cluster event log (sub-link) |
| ⚓️ AKS | AKS on Azure Local clusters — ARM-sourced overview |
| **Storage** | |
| 💾 Storage | Storage pools, virtual disks, physical disks — includes Storage QoS tab |
| **Network** | |
| 📡 Network | Physical network adapters — includes ATC Intents, Cluster Networks, SMB Networks, Logical Networks tabs |
| **Tools** | |
| 📄 Remote Logs | Browse and read log files on cluster nodes |
| ⚙️ Agent Services | HCI agent service status and control |
| **Platform** | |
| 🔄 Solution Updates | Available updates and update run history |
| ☁️ Azure Arc | Arc registration status — includes Arc Machines, Arc Extensions, Cluster Extensions tabs |
| &nbsp;&nbsp;↳ 🧱 Arc Resource Bridge | Arc appliance health and Azure Local sites (dedicated page) |
| &nbsp;&nbsp;↳ 📍 Custom Locations | Azure Arc custom locations (dedicated page) |
| 🔒 Security & Compliance | Per-node security features, WDAC, BitLocker, and drift detection |

**Sub-navigation strips**

Several page groups share a tab strip directly below the page heading, so you can switch between related pages without going back to the sidebar:

- **Virtual Machines group:** Virtual Machines · VM Performance · Virtual Switches · VM Checkpoints
- **Cluster group:** Cluster Info · Nodes · Roles · Storage · Performance · Events
- **Storage group:** Storage · Storage QoS
- **Network group:** Adapters · ATC Intents · Cluster Networks · SMB Networks · Logical Networks
- **Azure Arc group:** Registration · Arc Machines · Arc Extensions · Cluster Extensions

**Arc Resource Bridge** and **Custom Locations** are separate sidebar entries (indented below Azure Arc), not tabs.

**Admin section** (HciAdmin group only)

| Link | Page |
|---|---|
| ⚙️ Clusters | Add / edit / delete registered clusters |
| 📓 Audit Log | Complete history of all operator actions |
| 🔑 Settings | Encrypted application settings (ARM auth, SPN credentials) |
| 🛡️ Custom Roles | Fine-grained RBAC role management |
| 🔬 Diagnostics | PS/WinRM call log, cluster perf debug, background collector health |

**Custom role nav visibility**

If your account is subject to [Custom Roles (RBAC)](#23-admin--custom-roles-rbac), RBAC controls access at two levels:

1. **Cluster visibility** — which clusters appear in the home page picker and in the left navigation bar. Controlled by your role's `Clusters` permission. If a cluster does not appear in your picker, your role's `Clusters` pattern does not match it.
2. **Within a cluster** — which resource type links (VMs, Nodes, Storage, etc.) are shown. Controlled by the corresponding resource type permission on your role.

Section headers (*Compute*, *Storage*, *Network*, *Tools*, *Platform*) are hidden when all links within them are hidden.

If you navigate directly to a URL for a cluster or page you cannot access, you will see an access-denied message rather than data.

The **Overview** page respects the same rules: summary cards for Virtual Machines, Cluster Roles, and Azure Arc status are only shown if your role includes View access to those resource types. Items within each card are additionally filtered by any name patterns in your role (e.g. `PROD-*`).

> In pass-through mode (no custom role assignments configured), the full navigation is always shown and all clusters are visible.

### Global Search

A search box is always visible in the top bar (magnifying glass icon). Type two or more characters to search across all registered clusters simultaneously — results are grouped into **Clusters**, **Virtual Machines**, and **Nodes**.

- Results are sourced from the background collector's database snapshots — the search is instant and does not make any live WinRM connections.
- Click any result to navigate directly to that resource: cluster results open the cluster Overview, VM results open the Virtual Machines page pre-filtered to that VM, and node results open the Cluster Nodes page.
- Results respect your [Custom Role (RBAC)](#23-admin--custom-roles-rbac) permissions: clusters and VMs outside your allowed name patterns are never returned.
- Keyboard: press **Escape** to dismiss the dropdown; the input retains focus so you can type a new query immediately.

---

## 4. Status Indicators

All pages use coloured dots to indicate health or state at a glance.

| Colour | Meaning |
|---|---|
| 🟢 Green | Running / Healthy / Up / Succeeded / Connected |
| 🟠 Orange | Paused / Warning / InProgress / Pending / Degraded |
| 🔴 Red | Off / Failed / Critical / Disconnected |
| ⚫ Grey | Unknown / Other |

Hover over any dot to see the full status text as a tooltip.

---

## 5. Virtual Machines

**Route:** `/clusters/{name}/vms`  
**Access required:** HciRead (view) · HciOperate (Start/Stop/Migrate)

> The **Virtual Machines · VM Performance · Virtual Switches** tab strip at the top lets you switch between all three pages without returning to the sidebar.

> **Custom role filtering:** If your role includes a name pattern (e.g. `PROD-*`), only VMs whose names match that pattern are shown. VMs outside the pattern are completely hidden. If your role does not include View access to VMs at all, this page shows an access-denied message and the sidebar link is hidden.

Displays all VMs across every node in the cluster.

### Columns

| Column | Description |
|---|---|
| Name | VM name — click the row to expand network adapter details |
| State | Running / Off / Paused / Saved (with status dot) |
| CPU % | Current processor utilisation |
| Memory | Assigned / demand in MB or GB |
| Uptime | Time since last power-on |
| Host | The cluster node the VM is currently running on |
| Actions | Context-sensitive operation buttons |

### Viewing Network Adapters (inline flyout)

Click any VM row to expand an inline panel showing all virtual network adapters for that VM:

- **Adapter name**, connection status (dot + text)
- **MAC address**, virtual switch, VLAN mode/ID
- **IP addresses** (IPv4 summary)

Click the row again to collapse. Adapter data is fetched on first expand and cached for the duration of the page session.

### VM Actions

Action buttons appear based on the VM's current state and your permissions:

| Button | Available when | Effect |
|---|---|---|
| ▶ Start | VM is Off or Saved | Powers on the VM |
| ↺ Restart | VM is Running or Paused | Sends a restart command |
| ⏸ Suspend | VM is Running | Saves VM state (pauses without power-off) |
| ⏹ Stop | VM is Running, Paused, or Saved | Graceful shutdown |
| ⇒ Live Migrate | VM is Running, cluster has >1 node | Opens the node picker modal |

After an action the page automatically polls the cluster for 15 seconds (every 3 seconds) to pick up state transitions — you do not need to manually refresh.

### Live Migration

Click **⇒ Live Migrate** on a running VM. A modal dialog appears listing all other nodes in the cluster. Select the target node and click **Migrate**. The VM moves online with no guest disruption.

> Actions that fail will show a red error banner above the table. Check the [Audit Log](#21-admin--audit-log) for detail.

### Connect Shortcuts

> **Note:** All actions are recorded in the [Audit Log](#21-admin--audit-log) with the acting user, outcome, and duration.

Click any VM row to expand it, then scroll to the **Connect** section at the bottom of the flyout:

| Button | What it does |
|---|---|
| **RDP** | Downloads a pre-filled `.rdp` file for `mstsc.exe`. Windows auto-opens it — click **Connect** in the RDP dialog. Connects to the VM by **name** (requires the VM's guest OS to accept RDP). |
| **🖥 VMConnect** | Downloads a small `.bat` file. **Double-click** it to open the Hyper-V VM console (requires Hyper-V management tools — `vmconnect.exe` — installed on your local machine). Unlike RDP, VMConnect gives you a direct console session regardless of the guest OS network configuration. |

> **Why double-click for VMConnect?** Browsers can auto-open `.rdp` files because Windows has a built-in file association for them. `vmconnect.exe` has no equivalent — Windows cannot launch it directly from a browser download. Double-clicking the saved `.bat` is the closest one-click experience available without a custom registry URI handler.

### Unclustered VMs

Some VMs may be running on a cluster node but not registered as part of a Cluster Group. These VMs are shown in the table with an orange **Unclustered** badge in the Host column.

An unclustered VM:
- Is not protected by the cluster — if the host node goes offline the VM will not automatically fail over to another node.
- Cannot be live-migrated through the normal cluster migration path.

**Add to Cluster:** Click the **Add to Cluster** button (shown in the Actions column for unclustered VMs) to register the VM as a Highly Available (HA) cluster resource. A confirmation modal appears — click **Add to Cluster** in the modal to proceed. The operation calls `Add-ClusterVirtualMachineRole` on the cluster.

> **Pre-flight check:** If the host node is in a draining state or cannot be reached, the operation may fail. Check the [Audit Log](#21-admin--audit-log) for detail.
>
> **Exclusion patterns:** Administrators can configure `NonClusteredVmPatterns` (a comma-separated list of name patterns) in Admin → Settings to suppress the badge and button for VMs that are intentionally unclustered (e.g. management VMs that should not be HA).

### DDA (Device Assignment)

VMs with a **DDA** badge have a PCI device directly assigned via Discrete Device Assignment. These VMs cannot be live-migrated — the **Live Migrate** button is disabled to prevent the operation from failing. Saving the VM state and moving it manually is the recommended approach.

---

## 5a. VM Performance

**Route:** `/clusters/{name}/vms/perf`  
**Access required:** HciRead

> Part of the **Virtual Machines · VM Performance · Virtual Switches** tab group.

Displays performance charts per virtual machine gathered directly from the Hyper-V host. Select a VM from the table on the left to view its charts on the right.

| Chart | Description |
|---|---|
| CPU % | Processor utilisation over time |
| Memory MB | Assigned memory and demand |
| Disk IOPS | Read and write operations per second |
| Network throughput | Bytes in/out per second across all virtual adapters |

---

## 5b. Virtual Switches

**Route:** `/clusters/{name}/virtual-switches`  
**Access required:** HciRead

> Part of the **Virtual Machines · VM Performance · Virtual Switches** tab group.

Lists all Hyper-V virtual switches across every node in the cluster.

### Columns

| Column | Description |
|---|---|
| Name | Virtual switch name |
| Type | External / Internal / Private |
| Node | The cluster node that hosts this switch |
| SET | Whether Switch Embedded Teaming is enabled (blue badge when yes) |
| Management OS | Whether the host management OS shares this switch (Yes / No) |
| Adapters | Physical network adapters the switch is bound to |
| Notes | Additional switch notes or description |

> Use this page to verify switch names and teaming configuration before troubleshooting VM network issues.

---

## 5c. VM Checkpoints

**Route:** `/clusters/{name}/vm-checkpoints`  
**Access required:** HciRead

> Part of the **Virtual Machines · VM Performance · Virtual Switches · VM Checkpoints** tab group.

Lists all Hyper-V VM checkpoints (snapshots) across every node in the cluster.

> **On-demand load:** Checkpoint data is not fetched automatically when the page opens because enumerating checkpoints requires a WinRM call to every node and can take 10–30 seconds on large clusters. Click the **Load Checkpoints** button to initiate the fetch. Subsequent **Refresh** clicks re-fetch immediately.

### Columns

| Column | Description |
|---|---|
| VM | VM name — click to jump to the Virtual Machines page filtered to that VM |
| Host | The cluster node the VM is running on |
| Checkpoint Name | Snapshot label, with a `(root)` indicator for root checkpoints |
| Type | Checkpoint type — Standard, Production, or ProductionOnly |
| Created | Local timestamp when the checkpoint was taken |
| Age | Time elapsed since the checkpoint was created (colour-coded — orange > 72 h, red > 7 days) |

### Filtering by age

Use the **Age filter** dropdown in the toolbar to narrow results to checkpoints older than 24 hours, 72 hours, or 7 days. The status bar shows how many checkpoints match out of the total.

### Exporting

Click **Export CSV** to download a `.csv` file of all currently visible (filtered) rows.

### Why manage checkpoints?

Long-lived checkpoints grow the VHDX differencing chain, consume additional disk space, and can slow VM performance. The [Fleet Status Board](#28-fleet-status-board) shows a fleet-wide **Stale Checkpoints** pill (VMs with checkpoints older than 72 hours) to surface clusters that need attention. An **Alert rule** type (`VM Checkpoint Stale`) can notify you automatically when checkpoints exceed a configurable age threshold.

---

## 6. Cluster Nodes

**Route:** `/clusters/{name}/nodes`  
**Access required:** HciRead (view) · HciOperate (Pause/Resume)

> The **Cluster Info · Nodes · Roles · Storage · Performance · Events** tab strip at the top lets you move between all cluster sub-pages without returning to the sidebar.

> **Custom role filtering:** Name patterns apply to node names. If your role does not include View access to Nodes, this page shows an access-denied message and the sidebar link is hidden.

Shows live statistics for every node: CPU %, memory usage, uptime, OS version, and combined State/DrainStatus.

### Node Actions

| Action | Description |
|---|---|
| Pause (drain) | Moves all running roles and VMs off the node before pausing. Use before maintenance. |
| Pause (no drain) | Pauses the node immediately without migrating workloads. Quicker but disrupts any running VMs. |
| Resume | Brings the node back online and allows workloads to return. |
| Failback | Moves workloads back to this node that were migrated away during a drain. |

> Never pause more than one node at a time in a two-node cluster — the cluster will lose quorum.

### Connect (RDP)

The **Connect** column shows an **RDP** button for each node. Clicking it downloads a pre-filled `.rdp` file that opens a Remote Desktop connection to the node. Windows auto-launches `mstsc.exe` on download — click **Connect** in the RDP prompt.

---

## 7. Cluster Roles

**Route:** `/clusters/{name}/roles`  
**Access required:** HciRead (view) · HciOperate (Start/Stop/Move)

> The **Cluster Info · Nodes · Roles · Storage · Performance · Events** tab strip at the top lets you move between all cluster sub-pages without returning to the sidebar.

> **Custom role filtering:** Name patterns apply to role/group names. If your role does not include View access to Roles, this page shows an access-denied message and the sidebar link is hidden.

Lists all cluster resource groups (roles): VMs, file servers, generic services, etc.

### Columns

| Column | Description |
|---|---|
| Name | Role/group name |
| State | Online / Offline / Partially Online / Failed (with dot) |
| Owner Node | The node currently hosting this role |
| Resource Type | Type of cluster group |
| Actions | Start / Stop / Move buttons |

### Moving a Role (Failover)

Click **Move** to relocate a running role to another node. A dropdown showing available nodes appears inline. Select the target and confirm. This is equivalent to a manual failover.

---

## 8. AKS on Azure Local

**Route:** `/clusters/{name}/aks`  
**Access required:** HciRead  
**ARM required:** Yes — data is sourced from Azure Resource Manager

> **Custom role filtering:** If your role does not include View access to AKS, this page shows an access-denied message and the sidebar link is hidden.

Shows AKS (Azure Kubernetes Service) clusters deployed on this Azure Local cluster. Data is read from Azure Resource Manager (`microsoft.kubernetes/connectedClusters` + `Microsoft.HybridContainerService/provisionedClusterInstances`). If no AKS clusters are deployed, the page shows an informational message.

### Per-cluster information

Each AKS cluster is displayed as an expandable card showing:

| Field | Description |
|---|---|
| Provisioning State | Current ARM provisioning state (dot + text) |
| Arc Connectivity | Whether the AKS cluster is connected to Azure Arc |
| Kubernetes Version | Kubernetes release version running in the cluster |
| Latest Available | Highest non-preview version available to upgrade to (from upgrade profile) |
| Arc Agent Version | Version of the Arc connectivity agent |
| Last Heartbeat | Most recent connectivity timestamp with Azure |
| Region | Azure region the cluster is registered to |
| Total Nodes | Sum of all nodes across all node pools |
| Azure Monitor | Whether the Azure Monitor extension is installed |
| RBAC | Whether Azure RBAC is enabled on the cluster |
| OIDC Issuer | OIDC issuer URL (if enabled) |

The **🔗 Portal** link opens the AKS cluster resource in the Azure Portal.

### Extensions

Lists Kubernetes extensions installed on the cluster (from `Microsoft.KubernetesConfiguration/extensions`). Each row shows the extension name, type, version, whether auto-upgrade is enabled, and provisioning state.

### Node Pools

A table listing all node pools:

| Column | Description |
|---|---|
| Pool Name | Node pool identifier |
| Nodes | Number of nodes in the pool |
| VM Size | Virtual machine size used for the pool |
| OS | Operating system type |
| State | Provisioning state (dot + text) |

### Kubernetes live data

Namespace and pod data is **not available** via ARM for `ProvisionedCluster` kind on Azure Local.
All ARM resource paths (`/proxy/api/v1/namespaces`, `/namespaces`) return `ResourceTypeRegistrationNotFound` for this cluster type.
The AKS page shows ARM-sourced data only: cluster state, node pools, network/security config, upgrade availability, and installed extensions.

---

## 9. Cluster Info

**Route:** `/clusters/{name}/info`  
**Access required:** HciRead

> The **Cluster Info · Nodes · Roles · Storage · Performance · Events** tab strip at the top lets you move between all cluster sub-pages without returning to the sidebar.

> If your custom role does not include View access to this resource type, this page shows an access-denied message and the sidebar link is hidden.

A summary page showing:

- **Quorum** — current quorum type, witness type, and voting status
- **Nodes** — count of Up/Down nodes
- **S2D (Storage Spaces Direct)** — storage health state
- **Health Faults** — any active health faults reported by the cluster, with fault description and affected object

---

## 9a. Security & Compliance

**Route:** `/clusters/{name}/security`  
**Access required:** HciRead

Provides a per-cluster view of the security posture across four areas: Security Features, WDAC Application Control, BitLocker (data at rest), and Drift Detection.

> ARM-sourced compliance data (policy assignments and compliance state) is shown if Azure ARM is configured in Admin → Settings. The WinRM-sourced sections (Security Features, WDAC per-node mode, BitLocker) load from the background collector snapshot and, if stale, are re-fetched live on page load.

### Security Features

A per-node grid showing the status of each security feature reported by `Get-AzsSecurity` (Azure Local 24H2+ only; older clusters without this cmdlet will show no data in this section).

| Feature | Description |
|---|---|
| Drift Control | Whether the cluster enforces configuration drift protection |
| Credential Guard | Virtualization-based protection for LSASS credentials |
| VBS (Virtualization-based Security) | Hypervisor-protected code integrity |
| HVCI (Hypervisor-protected Code Integrity) | Memory integrity — prevents unsigned kernel code from running |
| DRTM (Dynamic Root of Trust for Measurement) | Hardware attestation of boot sequence |
| Side Channel Mitigation | CPU speculative execution mitigations |
| SMB Signing | Ensures SMB packets are signed to prevent man-in-the-middle attacks |
| SMB Cluster Encryption | Intra-cluster SMB traffic encryption |

Each cell shows a coloured dot (green = enabled, red = disabled/not configured, grey = not available).

### Application Control (WDAC)

Shows the WDAC policy mode per node from `Get-AsWdacPolicyMode`:

| Mode | Meaning |
|---|---|
| Enforced | Windows Defender Application Control is actively blocking unsigned code |
| Audit | WDAC is in audit-only mode — violations are logged but not blocked |
| Not configured | WDAC is not active on this node |

An expandable **ARM policy assignments** panel below the node table shows the cluster-level WDAC assignment (Audit/Enforce/NotConfigured), Secured-Core assignment, and SMB encryption assignment sourced from ARM, along with the last compliance evaluation timestamp. This section is only shown when ARM is configured.

### Data Encryption (BitLocker)

Lists all BitLocker volumes per node gathered via `Get-ASBitLocker` (or `Get-BitLockerVolume` as a fallback). Columns:

| Column | Description |
|---|---|
| Node | Cluster node the volume belongs to |
| Volume | Drive letter or volume label |
| Type | OperatingSystem / Data / Removable |
| Encryption | Encryption status (FullyEncrypted / EncryptionInProgress / Decrypted etc.) |
| Protection | Protection status (On / Off) — a green dot means BitLocker protectors are active |

An ARM compliance indicator above the table shows whether the cluster meets the Azure Local "data at rest encrypted" policy.

### Drift Detection

A collapsible section (click the **Drift Detection** heading to expand). When expanded, the app automatically runs `Invoke-AzStackHciVSRDriftDetectionValidation` against the cluster and shows a per-component pass/fail table comparing installed component versions against the Validated Solution Recipe (VSR) baseline.

Click **Run Drift Check** to manually trigger a fresh check at any time. Results remain visible for the duration of the browser session.

---

## 10. Storage

**Route:** `/clusters/{name}/storage`  
**Access required:** HciRead

Three sections displayed on the page. Use the **Storage / Storage QoS** tab strip at the top to switch to the QoS volumes view.

### Storage Pools

| Column | Description |
|---|---|
| Name | Pool name |
| Health | Healthy / Warning / Unhealthy / Critical (with dot) |
| Operational Status | Operational / Degraded etc. |
| Total Size | Total raw capacity |
| Allocated Size | Space currently allocated to virtual disks |
| Usage | Percentage of pool allocated |

### Virtual Disks

| Column | Description |
|---|---|
| Name | Virtual disk name |
| Health | Health status (dot) |
| Provisioning | Thin / Fixed |
| Size | Logical size |
| Used | Space consumed |
| Resiliency | Mirror / Parity etc. |

### Physical Disks

| Column | Description |
|---|---|
| Friendly Name | Disk model or label |
| Serial Number | Manufacturer serial |
| Health | Health status (dot) |
| Usage | Auto-Select / Hot Spare / Journal / Retired etc. |
| Size | Raw capacity |
| Bus Type | SAS / SATA / NVMe etc. |
| Media Type | SSD / HDD |

---

## 11. Storage QoS

**Route:** `/clusters/{name}/cluster-performance`  
**Access required:** HciRead

Accessible via the **Performance** tab in the Cluster tab strip, or via the **Storage QoS** tab on any Storage page.

Displays Storage QoS volumes with current performance metrics. Each row shows:

- **Volume name** (derived from the VHD file path)
- **Min/Max IOPS** policy values
- **Current IOPS** and **current throughput (MB/s)**
- **Latency (ms)**

Use this page to identify volumes that are hitting their IOPS limit or experiencing high latency.

---

## 12. Network

**Route:** `/clusters/{name}/network`  
**Access required:** HciRead

The Network section spans five pages, switchable via the **Adapters · ATC Intents · Cluster Networks · SMB Networks · Logical Networks** tab strip shown at the top of each page.

### Network Adapters (`/network`)

Physical adapters on each node, showing status, speed, MAC address, IP address, and whether RDMA is enabled.

The **Host** column shows which node the adapter belongs to. Adapters are sorted by node then by name.

### Network ATC Intents (`/network/intents`)

Azure Network Adapter (ATC) intents define which adapters are used for which traffic types (Management, Compute, Storage).

Click any intent row to **expand** a detail panel showing:

- Override settings: adapter overrides, cluster overrides, storage overrides
- VLAN configuration
- Adapter members
- Provisioning / health status

The dot colour reflects the intent's current provisioning health.

### Cluster Networks (`/network/cluster-networks`)

Lists the internal cluster network segments with their role (Cluster-only, Client + Cluster, Client-only) and state.

### SMB Networks (`/network/smb`)

Per-node expandable cards showing RDMA, Jumbo Frames, Flow Control, QoS and PFC adapter configuration, QoS policies, traffic classes, and SMB bandwidth limits. Use this page to verify storage network health across all nodes.

### Logical Networks (`/network/logical-networks`)

**ARM required:** Yes

Lists Azure Local logical networks (`Microsoft.AzureStackHCI/logicalNetworks`) sourced from Azure Resource Manager. Logical networks are used by Arc VMs to connect to the physical underlay network.

Each logical network is displayed as a card showing provisioning state (dot + text), the Hyper-V virtual switch backing the network, and region. A **Subnets** table shows the per-subnet configuration:

| Column | Description |
|---|---|
| Name | Subnet identifier |
| Address Prefix | CIDR notation (e.g. `192.168.1.0/24`) |
| VLAN | VLAN ID |
| DNS Servers | Configured DNS server addresses |
| IP Pool Start | Start of the IP allocation range |
| IP Pool End | End of the IP allocation range |

If ARM is not configured, the tab shows a message indicating ARM credentials are required in [Admin → Settings](#22-admin--settings).

---

## 13. Events

**Route:** `/clusters/{name}/events`  
**Access required:** HciRead

> The **Cluster Info · Nodes · Roles · Storage · Performance · Events** tab strip at the top lets you move between all cluster sub-pages without returning to the sidebar.

Shows the cluster event log — by default the 200 most recent entries. Each entry shows:

| Column | Description |
|---|---|
| Time | Event timestamp |
| Level | Critical / Error / Warning / Information |
| Source | Event log source (e.g. `Microsoft-Windows-FailoverClustering`) |
| Event ID | Numeric event identifier |
| Message | Truncated event message (hover to read full text) |
| Node | The cluster node that generated the event |

### Filtering

Use the **Level** checkboxes in the toolbar to show/hide specific severity levels. The count updates live.

### Export

Click **Export CSV** to download the currently visible events as a CSV file for further analysis in Excel or similar tools.

---

## 14. Remote Log Viewer

**Route:** `/clusters/{name}/logs`  
**Access required:** HciRead

Allows browsing and reading log files directly from cluster nodes without needing RDP or PowerShell access.

### How to use

1. **Select a node** from the dropdown on the left sidebar.
2. The default log directory (`C:\ProgramData\...`) is listed. Type a different path in the text box and click **Browse** to navigate to it.
3. Click any file in the list to read its contents on the right.
4. Use the **Tail** selector to limit output to the last N lines (100 / 500 / 1000 / All).
5. Click **Reload** to refresh the file content (useful for live log tailing).

The file list shows each file's size and last modified time to help identify the most recent logs.

---

## 15. Solution Updates

**Route:** `/clusters/{name}/updates`  
**Access required:** HciRead (view) · HciOperate (start update)

### Available Updates

Lists solution updates available for the cluster. Each row shows:

| Column | Description |
|---|---|
| Display Name | Update package name |
| Version | Package version string |
| State | Ready / Downloading / Installing etc. (with dot) |
| Publisher | Source publisher |
| Release Notes | Link to external documentation (if available) |

Click a row to expand the **detail panel** showing the full description, category, prerequisites, and package ID.

Click **▶ Start Update** to begin installation. You will be prompted to confirm before proceeding.

> Starting an update is an HciOperate-level action and is recorded in the audit log.

### Update Runs

The lower section lists previous update run history with start time, duration, state, and result summary.

Click **📋 View Progress** on any run to open the **Live Monitor** page.

### Live Update Monitor

**Route:** `/clusters/{name}/updates/monitor/{planId}`

Streams live output from the update action plan as it executes. Each line is appended in real time. The stream ends when the plan completes (success or failure). The output is scrollable and colour-coded (errors appear in red).

---

## 16. Azure Arc

The Azure Arc section spans four pages, switchable via the **Registration · Arc Machines · Arc Extensions · Cluster Extensions** tab strip shown at the top of each page. Click **Azure Arc** in the sidebar to land on Registration; use the tabs to move between the sub-pages.

**Arc Resource Bridge** and **Custom Locations** are separate dedicated pages accessible via the indented sub-links below Azure Arc in the sidebar — see [Arc Resource Bridge](#17-arc-resource-bridge) and [Custom Locations](#18-custom-locations).

### Arc Registration (`/arc`)

**Access required:** HciRead + Azure sign-in for ARM data

Shows the cluster's Azure Arc registration status:

| Field | Description |
|---|---|
| Connection State | Connected / Disconnected / Expired |
| Region | Azure region the cluster is registered to |
| Resource Group | Azure resource group |
| Subscription | Azure subscription name/ID |
| Last Sync | When the cluster last synced with Azure |
| Billing Model | Trial / PayAsYouGo / GovernmentTrial etc. |

The **Azure Portal** link opens the cluster resource directly in the Azure Portal.

If ARM data isn't loading, check that the SPN credentials are configured in Admin → Settings and that the SPN has the required Azure RBAC assignments.

### Arc Machines (`/arc-machines`)

Lists all machines registered as `Microsoft.HybridCompute/machines` in the Azure resource group. Shows:

- Machine name, OS type, OS version
- Agent version, provisioning state, connectivity status
- Location, last status change

### Arc Extensions (`/arc-extensions`)

**Access required:** HciRead (view) · HciOperate (Upgrade Selected)

Shows extensions installed on each Arc-enabled machine (e.g. AzureMonitorWindowsAgent, MDE). Columns include:

- Machine name, extension name
- Publisher, type
- Installed version
- **Available** — version reported by ARM as available for upgrade, with an orange indicator when it differs from the installed version. Shows `—` when ARM does not report a newer version (auto-upgrade extensions are managed by the platform and typically show `—` here).
- Provisioning state (dot + label, e.g. `Succeeded`, `Updating`)
- **Health** — shows an error label (with tooltip containing the ARM message) only when ARM reports a problem. Normal/healthy extensions show `—`.
- **Upgrade Mode** — **Auto** (platform-managed) or **Manual** (highlighted in orange; must be upgraded explicitly)

#### Upgrading extensions

1. Tick one or more rows (or use the header checkbox to select all).
2. Click **⇡ Upgrade Selected (N)** in the toolbar.
3. Extensions on the same machine are batched into a single ARM call. ARM responds with `202 Accepted` immediately — the actual rollout takes several minutes per machine.
4. Refresh the page after a few minutes to see updated provisioning states.

> **Auto-upgrade extensions** are upgraded automatically by the platform whenever a new version is available. Manual-upgrade extensions stay at their installed version indefinitely unless you trigger an upgrade here.

#### Required ARM permissions (Arc machine extensions)

The SPN must have **one** of the following assigned on the resource group or subscription:

| Role | Notes |
|---|---|
| **Azure Connected Machine Resource Manager** | Purpose-built for Arc machine operations. Recommended for least-privilege SPNs. |
| **Contributor** | Broader write access — acceptable if granularity is not required. |

The specific ARM action required is:
```
Microsoft.HybridCompute/machines/upgradeExtensions/action
```
Reader alone is **not** sufficient for upgrades; it only permits the GET calls used to list extensions.

If an upgrade fails with `AuthorizationFailed`, the error message shown in the red banner will include the exact ARM error code and the specific action that was denied.

### Cluster Extensions (`/cluster-extensions`)

**Access required:** HciRead (view) · HciOperate (Upgrade Selected)

Shows extensions installed at the cluster level via `Microsoft.AzureStackHCI/clusters/arcSettings/extensions`. This covers cluster-wide Arc extensions like billing, monitoring, and diagnostics.

Columns include name, publisher, extension type, version, aggregate state (rolled-up across all nodes), upgrade mode, and per-node state summary.

#### Upgrading cluster extensions

1. Tick one or more extension rows.
2. Click **⇡ Upgrade Selected (N)**.
3. Each selected extension triggers one ARM POST call. ARM responds with `202 Accepted` — rollout is asynchronous and may take several minutes per node in the cluster.
4. Refresh after a few minutes to see updated aggregate state.

#### Required ARM permissions (cluster extensions)

| Role | Notes |
|---|---|
| **Azure Stack HCI Administrator** | Purpose-built for HCI cluster operations. Recommended for least-privilege SPNs. |
| **Contributor** | Broader write access — acceptable if granularity is not required. |

The specific ARM action required is:
```
Microsoft.AzureStackHCI/clusters/arcSettings/extensions/upgrade/action
```
Reader alone is **not** sufficient.

---

## 17. Arc Resource Bridge

**Route:** `/clusters/{name}/arc-bridge`  
**Access required:** HciRead  
**ARM required:** Yes — data is sourced from Azure Resource Manager

> **Custom role filtering:** If your role does not include View access to Arc Resource Bridge, this page shows an access-denied message and the sidebar link is hidden.

Shows the Arc Resource Bridge appliance (`Microsoft.ResourceBridge/appliances`) deployed as part of this Azure Local cluster, along with Azure Local Sites.

### Arc Resource Bridge Appliance

Each appliance is displayed as a card:

| Field | Description |
|---|---|
| Status | Current operational status (dot + text) |
| Provisioning State | ARM provisioning state (dot + text) |
| Version | Arc Resource Bridge software version |
| Distro Version | Appliance distribution version |
| Region | Azure region |
| Resource Group | Azure resource group |

The **🔗 Portal** link opens the appliance directly in the Azure Portal.

If no appliance is found, the page notes that the appliance is automatically deployed when the cluster is Arc-registered using the Azure Local portal or CLI.

### Azure Local Sites

If Azure Local sites are configured, they appear below the appliance section. Each site card shows:

| Field | Description |
|---|---|
| Provisioning State | ARM provisioning state (dot + text) |
| Address | Physical address summary |
| Contact | Site contact name and email address |
| Region | Azure region |

---

## 18. Custom Locations

**Route:** `/clusters/{name}/custom-locations`  
**Access required:** HciRead  
**ARM required:** Yes — data is sourced from Azure Resource Manager

> **Custom role filtering:** If your role does not include View access to Custom Locations, this page shows an access-denied message and the sidebar link is hidden.

Shows Azure Arc Custom Locations (`Microsoft.ExtendedLocation/customLocations`) associated with this cluster. Custom locations are created when the Arc Resource Bridge is deployed and configured, enabling Azure services such as Arc VMs and AKS to run on-premises.

Each custom location is displayed as a card:

| Field | Description |
|---|---|
| Provisioning State | ARM provisioning state (dot + text) |
| Host Cluster | The Arc-enabled Kubernetes cluster this location is associated with |
| Namespace | Kubernetes namespace within the host cluster |
| Cluster Extensions | Extensions enabled at this custom location |
| Region | Azure region |
| Resource Group | Azure resource group |

The **🔗 Portal** link opens the custom location resource in the Azure Portal.

If no custom locations are found, the page notes that they are created when the Arc Resource Bridge is deployed and linked to the cluster.

---

## 19. Agent Services

**Route:** `/clusters/{name}/services`  
**Access required:** HciRead (view) · HciOperate (Start/Stop/Restart)

Shows the status of the HCI infrastructure agent services on each cluster node:

- `wssdagent` — MOC (Management of Cloud) agent
- `mochostagent` — MOC host agent
- Any other Windows services matching the configured filter

### Columns

| Column | Description |
|---|---|
| Service Name | Windows service name |
| Display Name | Human-readable name |
| Status | Running / Stopped / StartPending etc. (with dot) |
| Start Type | Automatic / Manual / Disabled |
| Node | Cluster node |

### Actions

| Button | Effect |
|---|---|
| ▶ Start | Starts a stopped service |
| ⏹ Stop | Stops a running service |
| ↺ Restart | Restarts a running service |

Actions are recorded in the audit log.

---

## 20. Admin — Clusters

**Route:** `/admin/clusters`  
**Access required:** HciAdmin only

Manage the list of clusters registered in the application.

### Adding a cluster

Click **+ Add Cluster** and fill in:

| Field | Description |
|---|---|
| Name | Friendly display name used in the UI and URLs |
| Address | Hostname or IP address of the cluster (or a node for Hyper-V standalone) |
| Username | Service account username (leave blank to use gMSA / Kerberos pass-through) |
| Password | Service account password (stored encrypted) |
| Credential Source | `gMSA` (use app pool identity — works for both gMSA and standard service accounts) or `KeyVault` (fetch from Azure Key Vault) |
| Key Vault Secret Name | Secret name in Key Vault (only required when Credential Source = KeyVault) |

Changes take effect immediately — no restart required. The connection pool is evicted and reconnected with the new configuration on the next page access.

### Editing and deleting

Click **Edit** on any cluster row to update its configuration. Click **Delete** to remove it. All changes are written to the audit log.

### Disabling a cluster

If a cluster is temporarily offline or decommissioned but you do not want to delete its registration, click **Disable** on the cluster row. A disabled cluster:

- Is hidden from the home page cluster picker
- Is excluded from the background collector (no WinRM connections are attempted)
- Blocks any direct page access — a clear error is shown if a URL pointing to the cluster is visited
- Remains in the database so configuration is preserved

The **Status** column shows **✔ Enabled** (green) or **⛔ Disabled** (red) for each cluster. Click **Enable** to re-activate a disabled cluster immediately. All enable/disable actions are written to the audit log.

---

## 21. Admin — Audit Log

**Route:** `/admin/audit`  
**Access required:** HciAdmin only

An immutable record of every mutating action taken through the app (Start VM, Stop VM, Drain Node, Start Update, etc.). Read-only page loads are never logged.

### Columns

| Column | Description |
|---|---|
| Timestamp | UTC date/time of the action |
| User | UPN of the signed-in user |
| Cluster | Target cluster |
| Resource | Specific resource affected (VM name, node name, etc.) |
| Action | Verb (Start, Stop, Restart, AddCluster, etc.) |
| Outcome | Success / Failure |
| Duration | How long the operation took in milliseconds |
| Detail | Error message or additional context (failures only) |

### Filtering

Use the dropdowns and date pickers at the top to filter by:
- Action type (Start, Stop, PauseNode, etc.)
- Outcome (Success / Failure / All)
- Date range

### Export

Click **Export CSV** to download the filtered entries as a CSV file.

### Purge

Click **Purge entries older than...** to delete historical records. Purge actions are themselves logged so there is a permanent record that a purge occurred.

---

## 22. Admin — Settings

**Route:** `/admin/settings`  
**Access required:** HciAdmin only

Stores encrypted application-level configuration in the database. Values survive every deployment and take effect immediately without restarting the app pool (except where noted).

### ARM Authentication Mode

Controls how the Azure Arc pages authenticate to Azure Resource Manager.

| Mode | Description |
|---|---|
| **SPN** (Service Principal) | Uses dedicated service principal credentials. Requires SPN Tenant ID, Client ID, and Client Secret below. The SPN must have at least Reader RBAC on the target subscriptions. |

### Settings reference

| Setting | Group | Description |
|---|---|---|
| ARM Auth Mode | Azure ARM Authentication | `SPN` |
| SPN Tenant ID | Azure ARM Authentication (SPN mode) | The Entra tenant ID for SPN authentication |
| SPN Client ID | Azure ARM Authentication (SPN mode) | The service principal application (client) ID |
| SPN Client Secret | Azure ARM Authentication (SPN mode) | The service principal client secret (encrypted at rest) |
| Enable Action SPN | Azure ARM Authentication (Action SPN) | `false` (default) or `true` — enables the dual-SPN mode described below |
| Action SPN Tenant ID | Azure ARM Authentication (Action SPN) | Entra tenant ID for the action SPN (usually the same as the read SPN tenant) |
| Action SPN Client ID | Azure ARM Authentication (Action SPN) | Application (client) ID for the action SPN |
| Action SPN Client Secret | Azure ARM Authentication (Action SPN) | Client secret for the action SPN (encrypted at rest) |

### Action SPN (optional dual-SPN mode)

By default a single SPN is used for all ARM calls. When **Enable Action SPN** is set to `true` and the three Action SPN credentials are filled in, **all ARM write operations** (POST, PUT, PATCH, DELETE — any call that modifies Azure resources) are routed through the action SPN instead. The read SPN continues to handle all GET calls.

This allows a least-privilege split:

- **Read SPN** — Reader on the resource group only (no write permissions).
- **Action SPN** — a custom role with only the specific write actions needed for the operations you want to permit.

Current write operations the app performs and the ARM actions they require:

| Operation | Page | ARM action |
|---|---|---|
| Upgrade Arc machine extensions | Arc Extensions | `Microsoft.HybridCompute/machines/upgradeExtensions/action` |
| Upgrade cluster extensions | Cluster Extensions | `Microsoft.AzureStackHCI/clusters/arcSettings/extensions/write` |

Future write operations added to the app will also automatically use the action SPN — no configuration change is required.

Assign the action SPN a custom role containing only the actions for the operations you want it to perform, scoped to the resource group. No Reader or Contributor assignment is required on the action SPN.

---

### SPN permissions reference

The following table lists every ARM action the app may invoke, the page that triggers it, and the minimum Azure built-in role that grants it. All roles are assignable at subscription or resource group scope.

| Action / Operation | Page | Minimum role | ARM action |
|---|---|---|---|
| List clusters, read Arc registration | Arc Registration | **Reader** | `Microsoft.AzureStackHCI/clusters/read` |
| List Arc machines | Arc Machines | **Reader** | `Microsoft.HybridCompute/machines/read` |
| List Arc machine extensions | Arc Extensions | **Reader** | `Microsoft.HybridCompute/machines/extensions/read` |
| List cluster extensions | Cluster Extensions | **Reader** | `Microsoft.AzureStackHCI/clusters/arcSettings/extensions/read` |
| **Upgrade Arc machine extensions** | Arc Extensions | **Azure Connected Machine Resource Manager** | `Microsoft.HybridCompute/machines/upgradeExtensions/action` |
| **Upgrade cluster extensions** | Cluster Extensions | **Azure Stack HCI Administrator** | `Microsoft.AzureStackHCI/clusters/arcSettings/extensions/upgrade/action` |

> A SPN configured only for **read** operations needs the **Reader** role only.  
> A SPN that also needs to **trigger upgrades** needs one of the write roles above in addition to Reader (or can use **Contributor** which covers everything).

#### Recommended SPN setup for least privilege

```
Subscription or resource group scope:
  Reader
  Azure Connected Machine Resource Manager   ← for Arc machine extension upgrades
  Azure Stack HCI Administrator              ← for cluster extension upgrades
```

If your organisation separates read and write access, configure the optional **Action SPN** in `Admin → Settings → Azure ARM Authentication (Action SPN)` — see the FAQ for full details.

### Daily Digest

The daily digest sends a health summary of all registered clusters once per day via email and/or Teams. Configure it in the **Daily Digest** settings group.

| Setting key | Description |
|---|---|
| `Digest:Enabled` | Set to `true` to enable. Default: `false`. |
| `Digest:SendTimeUtc` | Time of day to send the digest in UTC 24-hour format, e.g. `07:00`. The scheduler checks every minute and fires once per calendar day when the current UTC minute matches. Default: `07:00`. |
| `Digest:EmailTo` | Email recipient(s) for the digest (comma-separated). Leave blank to use the global `Smtp:ToAddress`. Set to `none` to suppress email delivery of the digest while keeping the global address active. |
| `Digest:WebhookUrl` | Teams Incoming Webhook URL for the digest. Leave blank to use the global `Alerting:TeamsWebhookUrl`. Set to `none` to suppress Teams delivery while keeping the global webhook active. |

**What the digest contains:**
- Fleet-wide summary: total VMs running/off, nodes up/down, active health faults
- Per-cluster row with VM count, node state, health fault count, and last poll age
- A stale data warning for any cluster whose most recent snapshot is older than the stale threshold

> The digest uses the same delivery channels (SMTP and Teams webhook) as alert notifications. Ensure at least one channel is configured in Admin → Settings before enabling the digest.

---

## 23. Admin — Custom Roles (RBAC)

**Route:** `/admin/roles`  
**Access required:** HciAdmin only

Fine-grained role-based access control layered on top of the base Entra group policies. Allows you to restrict which resources specific Entra groups can see or operate — down to individual VM names or wildcard/regex patterns.

### Default roles

On first startup (when no custom roles exist), the app automatically creates four built-in roles as a starting point. You can rename, modify, or delete them at any time.

| Role | Description | Key permissions |
|---|---|---|
| **Cluster Admin** | Full administrative access across all resources | VMs: Start/Stop/ForceStop/Migrate; Nodes: Drain/Resume; Roles: Start/Stop/Move; Services: Start/Stop/Restart; Updates: Start; Clusters: View; all other types: View |
| **VM Admin** | Full VM lifecycle control, read-only everywhere else | VMs: Start/Stop/ForceStop/Migrate; Clusters: View; all other types: View |
| **Solution Admin** | Can start and monitor solution updates, read-only everywhere else | Updates: View+Start; Clusters: View; all other types: View |
| **Reader** | View-only access to all resources, no operational actions | All types: View only (including Clusters) |

Default roles have **no group assignments** when created — they are inactive until you assign an Entra group Object ID (or AD group SID for Windows Auth deployments) to them via the Group Assignments panel.

> The defaults are seeded by name — each role is added only if a role with that exact name does not already exist. Safe to run on every deployment; renaming a default role prevents it being re-created automatically.

> **Automatic backfill:** On every startup, the app checks whether existing default roles are missing any permission claims — for example, when a new resource type was added after your initial deployment. Any missing claims are automatically added with the correct default permissions. This only applies to the four named default roles; admin-created custom roles are never automatically modified.

### How it works

The app has three base access tiers (see [Access Levels](#24-access-levels)). Custom roles add a further layer:

- A **Custom Role** groups a set of permissions and the groups they apply to.
- Each role has **Permission claims** defining what resource types and specific resource names a member can access and what operations they can perform.
- Each role has **Group Assignments** linking it to one or more Entra security group Object IDs.
- Users inherit permissions from every role their groups are assigned to.

**Pass-through mode:** When no group assignments exist, custom RBAC is fully inactive and access is controlled only by the base Entra group policies. You can run the app indefinitely without ever configuring custom roles.

**HciAdmin bypass:** Admins always have unrestricted access regardless of any custom role definitions.

### Creating a role

1. Click **+ New Role**.
2. Enter a name (e.g. `Prod VM Operators`) and an optional description.
3. Click **Create**.

### Adding permissions to a role

Expand the role row. In the **Permissions** panel:

1. Choose the **Resource type** (VMs, Nodes, Storage, Network, **Clusters**, etc.).
2. Choose a **Name filter**:
   - **All resources** — the permission applies to every resource of this type.
   - **Wildcard** — use `*` (any characters) and `?` (single character). Example: `PROD-*` matches `PROD-VM01`, `PROD-SQL`.
   - **Regex** — full .NET regular expression, case-insensitive. Example: `^(PROD|DR)-` matches any name starting with `PROD-` or `DR-`.
3. Check the **operations** you want to permit:

| Operation | Effect |
|---|---|
| View | Resource appears in list views. **Also controls:** sidebar link visibility, page-level access, and Overview card visibility for VMs and Roles |
| Start | Start / power-on (includes Restart) |
| Stop | Graceful stop / suspend |
| ForceStop | Hard / forced power-off |
| Migrate | Live migrate (VMs) / Move (cluster roles) |
| Drain | Pause + drain (cluster nodes) |
| Resume | Resume / failback (cluster nodes) |
| Delete | Reserved — not currently wired to UI actions |

4. Click **+ Add Permission**.

> A user with `View` on `PROD-*` VMs but not `ALL` will only see VMs whose names match `PROD-*` — other VMs are completely hidden from their view.
>
> **Clusters resource type:** Adding a permission on `Clusters` with a name pattern controls which clusters the user sees in the home page picker and in the left navigation. For example, a pattern of `PROD-*` shows only clusters whose names start with `PROD-`. Users who navigate directly to a cluster URL they do not have View access to will see an access-denied message.
>
> **View also controls four other access points:**
> - **Cluster picker** — the home page only shows clusters for which the user has `Clusters` View access.
> - **Sidebar link** — the navigation link for that resource type is hidden when the user has no View permission for it.
> - **Page access** — navigating directly to the URL shows an access-denied message instead of data.
> - **Overview card** — the VM and Roles summary cards on the Overview page are hidden when the user lacks View access to those resource types.

### Assigning a group to a role

Expand the role row. In the **Group Assignments** panel:

1. Type in the **search box** to find Entra groups by display name. Results appear as a typeahead dropdown — click a result to select it. The group Object ID is filled in automatically.
2. If the Graph API search is unavailable (not configured or consent not granted), click the **Or paste Object ID manually** link and enter the group GUID directly.
   - Find it in Entra Portal → **Groups** → select the group → **Overview** → **Object ID**.
3. Click **+ Assign Group** to apply the assignment.

All users in that group now inherit the permissions defined on the role.

> **Graph API requirement:** The group search uses the Microsoft Graph API (`Group.Read.All` delegated permission). If the search returns no results, ensure the Entra app registration has the `Group.Read.All` delegated permission with **admin consent** granted, and that the signed-in admin account can see groups in your tenant. Object ID paste always works regardless of Graph configuration.

### Removing permissions / groups

Click the **✕** button on any permission row or assignment row to remove it immediately. Changes take effect within 5 minutes (cache TTL) or instantly if the app pool is recycled.

### Example: give the Helpdesk group view-only access to Prod VMs

1. Create a role: **Helpdesk ReadOnly**
2. Add permission: Resource = `Clusters`, Filter = `Wildcard: PROD-*`, Operations = `View` only
3. Add permission: Resource = `VMs`, Filter = `Wildcard: PROD-*`, Operations = `View` only
4. Assign group: paste the Object ID of your Helpdesk Entra group

Result: Helpdesk members only see clusters matching `PROD-*` in the picker and can only see VMs matching `PROD-*`, with no action buttons visible. They cannot navigate to clusters or resource types not covered by their permissions.

### Example: two teams, two cluster sets

| Role | Clusters pattern | Resource permissions | Entra group |
|---|---|---|---|
| **Team A — Operator** | `Wildcard: PROD-*` | All resource types: View; VMs: Start/Stop | `sg-team-a` OID |
| **Team B — Operator** | `Wildcard: DEV-*` | All resource types: View; VMs: Start/Stop | `sg-team-b` OID |

Team A members see only `PROD-*` clusters in the picker and nav; Team B sees only `DEV-*`. If a Team A member tries to navigate directly to a `DEV-*` cluster URL, they see an access-denied message.

---

## 24. Access Levels

The app enforces three base access tiers via Entra security group membership.

| Level | Description | What you can do |
|---|---|---|
| **HciRead** | Read-only operations | View all pages. No actions. |
| **HciOperate** | Operational access | All read access + VM Start/Stop/Restart/Suspend/Migrate, Node Pause/Resume, Role Start/Stop/Move, Start Solution Update, Service Start/Stop/Restart. |
| **HciAdmin** | Full administrative access | All of the above + access to Admin pages (Clusters, Audit Log, Settings, Custom Roles). |

These groups are configured in `appsettings.json` on the server. All three can point to the same Entra group if a single access tier is sufficient.

**Group membership takes effect at sign-in.** If you are added to a group mid-session, sign out and back in to receive the updated claims.

**Custom Roles layer on top** of these tiers — they can restrict but not expand access beyond what your base tier permits. An HciRead member with a custom role granting `Op.Start` on VMs still cannot start VMs, because `Op.Start` is an HciOperate-level action.

**What custom role View restrictions control:**

When a custom role limits which resource types you can see, the restriction is enforced at three levels simultaneously:

| Where | Behaviour |
|---|---|
| Sidebar navigation | Links for resource types you cannot View are hidden automatically. Section headers (e.g. *Compute*, *Operations*) are hidden when all links within them are hidden. |
| Page access | Navigating directly to a restricted page (e.g. by URL) shows an access-denied message rather than data. |
| Overview cards | The VM and Cluster Roles summary cards on the Overview page are hidden when you lack View access to those resource types. Items within visible cards are filtered by any name patterns in your role. |

> **HciAdmin users** are exempt from all custom role restrictions — they always see the full navigation and all data.

---

## 25. Frequently Asked Questions

**Q: I signed in but see "Access Denied" on every page.**  
A: Your account is not in any of the three configured Entra groups. Contact your administrator to be added to the HciRead group (or higher).

**Q: I can see VMs but the action buttons aren't visible.**  
A: Your account is in HciRead but not HciOperate. You need to be added to the HciOperate group by your administrator. Alternatively, a custom role with the required Op flags may need to be assigned to your group.

**Q: The VM list is empty but I know there are VMs on the cluster.**  
A: A custom RBAC role with a name pattern may be filtering your view. Contact your administrator to check `/admin/roles` for any roles whose pattern doesn't match the VMs you expect to see.

**Q: I started an action but the VM state didn't update.**  
A: The page automatically polls for 15 seconds after an action. If the state still hasn't changed after that, click **↻ Refresh** manually. If the action failed, a red banner message will explain why.

**Q: The network adapter flyout shows "No network adapters found".**  
A: This can happen if the VM is off and doesn't have any configured adapters, or if WinRM connectivity to the host node is unavailable. Check the debug log (if you have Admin access) for any PowerShell errors.

**Q: The Azure Arc page says "ARM not configured".**  
A: The ARM authentication mode needs to be set in [Admin → Settings](#22-admin--settings). Select **SPN** and provide the SPN Tenant ID, Client ID, and Client Secret.

**Q: Solution Update progress isn't streaming.**  
A: The live monitor requires an active cluster connection. Refresh the page. If the update run has already completed, a static summary of the completed plan is shown instead of a live stream.

**Q: How do I add a new cluster to the app?**

**Q: Can I use separate SPNs for read and write/action operations?**  
A: Yes — the *Action SPN* feature lets you configure a second, lower-privilege service principal used exclusively for **all ARM write operations** (POST, PUT, PATCH, DELETE). The read SPN continues to handle all GET calls. Any write operation added to the app in future releases will also automatically use the action SPN — no reconfiguration required.

To enable it, go to **Admin → Settings** → *Azure ARM Authentication (Action SPN)*:

1. Set **Enable Action SPN** to `true`.
2. Fill in **Action SPN Tenant ID**, **Client ID**, and **Client Secret**.
3. Assign the action SPN a custom Azure role on the resource group with only these two actions:
   - `Microsoft.HybridCompute/machines/upgradeExtensions/action`
   - `Microsoft.AzureStackHCI/clusters/arcSettings/extensions/write`

No Reader or Contributor assignment is needed for the action SPN — it only needs those two actions.
The read SPN still requires **Reader** on the resource group (plus **Azure Connected Machine Resource Manager** and **Azure Stack HCI Administrator** if it also performs upgrades in any scenario).

**Q: The upgrade request succeeded but the extension state hasn't changed after 10 minutes.**  
A: ARM extension upgrades are asynchronous. The `202 Accepted` response means the request was accepted, not that it has completed. Check the **Provisioning** and **State** columns — `Updating` or `Upgrading` indicates the rollout is in progress. If the state remains `Failed`, check the Azure Portal → the machine's Extensions blade for detailed error information.  
A: Go to [Admin → Clusters](#20-admin--clusters) (HciAdmin required) and click **+ Add Cluster**.

**Q: Custom role changes aren't taking effect immediately.**  
A: Role changes are cached for up to 5 minutes. For instant effect, ask an administrator to recycle the IIS app pool (`AZLManagementPool`) on the server.

**Q: Is there an option for users on domain-joined machines who can't use MFA?**  
A: Yes — a second Windows Authentication site can be deployed alongside the primary Entra ID site. Domain-joined browsers receive silent Kerberos SSO with no login prompt. See [Windows Authentication Deployment](#26-windows-authentication-deployment) for details.

**Q: I'm on the Windows Auth URL but I'm still getting a sign-in prompt.**  
A: Work through this checklist in order:
1. **HTTP SPNs missing** — most common cause. Run `setspn -L DOMAIN\azlmgmt-svc$` on the server; you should see `HTTP/azlmgmt-win.yourdomain.com` and `HTTP/azlmgmt-win`. If missing, run `Setup-IIS-WinAuth.ps1` again (step 5b now auto-registers them).
2. **Machine not domain-joined** — NTLM fallback will prompt once; enter `DOMAIN\username` + password.
3. **Browser not passing Kerberos automatically** — for Edge/IE add `azlmgmt-win.yourdomain.com` to the Local Intranet zone. For Chrome it inherits IE zone settings. For Firefox add the URL to `network.negotiate-auth.trusted-uris` in `about:config`.
4. **`useAppPoolCredentials` not set** — run `Deploy-ToIIS.ps1` which applies this setting in step 8 on every deploy.

**Q: My group was given a custom role but I still can't see anything.**  
A: Ensure the role has at least one permission with `View` checked. Without a `View` claim, the resource filter returns nothing. Also confirm the Entra group Object ID in the assignment exactly matches what Entra is issuing in your token groups claim.

**Q: A yellow banner says "Data may be outdated" on a page. What does that mean?**  
A: The **background collector** did not successfully sync data for this cluster within the expected time window. The data shown is from the last successful poll and may not reflect recent changes. Click **Refresh live data** in the banner (or **↻ Refresh** in the toolbar) to force an immediate live data fetch directly from the cluster over WinRM. The stale flag clears once the fresh fetch completes.

**Q: How often does the background collector update the data?**  
A: By default, VM state is refreshed approximately every 2 minutes, cluster nodes every 4 minutes, cluster info and roles every 10 minutes, storage pools/virtual disks every 20 minutes, and solution updates/physical disks every 40 minutes (all approximate; intervals are auto-scaled with cluster count). These intervals are configurable in **Admin → Settings** under *Background Collector*. In auto mode the interval scales up automatically when multiple clusters are registered so total WinRM load stays constant.

**Q: The background collector shows a red "Circuit open" status in Admin → Diagnostics. What should I do?**  
A: The circuit breaker trips after 3 consecutive poll failures to prevent the poller from hammering an unreachable cluster. Common causes: WinRM not reachable (firewall, cluster down), gMSA Kerberos ticket invalid, or credentials changed. Check network connectivity and WinRM access from the app server, then click the **Reset** button that appears next to the circuit-open row in **Admin → Diagnostics** to reset the circuit immediately without an IIS recycle.

---

## 26. Windows Authentication Deployment

The app supports a **second IIS site** running Windows Authentication (Kerberos/NTLM) alongside the primary Entra ID site. Both sites share the same database, same cluster registry, and the same app binaries.

| Site | URL | Authentication | Typical users |
|---|---|---|---|
| Primary | `https://azlmgmt.yourdomain.com` | Entra ID (SSO) | Cloud/external users, MFA required |
| Windows Auth | `http://azlmgmt-win.yourdomain.com` (HTTP by default) | Kerberos/NTLM | Domain-joined machines, no MFA prompt |

> **HTTPS on the Windows Auth site:** HTTP is the default. Run `Add-HttpsBinding.ps1 -SiteName AZLManagementWinAuth` once a certificate is available to enable HTTPS.

### How sign-in works

When you browse to the Windows Auth URL from a **domain-joined machine**, your browser automatically supplies your Kerberos ticket — no sign-in prompt appears. On a non-domain machine the browser will show an NTLM username/password prompt.

### Access control

The Windows Auth site uses **AD Security Group SIDs** (not Entra Object IDs) to define the three access levels:

| Level | AD group SID set in `appsettings.WinAuth.json` | Access |
|---|---|---|
| HciRead | `Groups:HciRead` SID | Read-only cluster data |
| HciOperate | `Groups:HciOperate` SID | Read + VM/node operations |
| HciAdmin | `Groups:HciAdmin` SID | Full access including admin pages |

To get an AD group's SID, run on any domain controller:
```powershell
(Get-ADGroup "YourGroupName").SID.Value
# Returns: S-1-5-21-...-xxxx
```

Then paste the SID into `C:\apps\azlmgmt\appsettings.WinAuth.json` on the app server.

### Fine-grained RBAC

The Windows Auth site runs with **RBAC pass-through** enabled (`Rbac:ForcePassThrough = true`). This means the custom roles and patterns configured in [Admin - Custom Roles](#23-admin--custom-roles-rbac) do **not** apply on the Windows Auth site. Access is controlled solely by the three flat group policies above.

This is intentional — the custom RBAC `RoleAssignments` table uses Entra group OIDs. If you need per-resource or pattern-based RBAC on the Windows Auth site, see the developer documentation (BACKLOG-9, Option B) for the upgrade path.

### Limitations on the Windows Auth site

| Feature | Availability |
|---|---|
| All WinRM-based pages (VMs, Nodes, Storage, Network, Events, etc.) | Fully functional |
| Arc Registration status | Functional (WinRM data only) |
| Azure Arc ARM enrichment (Arc Machines, Extensions, Cluster Extensions) | Not available — requires an Entra token |
| Custom RBAC roles and name-pattern filtering | Bypassed — flat group policies only |
| Audit log | Shared with Entra site — all actions logged normally |

Users who need the full Arc ARM data (extension upgrade buttons, Arc machine status) should use the primary Entra ID site.

### Setup (administrator)

The Windows Auth site is set up by running `Setup-IIS-WinAuth.ps1` on the app server **after** `Setup-IIS.ps1` and the first `Deploy-ToIIS.ps1` deploy have already been completed.

```powershell
# On the app server, run as Domain Admin
.\scripts\Setup-IIS-WinAuth.ps1
```

The script:
1. Creates the `AZLManagementWinPool` app pool (same gMSA identity as the Entra site)
2. Sets `ASPNETCORE_ENVIRONMENT = WinAuth` on the pool so the overlay config is loaded
3. Creates the `AZLManagementWinAuth` IIS site on **port 80** (HTTP) pointing at `C:\apps\azlmgmt`
4. Disables Anonymous Authentication and enables Windows Authentication on the site
5. Registers `HTTP/azlmgmt-win.yourdomain.com` and `HTTP/azlmgmt-win` SPNs on the gMSA (required for Kerberos)

After running the script:
1. Populate AD group SIDs in `C:\apps\azlmgmt\appsettings.WinAuth.json`  
   *(Leave all three values empty to allow any authenticated domain user — safest starting point)*
2. Verify `AllowedHosts` in `C:\apps\azlmgmt\appsettings.json` includes `azlmgmt-win.yourdomain.com` — both hostnames must be listed or the app returns HTTP 400
3. Add a DNS A record: `azlmgmt-win.yourdomain.com` → server IP
4. Run `scripts\Install.ps1` once to deploy the app binaries if not done already
   (or `Install.ps1 -Upgrade` if you already have a previous version installed)
5. Test from a domain-joined machine: `http://azlmgmt-win.yourdomain.com/`
6. *(Optional)* Run `Add-HttpsBinding.ps1 -SiteName AZLManagementWinAuth` for HTTPS

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ERR_CONNECTION_REFUSED` | Site bound to wrong port, or Default Web Site competing on port 80 | Check IIS bindings — site must be on port 80 (or 443 for HTTPS). Stop Default Web Site if needed. |
| Credential prompt loops — never accepts password | Missing HTTP SPNs on gMSA (`SEC_E_NO_CREDENTIALS`) | Run `setspn -S HTTP/azlmgmt-win.yourdomain.com DOMAIN\azlmgmt-svc$` and `setspn -S HTTP/azlmgmt-win DOMAIN\azlmgmt-svc$`, then recycle `AZLManagementWinPool` |
| `HTTP 400 Bad Request — Invalid Hostname` | `AllowedHosts` in `appsettings.json` does not include the WinAuth hostname | Add `azlmgmt-win.yourdomain.com` to `AllowedHosts` in `appsettings.json`; `Deploy-ToIIS.ps1` will keep it in sync |
| Prompt loop only on HTTPS (not HTTP) | `UseHttpsRedirection()` redirects HTTP→HTTPS; IIS re-challenges on the redirected HTTPS connection | Already fixed in code — ensure you are running the latest deployed version |
| `401.2 Unauthorized` with no prompt | `useAppPoolCredentials` not set — IIS tries the machine account key, not the gMSA | Run `Deploy-ToIIS.ps1` (step 8 re-applies this setting) or set manually in IIS Manager |

---

## 27. Admin — Diagnostics & Background Collector

**Route:** `/admin/debug-perf`  
**Access required:** HciAdmin only

The Diagnostics page has three panels.

---

### Background Collector Health

Shows the current state of the **background poller** — the in-process service that periodically connects to each registered cluster over WinRM/CIM and writes VM state, node health, cluster info, cluster roles, solution updates, storage pools, virtual disks, physical disks, and Arc registration data (9 data types) to the local SQLite database. Pages load instantly from this cache rather than waiting for a live WinRM call on every visit.

| Column | Description |
|---|---|
| Cluster | Cluster name |
| Status | Green = OK · Orange = recent failures · Red = circuit open |
| Last Success | Time of the last successful poll, with age (e.g. `14:32:01 (48s ago)`) |
| Failures | Consecutive failure count since the last success |
| Last Message | Error text when in a failure state; green *"Collected successfully (N ms, N calls)"* when the last poll was clean — clears automatically after the next successful poll |

**Status values:**

| Status | Meaning |
|---|---|
| 🟢 OK | Last poll succeeded; no consecutive failures |
| 🟠 Degraded (N fails) | Polls are partially failing; circuit still closed |
| 🔴 Circuit open until HH:mm:ss | Too many consecutive failures — poller is skipping this cluster until the retry time |
| ⚫ Never polled | App just started; first poll hasn't run yet (~15 s after startup) |

**Circuit breaker reset:** Click the **Reset** button next to the circuit-open row in the health table. This resets the poller retry state immediately and allows it to reconnect without an IIS recycle. Recycling `AZLManagementPool` in IIS Manager also resets the circuit but causes all in-flight requests to restart.

**Collector settings** are configurable in **Admin → Settings** under the *Background Collector* group:

| Setting | Default | Description |
|---|---|---|
| Enabled | `true` | Master on/off switch |
| Min Poll Interval (s) | `120` | Minimum time between polls for any data type |
| Auto Interval | `true` | Scales interval up with cluster count to keep WinRM load constant |
| Max Concurrent Polls | `3` | Maximum clusters polled simultaneously |
| VM Interval Multiplier | `1` | VMs polled every 1x min interval (~2 min) |
| Node Interval Multiplier | `2` | Nodes polled every 2x min interval (~4 min) |
| Info/Roles Multiplier | `5` | Cluster info and roles polled every 5x min interval (~10 min) |
| Updates Multiplier | `20` | Solution updates polled every 20x min interval (~40 min) |
| Storage Pool Multiplier | `10` | Storage pools polled every 10x min interval (~20 min; uses CIM) |
| Physical Disk Multiplier | `20` | Physical disks polled every 20x min interval (~40 min; uses CIM) |
| Arc Info Multiplier | `30` | Arc registration data polled every 30x min interval (~60 min) |
| History Retention (days) | `3` | How long snapshot history rows are kept before pruning |
| Circuit Breaker Threshold | `3` | Consecutive failures before the circuit opens |
| Circuit Breaker Timeout (s) | `300` | How long the circuit stays open before auto-retry |
---

### PS / WinRM Call Log

A rolling buffer of the last 500 PowerShell/WinRM calls made by the app — includes both background collector calls and live page calls. Useful for diagnosing slow pages, credential failures, or unexpected empty results.

| Column | Description |
|---|---|
| Time | Timestamp to millisecond precision |
| Cluster | Cluster the call was made against |
| Method | Service method called (green = success, red = had PS errors) |
| Duration | End-to-end time in ms — amber ≥ 2 s, orange ≥ 5 s |
| Rows | Result row count returned |
| Error | First PS error stream message, if any |

Rows with PS errors are highlighted with a dark red background. The log auto-refreshes every 5 seconds while the page is open. Click **Clear** to flush the buffer.

---

### Raw Get-ClusterPerf Diagnostic

Runs a raw `Get-ClusterPerf` PowerShell script against a selected cluster and dumps all returned series names, values, and unit enums. Used by developers and administrators to diagnose missing or unexpected performance metrics on the Storage QoS / VM Performance pages.

Select a cluster and target type (ClusterNode / Volume / VM), then click **▶ Run Diagnostic**.

---

### Database schema migration (existing installations)

If you are upgrading from a version that did not include the background collector, the three collector tables must be added to the existing `app.db` manually — `EnsureCreated` only runs against an empty database and will not modify an existing file.

**Step 1 — Install sqlite3.exe on the IIS server**

Download from **https://www.sqlite.org/download.html** → *Precompiled Binaries for Windows* → `sqlite-tools-win-x64-*.zip`. Extract and copy `sqlite3.exe` to `C:\Windows\System32` (or anywhere on the server PATH).

```powershell
# Alternative: winget
winget install SQLite.SQLite

# Alternative: Chocolatey
choco install sqlite
```

**Step 2 — Run the migration SQL**

```powershell
$db = "C:\apps\azlmgmt-data\app.db"
sqlite3.exe $db @"
CREATE TABLE IF NOT EXISTS LatestSnapshots (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    ClusterName TEXT NOT NULL,
    DataType TEXT NOT NULL,
    JsonData TEXT NOT NULL,
    CollectedAt TEXT NOT NULL,
    CollectDurationMs INTEGER NOT NULL DEFAULT 0,
    UNIQUE(ClusterName, DataType)
);
CREATE TABLE IF NOT EXISTS SnapshotHistories (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    ClusterName TEXT NOT NULL,
    DataType TEXT NOT NULL,
    JsonData TEXT NOT NULL,
    CollectedAt TEXT NOT NULL,
    CollectDurationMs INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX IF NOT EXISTS IX_SnapshotHistories_Cluster
    ON SnapshotHistories (ClusterName, DataType, CollectedAt);
CREATE TABLE IF NOT EXISTS CollectorHealths (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    ClusterName TEXT NOT NULL,
    LastSuccessAt TEXT,
    LastFailureAt TEXT,
    LastError TEXT,
    LastPollDurationMs INTEGER NOT NULL DEFAULT 0,
    LastPollCallCount INTEGER NOT NULL DEFAULT 0,
    ConsecutiveFails INTEGER NOT NULL DEFAULT 0,
    IsCircuitOpen INTEGER NOT NULL DEFAULT 0,
    CircuitOpenUntil TEXT,
    UNIQUE(ClusterName)
);
"@
```

**Step 2b — Add IsDisabled column to Clusters table**

If you are upgrading from a version before cluster disable was added, add the new column. The command is safe to run on any install — it is silently ignored if the column already exists.

```powershell
sqlite3.exe $db "ALTER TABLE Clusters ADD COLUMN IsDisabled INTEGER NOT NULL DEFAULT 0;" 2>&1 | Out-Null
```

**Step 3 — Verify**

```powershell
sqlite3.exe $db ".tables"
# Output should include: CollectorHealths  LatestSnapshots  SnapshotHistories

sqlite3.exe $db ".indexes"
# Output should include: IX_SnapshotHistories_Cluster
```

**Step 4 — Recycle the app pool**

Recycle `AZLManagementPool` in IIS Manager, or:

```powershell
Invoke-Command -ComputerName azlmgmt.yourdomain.com -ScriptBlock {
    Import-Module WebAdministration
    Restart-WebAppPool AZLManagementPool
}
```

The background collector starts ~15 seconds after startup. Confirm data is flowing after 30 seconds:

```powershell
sqlite3.exe $db "SELECT ClusterName, DataType, CollectedAt FROM LatestSnapshots;"
sqlite3.exe $db "SELECT ClusterName, ConsecutiveFails, IsCircuitOpen FROM CollectorHealths;"
```

---

*For deployment and infrastructure documentation, see [IIS-Deployment.md](IIS-Deployment.md).*  
*For developer and architecture documentation, see [../.github/copilot-instructions.md](../.github/copilot-instructions.md).*

> **Maintainers:** when adding a new page or feature, update both this file (`docs/USER-GUIDE.md`) and `.github/copilot-instructions.md` as part of the same commit. The new-page checklist in `copilot-instructions.md` lists both as required steps.

---

## 28. Fleet Status Board

**Route:** `/status`  
**Access required:** HciRead (or higher)

The Fleet Status Board provides a **zero-WinRM, read-only overview of all registered clusters** in a single page. All data comes from the background collector's database snapshots — no live WinRM connections are made when the page loads.

---

### Summary Pills

A row of coloured pills at the top of the page shows fleet-wide totals:

| Pill | What it counts |
|---|---|
| VMs Running | Total VMs in Running state across all clusters |
| VMs Off | Total VMs in Off state |
| Nodes Up | Total cluster nodes reported Up |
| Health Faults | Total active health faults (red if > 0) |
| Updates Available | Total clusters with at least one pending solution update |
| Clusters | Total registered clusters included in the view |

---

### Fleet Table

One row per cluster. Columns:

| Column | Description |
|---|---|
| Cluster | Cluster name, hyperlinked to its `/clusters/{name}/overview` page |
| VMs | Running / Off / Paused counts |
| Nodes | Nodes Up / Total (coloured dot: green = all up, orange = some down, red = majority down) |
| Health | Fault summary: Critical / Warning count with traffic-light dot; green if no faults |
| Updates | Failed / In-progress / Available count / Current (all-green) |
| Collector | Status dot from the background collector health record |
| Last Poll | Time since the last successful collector run (full timestamp on hover tooltip) |

Click a cluster name to navigate directly to its Overview page.

---

### Snapshot Freshness Grid

A collapsible section (collapsed by default) showing a grid of **9 data types × all clusters**. Each cell shows a coloured dot and age label for that cluster's most recent snapshot of that type.

| Dot colour | Meaning |
|---|---|
| 🟢 Green | Fresh — age is less than 50% of the stale threshold |
| 🟠 Orange | Ageing — age is between 50% and 100% of the threshold |
| 🔴 Red | Stale — age exceeds the stale threshold |
| ⚫ Grey | No snapshot exists yet |

**Stale thresholds per data type:**

| Type | Threshold |
|---|---|
| VMs | 10 minutes |
| Nodes | 15 minutes |
| Cluster Info | 30 minutes |
| Cluster Roles | 30 minutes |
| Storage Pools | 1 hour |
| Virtual Disks | 1 hour |
| Physical Disks | 2 hours |
| Solution Updates | 2 hours |
| Arc Registration | 4 hours |

Click **▼ Snapshot Freshness** / **▶ Snapshot Freshness** to expand or collapse the grid.

---

### Use Cases

- **Morning health check:** Open the Fleet Status Board before starting work to see at a glance if any cluster has faults, stale data, or collector problems.
- **Collector monitoring:** The Collector column and freshness grid immediately show which clusters have polling problems without navigating to each cluster individually.
- **Pre-maintenance scan:** Confirm all nodes are Up and no updates are failing before scheduling a maintenance window.
- **Alternative to individual cluster browsing:** For read-only stakeholders who need a high-level view, the Fleet Status Board may be sufficient without ever visiting a per-cluster page.

> **Note:** The Fleet Status Board only reflects data already collected by the background collector. If a cluster was added recently and has not been polled yet, it will show no data in the table until the first successful poll completes.

---

## 28a. Fleet Schedules

**Route:** `/schedules`  
**Access required:** HciRead (view) · `Updates → Start` permission (create/cancel schedules)

The Fleet Schedules page provides a single place to view, create, and cancel solution update schedules across all registered clusters.

> Fleet Schedules is accessible from the left sidebar (above Fleet Status, visible on all pages — not cluster-scoped).

### Pending Schedules

Lists all update schedules that have not yet been triggered or cancelled. Schedules can be:

- **Single-cluster** — one update entry for one cluster.
- **Batch** — a set of schedules created together in one operation (e.g. rolling a cumulative update across multiple clusters). Batch rows are grouped with a collapse/expand toggle showing the number of clusters. Click the row to expand individual cluster entries.

#### Columns

| Column | Description |
|---|---|
| Update | Display name of the update package |
| Version | Update version string |
| Cluster(s) | Cluster name (single) or comma-separated list (batch header) |
| Scheduled Start | UTC start time with a status dot: green = future, orange = due within 1 hour, grey = past/overdue |
| Live State | Real-time update run state sourced from the latest cluster snapshot (e.g. Running, Succeeded) |
| Created By | UPN of the user who created the schedule |
| Notes | Free-text notes supplied when the schedule was created |
| Action | **Cancel** (single) or **Cancel All** (batch header) — requires `Updates → Start` permission |

> The scheduler triggers the update at the scheduled start time (checked every minute). If the app server is offline during the trigger window, the schedule is triggered on the next check-in.

### Schedule History

Shows the 50 most recent completed schedule entries. Columns match the Pending table with the addition of:

- **Triggered / Cancelled** — timestamp when the schedule was acted on
- **Schedule Status** — Pending / Triggered / Cancelled (with dot)

### Creating a Schedule

Click **New Schedule** (requires `Updates → Start` permission). A modal opens:

1. **Select cluster(s)** — check one or more clusters (multiple selections create a batch).
2. **Select update** — pick from the available updates shown for the selected cluster(s).
3. **Scheduled start (UTC)** — date-time picker; defaults to 30 minutes from now.
4. **Notes** — optional free-text note.

Click **Create** to save. For batches, the scheduler creates one entry per cluster under a shared batch ID.

### Per-cluster update scheduling

Individual update schedules can also be created from the [Solution Updates](#15-solution-updates) page for a specific cluster using the **Schedules** tab.

---

## 29. Admin — Alerts

**Route:** `/admin/alerts`  
**Access required:** HciAdmin (view and manage rules) · `Op.Acknowledge` to acknowledge fired alerts

The Alerts page has two sections: **Alert Rules** (top) and **Alert History** (bottom).

---

### Alert Rules

Lists every configured alert rule. Click **+ New Rule** to create one, or the **Edit** / **Delete** buttons on any row to modify or remove an existing rule.

#### Rule fields

| Field | Description |
|---|---|
| **Name** | Display name shown in the history table and in email/Teams notifications |
| **Cluster** | Cluster this rule applies to, or `*` to apply to all clusters |
| **Rule Type** | One of the four types — see below |
| **Severity** | `Critical` (red) or `Warning` (orange) — reflected in notification colour |
| **Threshold** | Minimum fault count before firing (ClusterHealthFault only; ignored for other types) |
| **Cooldown (mins)** | Minimum minutes between repeated notifications for the same rule and cluster. Prevents alert floods. **Acknowledging an alert resets the cooldown** — the next firing sends a fresh notification. |
| **Webhook URL** | Optional — Teams incoming webhook URL for this rule only. Overrides the global webhook URL in Settings. Leave blank to use the global URL. |
| **Email To** | Optional — email recipient(s) for this rule only (comma-separated). Overrides the global `Smtp:ToAddress` in Settings. Leave blank to use the global address. |
| **VM Name Pattern** | **VMUnexpectedStop only** — glob filter on VM names. Supports `*` (any sequence) and `?` (single character), case-insensitive. Leave blank to alert on all VMs. Examples: `PROD-*`, `*-SQL-*`. Shown as a hint in the Cluster column when set. |
| **Enabled** | Toggle a rule on/off without deleting it |

#### Rule types

| Type | What triggers it |
|---|---|
| **Health Fault** | One or more active health faults on the cluster (from `Get-HealthFault`). Threshold controls how many faults are required before firing. |
| **Node Offline** | Any cluster node moves to a state other than `Up`. Fires independently for each affected node. |
| **Cluster Unreachable** | The background poller's circuit breaker trips — the cluster is unreachable for consecutive poll cycles. |
| **VM Stop** | One or more VMs transition to the `Off` state unexpectedly (not triggered by a user action through the app). Use the `VmNamePattern` field to limit which VMs trigger the rule. |

> **Health Fault threshold:** Setting threshold to `1` (default) fires on any fault. Setting it to `3` requires at least 3 simultaneous faults. Use a threshold of `1` for production clusters where any fault is actionable.

#### Channels — how alerts are delivered

Each rule fires through **both** channels simultaneously if both are configured. If a specific rule overrides a channel setting, the override takes precedence over the global setting.

| Channel | Global setting location | Per-rule override field |
|---|---|---|
| Teams webhook | Admin → Settings → `Alerting:TeamsWebhookUrl` | Rule **Webhook URL** field |
| Email (SMTP) | Admin → Settings → `Smtp:ToAddress` | Rule **Email To** field |

If neither a per-rule override nor the global setting is configured, that channel is silently skipped for the rule — no error is shown. Configure at least one channel in Admin → Settings before creating rules.

---

### Alert History

Shows the last 200 fired alert events. Use the filter dropdowns to narrow by cluster, rule type, or outcome.

#### History columns

| Column | Description |
|---|---|
| Time | UTC timestamp when the alert fired |
| Cluster | Cluster that triggered the alert |
| Rule | Rule name |
| Type | Rule type badge |
| Severity | Critical / Warning |
| Message | Short description of what triggered the alert |
| Outcome | `Sent` — at least one channel delivered successfully · `Failed` — all channels failed · `Suppressed` — within cooldown window |
| Ack | Acknowledgement status — see below |

#### Filtering

Three dropdowns in the toolbar filter by Cluster, Rule Type, and Outcome. Click **Refresh** to re-query.

---

### Acknowledging Alerts

Users with the `Op.Acknowledge` permission on the `AlertRules` resource type can acknowledge fired alerts directly from the history table.

**To acknowledge an alert:**

1. Locate the entry in the Alert History table. Unacknowledged `Sent` entries show an **Ack** button in the Ack column.
2. Click **Ack** — a modal appears with an optional free-text note field.
3. Enter a note (e.g. `"Investigated — storage rebalancing, expected"`) and click **Acknowledge**.

Acknowledged entries show a green check mark in the Ack column. Hover over the check to read the note.

**Effect on cooldown:** Acknowledging an alert resets the cooldown timer for that rule on that cluster. The next occurrence of the same alert condition will trigger a fresh notification immediately, rather than being suppressed by the previous event's cooldown. This is intentional — it prevents an operator from dismissing an alert and then missing a recurrence.

#### Who can acknowledge

By default, the `Cluster Admin` role includes `Op.Acknowledge` on `AlertRules`. To grant acknowledgement to other roles, add the `Acknowledge` operation to the `AlertRules` resource type in Admin → Custom Roles.

---

### Maintenance Windows

> **Note:** Maintenance windows suppress alert notifications without requiring individual alert rules to be disabled.

Configured in **Admin → Maintenance** (accessible from the Admin menu). During a maintenance window:

- The background poller continues collecting data normally.
- `AlertEngine` evaluates rules normally.
- Notification delivery (Teams webhook + email) is suppressed for any cluster inside its maintenance window.
- Alert history entries are still written with outcome `Suppressed (maintenance)`.

This prevents alert floods during planned maintenance (node firmware updates, storage rebalancing, update runs) without needing to disable and re-enable rules.

**Actions available on each row:**

| Button | When shown | Effect |
|---|---|---|
| **Deactivate** | Active/running windows and scheduled-future windows only | Immediately marks the window inactive — no further notifications are suppressed |
| **Edit** | All active windows (running or scheduled) | Opens a modal pre-populated with the current cluster, start/end times, and reason — save to apply changes |
| **Delete** | Always | Permanently removes the row from the database |

---

### RBAC for Alerts

| Permission | What it controls |
|---|---|
| `AlertRules → View` | Can see the Alerts page, rules list, and history |
| `AlertRules → Configure` | Can create, edit, and delete alert rules |
| `AlertRules → Acknowledge` | Can acknowledge fired alert history entries |

HciAdmin users always have all three permissions. Non-admin users need a custom role with the appropriate operations assigned on the `AlertRules` resource type.
