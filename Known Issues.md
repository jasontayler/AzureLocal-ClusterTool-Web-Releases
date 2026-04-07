# Azure Local Cluster Tool — Web: Known Issues

> Report new bugs or feedback via [GitHub Issues](https://github.com/jasontayler/AzureLocal-ClusterTool-Web-Releases/issues).

This page lists confirmed bugs and known limitations in the current release. Check back here before raising a new issue — yours may already be documented.

---

## Solution Updates

### Update Runs table shows duplicate entries

**Affected versions:** v0.9.12-rc1 and earlier  
**Symptom:** The Update Runs section on the Solution Updates page displays the same run multiple times — typically once for every installed update on the cluster. For example, if 6 updates are installed and one run is InProgress, that InProgress run appears 6 times in the table.  
**Cause:** The PowerShell pipeline `Get-SolutionUpdate | Get-SolutionUpdateRun` fires `Get-SolutionUpdateRun` once per update piped through it. When a run is InProgress, it is returned on every invocation rather than being scoped to its owning update.  
**Workaround:** The runs are genuine duplicates — there is only one actual run. Clicking **Details** on any of the duplicate rows opens the correct monitor.

---

### Starting an update shows an error even when the update started successfully

**Affected versions:** v0.9.12-rc1 and earlier  
**Symptom:** After pressing **Start** on a solution update (or after pressing **Get WinRM Data** which can trigger the update indirectly), the page shows a red error banner. However, checking the Solution Updates page again shows the update is actually running.  
**Cause:** `Start-SolutionUpdate` on the cluster frequently emits non-terminating PowerShell error records alongside a successful start — ARM synchronisation notices and similar informational output that the PowerShell SDK surfaces as errors. The app was treating any error record as a failure and reporting it to the UI.  
**Workaround:** If you see this error, wait 10–15 seconds and press **Refresh**. If the update state has moved to `Downloading`, `HealthChecking`, or `Installing`, the update started correctly. The error can be safely ignored.

---

### "Get WinRM Data" button can inadvertently trigger a pending update

**Affected versions:** v0.9.12-rc1 and earlier  
**Symptom:** Pressing **Get WinRM Data** on the Solution Updates page (the button that fetches the update catalog directly from the cluster, bypassing ARM) can cause a pending update to begin, or display a confusing error related to starting an update.  
**Cause:** ARM-sourced updates include a full `ResourceId` which is used as the identifier when the **Start** button is pressed. If WinRM data is loaded after ARM data, the `ResourceId` format may differ between the two sources and the mismatch can cause unexpected behaviour.  
**Workaround:** Use the **Refresh** button rather than **Get WinRM Data** when an update is in a transitional state.

---

## Windows Authentication Site (WinAuth) — API Access

### HTTP 500.19 Lock Violation after deploying web.config

**Affected versions:** Existing installs upgraded to v0.9.12 (new installs are not affected)  
**Symptom:** All requests to the WinAuth site return HTTP 500.19 after deploying the new `web.config` that enables anonymous authentication for the `/api` path.  
**Cause:** IIS `applicationHost.config` has `overrideModeDefault="Deny"` for authentication config sections by default. The `<location path="api">` block in `web.config` tries to override these sections, which is blocked.  
**Fix:** Run the following two commands on the IIS server (administrator PowerShell or Command Prompt), then recycle the app pool:

```
%windir%\system32\inetsrv\appcmd unlock config /section:system.webServer/security/authentication/anonymousAuthentication
%windir%\system32\inetsrv\appcmd unlock config /section:system.webServer/security/authentication/windowsAuthentication
%windir%\system32\inetsrv\appcmd recycle apppool /apppool.name:AZLManagementWinPool
```

New installs using `Setup-IIS-WinAuth.ps1` v0.9.12 or later run these unlock commands automatically.

---

### API key authentication fails with lowercase scheme variant

**Affected versions:** v0.9.12-rc1 and earlier  
**Symptom:** API calls using `Authorization: Apikey <token>` (lowercase k) return HTTP 401, while `Authorization: ApiKey <token>` (capital K) succeeds.  
**Cause:** The API key middleware used a case-sensitive prefix check.  
**Fix:** Included in v0.9.12. Any capitalisation of the scheme prefix (`ApiKey`, `Apikey`, `APIKEY`) is now accepted.

---

## Cluster Roles

### Group type shown as a number instead of a name

**Affected versions:** v0.9.12-rc1 and earlier  
**Symptom:** The **Type** column in the Cluster Roles table shows a raw integer (e.g. `111`) instead of a readable name like `Virtual Machine`.  
**Cause:** The integer-to-name mapping for MSCluster group types was incomplete. The correct mappings for Azure Local clusters are `108` = Storage Pool, `111` = Virtual Machine, `112` = Virtual Machine Configuration, `9999` = Azure Stack HCI Service.

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