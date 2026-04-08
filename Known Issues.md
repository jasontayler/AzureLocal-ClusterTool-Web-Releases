# Azure Local Cluster Tool — Web: Known Issues

> Report new bugs or feedback via [GitHub Issues](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues).

This page lists confirmed bugs and known limitations in the current release. Check back here before raising a new issue — yours may already be documented.

---

- Snapshot Freshness for Solution Updates shows Warning for stale date however data is not polled regualary as state changes do not occur frequently.

---

## General Limitations

- **VM creation** is not supported — the tool manages existing VMs only. Use Windows Admin Center or PowerShell to provision new VMs.
- **Live Update Monitor** streaming has not been fully validated against all update run states. The monitor page connects correctly, but output may stop if the ECE action plan completes very quickly.
- **WinRM connections** to cluster nodes require port 5985 (HTTP) or 5986 (HTTPS) to be reachable from the app server. Firewall rules blocking these ports will cause timeouts on pages that require a live cluster connection.

---

## Reporting a New Issue

Please include the following when opening an issue:

- Azure Local / Windows Server version
- App version (visible in the page footer)
- Browser console errors (F12)
- Relevant entries from **Admin → Perf Debug** (last WinRM/CIM call that failed)
- Application event log entries from the app server (`Event Viewer → Windows Logs → Application`)

[Open an issue](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues/new)