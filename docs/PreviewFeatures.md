# Azure Local Cluster Tool — Web: Preview Features

> **Preview features** are functional but may change before becoming generally available. They are disabled by default and must be explicitly enabled by an administrator. Test in a non-production environment before using against a production cluster.

---

## Contents

1. [Disk Replacement Wizard](#1-disk-replacement-wizard)
   - [1a. Enabling the Feature](#1a-enabling-the-feature)
   - [1b. Required Permissions](#1b-required-permissions)
   - [1c. Opening the Wizard](#1c-opening-the-wizard)
   - [1d. Wizard Steps](#1d-wizard-steps)
   - [1e. Resuming a Partially-Complete Replacement](#1e-resuming-a-partially-complete-replacement)
   - [1f. Audit Logging](#1f-audit-logging)
   - [1g. Known Limitations](#1g-known-limitations)
---

## 1. Disk Replacement Wizard

The Disk Replacement Wizard provides a guided, step-by-step process for safely retiring and removing a failed or degraded physical disk from an S2D (Storage Spaces Direct) storage pool. It enforces the correct sequence — retire first, wait for S2D to rebuild data, then physically remove — to minimise the risk of data loss.

### How S2D disk replacement works

Removing a disk from an S2D pool without preparation can result in data loss if the pool does not have enough redundancy to survive the sudden loss. The safe procedure is:

1. **Retire** the disk — this signals S2D to treat the disk as gone and begin copying its data ("repair jobs") to the remaining disks in the pool.
2. **Wait** for all repair jobs to complete — S2D rebuilds the data stripes across the remaining disks. This can take minutes to hours depending on data volume and disk speed.
3. **Remove** the disk from the pool — once redundancy is restored, the disk can be cleanly removed from the storage pool.
4. **Replace** the physical disk — the new drive is detected automatically by S2D and added as a hot spare or used to expand the pool.

The wizard enforces this sequence and monitors repair progress automatically.

---

### 1a. Enabling the Feature

The wizard is disabled by default. An HciAdmin must enable it via **Admin > Settings**:

1. Navigate to **Admin** > **Settings**.
2. Locate the **Preview Features** section.
3. Find **Disk Replacement Wizard (Preview)** and set it to **true**.
4. Click **Save**.

The button appears immediately on the Physical Disks tab of the **Storage** page for all users who have the required permission (see [1b. Required Permissions](#1b-required-permissions)).

To disable the wizard again, set the value back to **false** and click **Save**.

---

### 1b. Required Permissions

| Requirement | Detail |
|---|---|
| Page access | **HciRead** group (or higher) — standard Storage page access |
| Wizard button visible | Feature flag must be **true** |
| Wizard usable | User must have **Storage / Configure** permission in their custom RBAC role |

The wizard button is only rendered when both the feature flag is enabled **and** the signed-in user holds `Configure` on the `Storage` resource type. Users with View-only storage access will not see the button even when the feature is enabled.

If you do not have a custom RBAC role configured (pass-through mode), all authenticated users in the **HciOperate** or **HciAdmin** group can use the wizard once it is enabled.

---

### 1c. Opening the Wizard

1. Navigate to **Storage** for the target cluster.
2. The **Physical Disks** tab is shown by default.
3. Locate the disk you want to replace in the table.
4. Click the **Replace** button in the **Actions** column for that disk.

The wizard opens as a floating dialog over the current page. The title bar shows the disk's friendly name and serial number (if available).

> If the disk is already in **Retired** usage state when you open the wizard (e.g. it was retired previously), the wizard skips the Pre-flight and Retire steps and opens directly at **Step 2 — Repair** so you can continue monitoring rebuild progress.

---

### 1d. Wizard Steps

The wizard breadcrumb at the top of the dialog shows your current position:

**Pre-flight &rsaquo; 1 - Retire &rsaquo; 2 - Repair &rsaquo; 3 - Remove &rsaquo; Done**

---

#### Pre-flight

Runs a set of safety checks against the cluster before allowing the retirement to proceed.

| Check | Pass condition | Block condition |
|---|---|---|
| Pool health | `Healthy` | Any other state — retiring a disk while the pool is degraded can cause data loss |
| Active repair jobs | 0 jobs running | 1 or more — another rebuild is already in progress; wait for it to finish |
| Other retired disks | Informational only (shown as a warning count, not a blocker) | — |

Click **Check** to run the checks. Results appear as a table with colour-coded status dots.

- If a blocker is found, the reason is displayed in red. Fix the underlying issue and click **Re-check**.
- If all checks pass, click **Retire Disk** to proceed, or **Re-check** to re-run before proceeding.
- Click **Cancel** to close the wizard without making any changes.

---

#### Step 1 — Retire

The wizard calls `Set-PhysicalDisk -FriendlyName <name> -Usage Retired` via the cluster's WinRM connection. A spinner is shown while this runs.

- If the command succeeds, the wizard advances automatically to Step 2 and begins polling for repair jobs.
- If the command fails, the error message is shown and the wizard returns to Pre-flight for review.

This step is logged to the **Audit Log** as action `RetireDisk`.

---

#### Step 2 — Repair

Displays all active `MSFT_StorageJob` instances from the cluster's storage namespace. The list auto-refreshes every **10 seconds**.

| Column | Description |
|---|---|
| Job | S2D internal job name |
| State | Current job state (Running, Completed, Failed, Suspended, etc.) |
| Progress | Visual progress bar + percentage |
| Elapsed | Time since the job started (e.g. `3m 42s` or `1h 12m`) |

**Proceeding to removal:**

- The **Proceed to Remove** button is enabled only when all jobs are in Completed state (or no jobs exist).
- If one or more jobs are in **Failed** or **Suspended** state, a warning banner appears with a checkbox: _"I understand the risk and want to remove the disk anyway"_. Checking this box unlocks the Proceed button. Use this only if you have verified in the cluster that the pool has sufficient redundancy despite the stalled job.

Click **Refresh Now** to poll immediately without waiting for the 10-second timer.

> The polling timer runs only while Step 2 is active. It stops automatically when you proceed to Step 3 or close the wizard.

---

#### Step 3 — Remove

A confirmation step before the destructive remove operation.

- Displays the disk name and serial number for final verification.
- The **×** close button is hidden on this step to prevent accidental dismissal during the remove operation.
- Click **Confirm Remove** to call `Remove-PhysicalDisk` via PowerShell against the cluster.
- Click **Back** to return to Step 2 if you need to review the repair job state again.

The PowerShell script used:

```powershell
$disk = Get-PhysicalDisk -FriendlyName '<name>'
$pool = Get-StoragePool |
        Where-Object { $_.IsPrimordial -eq $false } |
        Get-PhysicalDisk |
        Where-Object { $_.UniqueId -eq $disk.UniqueId } |
        ForEach-Object { Get-StoragePool -PhysicalDisk $_ } |
        Select-Object -First 1
Remove-PhysicalDisk -StoragePool $pool -PhysicalDisk $disk -Confirm:$false
```

This step is logged to the **Audit Log** as action `RemoveDisk`.

---

#### Done

Confirms the disk has been removed from the storage pool.

- The disk will no longer appear in the Physical Disks table after the page refreshes.
- You can now **physically remove** the disk from the server. The replacement disk will be detected automatically by S2D and added to the pool as a hot spare or used to expand capacity.
- Click **Close & Refresh** to close the wizard and reload the Physical Disks table.

---

### 1e. Resuming a Partially-Complete Replacement

If you close the wizard partway through — for example, after retiring a disk but before the repair jobs have finished — you can resume by clicking **Replace** on the same disk again. The wizard detects that the disk's usage state is `Retired` and opens directly at **Step 2 — Repair**, skipping the Pre-flight and Retire steps.

---

### 1f. Audit Logging

Both mutating operations are written to the audit log (**Admin > Audit Log**):

| Action | Triggered by | Resource type |
|---|---|---|
| `RetireDisk` | Clicking **Retire Disk** in Pre-flight | `PhysicalDisk` |
| `RemoveDisk` | Clicking **Confirm Remove** in Step 3 | `PhysicalDisk` |

Each entry records the signed-in user (UPN, display name, object ID), the cluster name, the disk's friendly name, the outcome (Success / Failure), the error detail on failure, and the duration in milliseconds.

You can filter the Audit Log by action using the **Action** dropdown — `RetireDisk` and `RemoveDisk` are listed there.

---

### 1g. Known Limitations

| Limitation | Detail |
|---|---|
| Single disk at a time | The wizard handles one disk per session. Close and re-open to replace a second disk. |
| No multi-pool support | If the cluster has more than one non-primordial storage pool, the remove step targets the first pool that contains the disk. Clusters with a single S2D pool (the standard configuration) are not affected. |
| Repair job visibility | The job list shows all `MSFT_StorageJob` instances from the cluster — not just those created by retiring this specific disk. On busy clusters with concurrent rebuild activity, unrelated jobs may appear. |
| No cancel after Step 1 | Once a disk is retired (Step 1), the **Cancel** / **×** button is no longer shown during the Remove step. This is intentional — a retired disk should proceed through to removal rather than being left in an indeterminate state. You can still close the wizard during Step 2 (Repair); the disk will remain in Retired state and the wizard will resume from Step 2 on next open. |
| Requires WinRM | The `Set-PhysicalDisk` and `Remove-PhysicalDisk` commands run over WinRM to the cluster. Ensure WinRM connectivity is working correctly before using the wizard. |

---

*This document covers preview features that are subject to change. For generally available feature documentation, see [USER-GUIDE.md](USER-GUIDE.md).*
