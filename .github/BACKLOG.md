# Backlog — Internal Development Notes

This file supplements the public [ROADMAP.md](../ROADMAP.md) with implementation detail, investigation logs, and context that is too verbose for a public-facing roadmap entry. It lives in `.github/` so it is available in both the public releases repo and the private dev repo.

Items are numbered (BACKLOG-N) so they can be referenced from the roadmap and code comments.

---

## BACKLOG-1 — RBAC Group Picker

**Public roadmap entry:** Planned  
**Status:** Not started

### Context
Currently the Roles admin UI (`/admin/roles`) requires an operator to paste an Entra security group Object ID manually. This is error-prone and requires the admin to have a separate browser tab open in Entra/Azure Portal.

### Proposed approach
- Add a `GET /api/groups/search?q={term}` endpoint in `Arm/EntraGroupService` using the Microsoft Graph `GET /groups?$search="displayName:{term}"` API (requires `ConsistencyLevel: eventual` header and `Group.Read.All` application permission).
- Add a debounced combobox component in the Roles admin page — type ≥ 3 characters to trigger search, display `displayName` + `id` in results, select to populate the OID field.
- Fallback: if the app SPN lacks `Group.Read.All`, the field remains a plain text input with an inline note.

### Dependencies
- ARM SPN must have `Group.Read.All` (or delegated `GroupMember.Read.All`) — add to the pre-requisites doc.

---

## BACKLOG-2 — Alerting Phase 2

**Public roadmap entry:** Planned  
**Status:** Not started

### Context
Phase 1 alerting covers: VM stopped, node down, health fault, Network ATC degraded, Arc connectivity, storage capacity. Phase 2 extends the rules engine.

### Planned rule types
| Rule | Source | Metric |
|---|---|---|
| Storage capacity threshold | S2D pool | Used % > configurable threshold |
| Update available | Solution Updates | Any available update detected |
| Arc connectivity lost | Arc status | Machine connectivity state = Disconnected |

### Alert acknowledgement
- Add `AcknowledgedAt` / `AcknowledgedBy` columns to `AlertHistory`.
- UI: inline acknowledge button on alert history rows; acknowledged alerts move to a separate "resolved" tab.

### Per-alert escalation
- Add an optional `EscalationWebhook` and `EscalationDelayMinutes` per rule — if alert is not acknowledged within the delay, re-fire to the escalation target.

---

## BACKLOG-3 — Historical Trending

**Public roadmap entry:** Planned  
**Status:** Not started

### Context
The background collector already polls CPU/memory/storage per cluster on a schedule. The data is not currently persisted beyond the latest snapshot.

### Proposed approach
- Add a `ClusterMetricHistory` table: `(ClusterId, CollectedAt, MetricType, NodeName, Value)`.
- Extend the background collector to insert a row for each metric on each poll cycle (with configurable retention, default 90 days).
- Add a `/cluster/{id}/trends` page using a charting library (e.g. Chart.js via `IJSRuntime`) to render 7/14/30-day utilisation graphs.
- Add a retention purge job (runs nightly, mirrors the audit log purge pattern).

---

## BACKLOG-4 — AKS Arc Test & Remediation (Blocked)

**Public roadmap entry:** Under Investigation / Blocked  
**Status:** Blocked — external module limitation

### Investigation log

**2026-03-xx** — Initial investigation by @jasontayler  
Attempted to invoke `Test-SupportAksArcKnownIssues` via `Invoke-Command` from the app server. The cmdlet internally opens a new WinRM session from the cluster node to other nodes — classic double-hop authentication problem. Constrained delegation and CredSSP were both evaluated:
- **Constrained delegation** — would require AD changes that are impractical for customers to configure.
- **CredSSP** — security risk; not acceptable as a default configuration requirement.

**Current state:**  
The `Support.AksArc` PowerShell module (as of v1.2.x) does not accept `-Credential` or `-PSSession` parameters on the diagnostic cmdlets, so there is no way to forward credentials without CredSSP.

**Re-evaluation triggers:**
1. The `Support.AksArc` module adds `-Session` / `-Credential` parameter support.
2. A JEA endpoint is available on cluster nodes (BACKLOG-6 dependency resolved first).
3. Microsoft exposes equivalent diagnostics via an ARM API.

---

## BACKLOG-5 — SIEM Sink for Audit Logs

**Public roadmap entry:** Planned  
**Status:** Not started

### Context
Customers with SIEM tooling (Sentinel, Splunk, Elastic) want to forward audit entries in real time rather than relying on CSV export.

### Proposed approach
- Add an `AuditSink` configuration section in Admin → Settings (type: `AzureMonitor | Splunk | Webhook`).
- On each `AuditService.RecordAsync` call, if a sink is configured, fire-and-forget post the entry to the sink endpoint.
- For Azure Monitor, use the DCR/Logs Ingestion API (`POST https://{dce}.ingest.monitor.azure.com/dataCollectionRules/{dcr}/streams/{stream}`).
- For Splunk / generic webhook, POST a JSON payload using the existing `HttpClient` pattern from the Teams alerting code.

---

## BACKLOG-6 — JEA Endpoints

**Public roadmap entry:** Planned  
**Status:** Not started

### Context
Currently the app server requires a service account with broad WinRM access to cluster nodes. A JEA (Just Enough Administration) endpoint would restrict the account to an approved cmdlet allowlist.

### Proposed approach
- Ship a `JEA\AzureLocalClusterTool.psrc` capability file listing every cmdlet the app invokes.
- Ship a `JEA\AzureLocalClusterToolSession.pssc` session configuration file.
- Add `scripts\Setup-JEA.ps1` to register the endpoint on each cluster node.
- Update `WinRmConnectionService` to use the JEA endpoint name when `UseJea = true` is configured.

---

## BACKLOG-7 — Scheduled Reporting

**Public roadmap entry:** Planned  
**Status:** Not started

### Context
Operations teams want a weekly summary email without having to log into the portal.

### Proposed approach
- Add a `ReportSchedule` table: `(Id, Name, CronExpression, Format [CSV|PDF], RecipientList, LastRunAt)`.
- Use the existing hosted service / background scheduler pattern to evaluate cron expressions and trigger report generation.
- CSV: direct from existing data queries. PDF: consider `QuestPDF` or `PuppeteerSharp` for headless rendering.
- Deliver via the existing SMTP service.
