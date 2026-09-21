# High-Level Release Notes - Azure Local Cluster Tool (Web)

This file is the source for published GitHub Release bodies.
Each release body is generated from exactly one matching section (latest-only, non-cumulative).

## v0.12.15 - 2026-09-21

### Highlights
- Saved views (All Clusters / My Favourites / personal / shared) from the All Clusters dashboard are now also available on the Fleet Update Status page, and are shared between the two.

### Reliability
- Fixed SBE prerequisite detection to match the real ARM data — updates now correctly show every acceptable SBE baseline version instead of relying on a field ARM doesn't actually return.

## v0.12.14 - 2026-09-20

### Highlights
- Fleet Update Status search box now supports wildcards and multi-cluster search, e.g. `AZL-*` or `AZ-NUC-CL01, AZL-NUC-CL02`.
- The Solution Version filter now covers every installed version across the fleet instead of just the top 5.
- Both fleet tables are now paginated (25/50/100 rows) so the page stays responsive on fleets of 100-300+ clusters.
- Actively-installing clusters now show the exact ARM run step and elapsed time in the fleet view, matching the per-cluster Solution Updates page.

### Reliability
- Failed update runs now show the real ARM error message instead of a generic "Failed" label.
- Fixed `NotifyMessage` never being populated from ARM due to an incorrect nested-property assumption.
- ARM/ARG call volume is now capped and targeted to reduce throttling risk on large fleets.

## v0.12.13 - 2026-09-18

### Reliability
- Fixed misleading Solution Update diagnostics by separating the read-only ID lookup from the actual start command and removing an early pipeline-stop trigger.
- All update-start returns are now verified against cluster state and reported as Accepted, Rejected, or Indeterminate instead of a clean PowerShell return being assumed successful.
- Added correlation IDs and duplicate-submission guards for manual and scheduled update actions.

## v0.12.12 - 2026-09-17

### Highlights
- Redesigned the fleet Update Status page to show Feature Update and Cumulative Update side by side per cluster, instead of hiding one behind a collapsed history row.
- Added a live SBE Version column and an SBE Family column to both the fleet and per-cluster update views, replacing a misleading OEM Version column.
- Updates that are actually ready to install now stand out with a distinct orange "READY TO INSTALL" badge, instead of sharing the same color as already-installed updates.
- Updates that require a minimum SBE version first now show a clear warning badge explaining the requirement.

### Reliability
- Fixed SBE Version showing as blank in cases where the value was already available elsewhere in the update data.
- Cleaned up the expandable update history into a proper table for easier scanning.

## v0.12.11 - 2026-09-17

### Highlights
- Added a "Force Poll Now" admin action to immediately refresh a cluster's background data during an outage or right after fixing connectivity, instead of waiting for the next scheduled poll.

### Reliability
- Fixed the fleet Update Status page showing "No snapshot yet" for clusters that had actually been checked but had nothing new to report.
- Fixed a case where a transient background polling failure could show stale or empty update data on the fleet Update Status page even though the per-cluster page was correct; the fleet page now prefers a live, direct read for this data.

## v0.12.10 - 2026-09-17

### Highlights
- Improved Admin cluster onboarding UX by replacing raw save exceptions with a clear duplicate-name message.
- Standardized duplicate cluster behavior so API and UI return consistent conflict outcomes.

### Reliability and Upgrade Safety
- Added startup schema compatibility guards for legacy `Clusters` columns (`AksApiEndpoint`, `AksApiToken`) in drifted customer databases.
- Prevented insert failures in upgraded environments where legacy AKS columns remain `NOT NULL` without defaults.

### Support and Operations
- Added explicit support runbook guidance for schema drift verification and remediation in IIS deployment documentation.

## v0.12.9 - 2026-07-14

### Highlights
- AKS workload operations were expanded with scale and rollback, plus inline per-row actions.
- A workload browse-and-select picker was added to reduce manual entry errors during AKS actions.
- AKS pages received performance and security-focused refinements for better responsiveness.
- Fleet-level AKS visibility and action flows were refined for more consistent cross-cluster operations.

### Platform and Access
- RBAC handling for AKS and related action surfaces was tightened to align view and operate behavior.
- GitOps and AKS action paths were improved for clearer operator control and safer execution.

### Internal and Operational
- Instruction and optimization guidance files were refreshed to match current implementation patterns.

## v0.12.8 - 2026-06-01

### Highlights
- Added AKS cross-cluster workload restart actions from a global operations page.
- Added WinAuth support for multiple HciAccess group SIDs.
- Added a WinAuth cross-domain role-assignment policy gate to reduce false 403 outcomes.
