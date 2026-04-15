# Quick Start — Azure Local Cluster Tool (Web)

High-level steps to get from a fresh Windows Server to a running portal.
For detailed script parameters, reference material, and troubleshooting, see the [Deployment Guide](IIS-Deployment.md).

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Windows Server 2019+ (domain-joined) | Must be joined to the same domain as the cluster nodes for Kerberos WinRM authentication |
| gMSA or standard service account | Created in Step 3. Must have WinRM and local admin access on each cluster node |
| Entra ID app registration | Free. Created in Step 1. Required for browser SSO |
| WinRM open from app server to cluster nodes | Port 5985 (HTTP) or 5986 (HTTPS) |
| Internet access from app server | Required during `Setup-Prerequisites.ps1` for winget downloads |

> `Setup-Prerequisites.ps1` automatically installs IIS, .NET Hosting Bundle, PowerShell 7, kubelogin, and PostgreSQL. No manual installation of these components is required.

---

## Step 1 — Set up Entra ID (5 minutes)

1. In **Entra Portal** → App registrations → **New registration**
   - Name: `AzureLocalClusterTool` (or similar)
   - Supported account types: **Accounts in this organizational directory only**
   - Redirect URI: `https://<your-server-hostname>/signin-oidc`
   - Add a second redirect URI: `https://<your-server-hostname>/signout-callback-oidc`
2. Under **Authentication** → enable **ID tokens**
3. Under **Manifest**, set `"groupMembershipClaims": "SecurityGroup"`
4. Under **Certificates & secrets** → create a **Client secret** and copy the value
5. Create 1–3 **Entra security groups** (or reuse existing):
   - `HciRead` / `HciOperate` / `HciAdmin` — or all three can point to the **same group** for a single access level
   - Copy each group's **Object ID** from Entra Portal → Groups → [group] → Overview

---

## Step 2 — Set up Application Proxy (recommended)

Entra Application Proxy lets external users reach the portal over HTTPS without opening firewall ports or requiring a VPN. Skip this step if all users are on the same LAN as the app server.

