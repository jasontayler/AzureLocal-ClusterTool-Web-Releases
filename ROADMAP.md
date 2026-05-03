# Azure Local Cluster Tool — Roadmap

This is the public-facing roadmap for the Azure Local Cluster Tool. No fixed timelines — items move from *Planned* → *In Progress* → shipped in the [Changelog](CHANGELOG.md).

> **Want to suggest a feature?** [Open a feature request →](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/new?template=feature_request.yml)

---

## 🚀 Planned

Items that have been accepted and will be built in an upcoming release.

| Feature | Description |
|---|---|
| **RBAC group picker** | Search and select Entra groups by name in the Roles admin UI instead of pasting Object IDs manually |
| **Alerting — Phase 2** | Additional rule types (storage threshold, update available, Arc connectivity); alert acknowledgement; per-alert escalation policy |
| **Historical trending** | Background collection of CPU/memory/storage utilisation over time; capacity forecasting graphs |
| **SIEM sink for audit logs** | Forward audit entries to Azure Monitor / Log Analytics, Splunk, or a generic HTTP webhook |
| **Scheduled reporting** | Weekly cluster health report emailed as CSV/PDF |
| **ARM REST strategic review** | Evaluate replacing WinRM reads with ARM API calls for clusters on Azure Local 25H2+ |
| **JEA endpoints** | Restrict WinRM to an approved cmdlet allowlist for least-privilege access from the app server |

---

## 💬 Community Requested — Under Consideration

The following ideas have been raised by the community. They have been acknowledged and are being evaluated for a future release. Links back to the original issue are included so you can follow along or add context.

| Feature | Raised in | Summary |
|---|---|---|
| **Detect and alert on unclustered VMs** | [#24](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/24) | Identify VMs running on a node but no longer registered as a clustered role in Failover Cluster Manager, and generate an alert |
| **Built-in alert: Mgmt Tool Health Summary** | [#22](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/22) | A new built-in alert type that summarises overall management tool health in a single notification |
| **Sortable dashboard columns + separate Hyper-V** | [#20](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/20) | Allow column sorting on the Fleet Status dashboard; visually distinguish standalone Hyper-V hosts from Azure Local clusters |
| **Confirmation dialogs for action buttons** | [#19](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/19) | Add confirmation prompts before destructive or impactful actions (e.g. VM stop, node drain) to reduce accidental operations |
| **Hyper-V Replica status** | [#18](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/18) | Expose replication health, state, and lag for VMs configured with Hyper-V Replica on Azure Local clusters |
| **Enhanced search — VMs + AI/natural language** | [#17](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/17) | Expand search scope across VMs fleet-wide; optionally support AI-assisted / natural language queries |

> Have more ideas? [Submit a feature request](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/new?template=feature_request.yml) — every suggestion is read and considered.

---

## 🔬 Under Investigation / Blocked

Items that are desirable but are currently blocked by an external dependency or technical constraint.

| Feature | Blocker | Detail |
|---|---|---|
| **AKS Arc test & remediation** | `Support.AksArc` module does not support remote execution | `Test-SupportAksArcKnownIssues` and `Invoke-SupportAksArcRemediation` internally WinRM to each cluster node — unresolvable double-hop with current tooling. Will revisit when the module accepts `-Credential`/`-Session`, or a JEA endpoint is available. See [BACKLOG-4](.github/BACKLOG.md) for the full investigation log. |

---

## ✅ How to Submit an Idea

1. Check [existing issues](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues) to see if your idea has already been raised.
2. If not, [open a feature request](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/new?template=feature_request.yml) using the template — describe the operational problem you are solving and what you would like to see.
3. Accepted ideas will be added to the **Community Requested** section above so they are not lost and you can track progress.

---

*For detailed implementation notes, investigation logs, and internal context see [`.github/BACKLOG.md`](.github/BACKLOG.md).*  
*For a full history of shipped features see the [Changelog](CHANGELOG.md).*