1. Follow the Microsoft guide: [Add an on-premises application — Entra Application Proxy](https://learn.microsoft.com/en-us/entra/identity/app-proxy/application-proxy-add-on-premises-application)
2. Install the **Entra Application Proxy connector** on the app server (or a connector server on the same network)
3. Set the **internal URL** to your IIS site address (e.g. `https://azlmgmt.yourdomain.com`)
4. Add the **external URL** provided by Entra as an additional redirect URI in your app registration

---

## Step 3 — Create a service account or gMSA

The IIS app pool runs as this account. It uses Kerberos (no stored password) to authenticate WinRM connections to cluster nodes, so it needs **local Administrator membership on each cluster node**.

**gMSA (recommended):** Run `New-AppServiceAccount.ps1` on a DC or a machine with RSAT AD tools:

```powershell
.\scripts\New-AppServiceAccount.ps1 `
    -AppServerName "azlmgmt" `
    -ClusterNodes  @("NODE1", "NODE2", "NODE3")
```

**Standard service account:** add `-AccountType Standard` to the command above.

For full options, see [Deployment Guide — gMSA Requirements](IIS-Deployment.md#gmsa-requirements).

---

## Step 4 — Download and extract

1. Download the latest release ZIP from the [Releases](../../releases) page
2. Extract to a temporary folder on the **app server**, e.g. `C:\temp\alm-release\`

Contents:

```
AzureLocal.ClusterTool.Web/    app binaries
scripts/                       setup scripts (run these on the app server)
docs/                          deployment and user documentation
appsettings.template.json      configuration template
```

---

## Step 5 — Run Setup-Prerequisites.ps1

On the app server, open **PowerShell as Administrator** and run:

```powershell
cd C:\temp\alm-release\scripts
.\Setup-Prerequisites.ps1 -ServiceAccount "DOMAIN\azlmgmt-svc$"
```

This installs in one pass — **no manual steps required:**
- IIS and all required Windows features
- ASP.NET Core Hosting Bundle (.NET 10)
- PowerShell 7
- kubelogin (for AKS on Azure Local)
- PostgreSQL 17 — creates the `azlmgmt` database and `azlmgmt_app` user
- Application directories with correct NTFS permissions for the service account

> **Restart if prompted.** Re-run the script after rebooting if Windows features need a restart.

**Default paths:** `C:\apps\azlmgmt` (binaries) and `C:\apps\azlmgmt-data` (runtime data).
To use different paths, pass `-AppPath` and `-DataPath`:
```powershell
.\Setup-Prerequisites.ps1 -ServiceAccount "DOMAIN\azlmgmt-svc$" `
    -AppPath  "D:\apps\azlmgmt" `
    -DataPath "D:\apps\azlmgmt-data"
```

At the end the script prints a `Database` and `DataProtection` JSON snippet — **copy this for Step 6**.

---

## Step 6 — Create appsettings.Production.json

Copy `appsettings.template.json` to `C:\apps\azlmgmt\appsettings.Production.json` and fill in:

```json
{
  "AzureAd": {
    "TenantId":     "<your-entra-tenant-id>",
    "ClientId":     "<your-app-registration-client-id>",
    "ClientSecret": "<client-secret-from-step-1>"
  },
  "Groups": {
    "HciRead":    "<object-id>",
    "HciOperate": "<object-id>",
    "HciAdmin":   "<object-id>"
  },
  "Database": {
    "Provider":         "PostgreSQL",
    "ConnectionString": "<paste from Setup-Prerequisites.ps1 output>"
  },
  "DataProtection": {
    "KeyPath": "<paste from Setup-Prerequisites.ps1 output>"
  }
}
```

Then open `clusters.json` and add your cluster(s):

```json
[
  {
    "name":             "MY-CLUSTER-01",
    "address":          "MY-CLUSTER-01.domain.local",
    "credentialSource": "gMSA",
    "description":      "Production cluster 1"
  }
]
```

`credentialSource` options: `"gMSA"` (use app pool identity — recommended) or `"KeyVault"` (see [Deployment Guide](IIS-Deployment.md)).

---

## Step 7 — Install the app and create the IIS site

```powershell
# Copies app binaries to C:\apps\azlmgmt (or the AppPath you chose in Step 5)
.\Install.ps1

# Creates the IIS site and app pool
.\Setup-IIS.ps1 `
    -siteName       "AZLManagement" `
    -appPoolName    "AZLManagementPool" `
    -physicalPath   "C:\apps\azlmgmt" `
    -hostHeader     "azlmgmt.yourdomain.com" `
    -serviceAccount "DOMAIN\azlmgmt-svc$"

# Add HTTPS binding with a self-signed certificate
.\Add-HttpsBinding.ps1 -SiteName "AZLManagement" -HostHeader "azlmgmt.yourdomain.com"
```

`Setup-IIS.ps1` creates the site, configures the app pool identity, sets NTFS permissions, and applies idle timeout and periodic restart settings required for Blazor Server.

For a standard service account (not gMSA), also pass `-serviceAccountType Standard -serviceAccountPassword "yourpassword"`.

---

## Step 8 — First sign-in

1. Navigate to `https://azlmgmt.yourdomain.com`
2. Sign in with your Entra ID account (must be in a configured security group)
3. Go to `/admin/clusters` to register your first cluster

---

## Upgrading

Extract the new release ZIP to the app server, then from the `scripts\` folder:

```powershell
.\Install.ps1 -Upgrade
# If your pool has a non-default name:
.\Install.ps1 -Upgrade -AppPoolName "YourPoolName"
```

Stops the app pool, replaces binaries, restarts. `appsettings.Production.json` and `clusters.json` are preserved.

> **Building from source:** use `Deploy-ToIIS.ps1` from your dev machine instead.

---

## Common Issues

| Problem | Solution |
|---|---|
| Sign-in redirect loop | Verify `groupMembershipClaims: SecurityGroup` in the Entra app manifest |
| "Access denied" after sign-in | Account not in a configured Entra security group |
| App pool starts then stops immediately | gMSA not installed on app server — run `Test-ADServiceAccount -Identity <name>` |
| WinRM errors on cluster pages | Port 5985 not open; or gMSA not in Administrators on cluster nodes |
| Blank VM / Node pages | gMSA not in cluster node local Administrators group on the nodes |
| AKS page shows no data | ARM SPN not configured — see [AKS-Arc-SPN-Setup.md](AKS-Arc-SPN-Setup.md) |

For detailed troubleshooting, see [Deployment Guide — Troubleshooting](IIS-Deployment.md#troubleshooting).

---

## Getting Help

- **Bug reports / feedback:** [GitHub Issues](../../issues)
- **User guide:** [docs/USER-GUIDE.md](USER-GUIDE.md)
- **Deployment guide:** [docs/IIS-Deployment.md](IIS-Deployment.md)
