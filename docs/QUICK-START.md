# Quick Start — Azure Local Cluster Tool (Web)

This guide gets you from a fresh Windows Server to a running portal in about 30 minutes.
For full details on every step, see the [IIS Deployment Guide](IIS-Deployment.md).

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Windows Server 2022+ with IIS | IIS role must be installed (`Install-WindowsFeature Web-Server -IncludeManagementTools`) |
| No .NET runtime required | The build is self-contained |
| gMSA service account | Must have WinRM access to cluster nodes. See [IIS Deployment Guide](IIS-Deployment.md) for setup. A standard Windows service account also works. |
| WinRM open from the app server | Port 5985 (HTTP) or 5986 (HTTPS) to each cluster node |
| Entra ID app registration | Free — used for browser SSO. Takes 5 minutes. See below. |

---

## Step 1 — Download and Extract

1. Download the latest release ZIP from the [Releases](../../releases) page
2. Extract to a temporary folder on your app server, e.g. `C:\temp\alm-release\`

The extracted folder contains:

```
AzureLocal.ClusterTool.Web/    app binaries
scripts/                       setup and install scripts (run these on the app server)
docs/                          user guide and deployment documentation
appsettings.template.json      configuration template — Install.ps1 copies this into place
clusters.json.example          starting cluster list — Install.ps1 copies this into place
CHANGELOG.md                   release notes
```

---

## Step 2 — Configure Entra ID (5 minutes)

1. In **Entra Portal** → App registrations → **New registration**
   - Name: `AzureLocalClusterTool` (or similar)
   - Supported account types: **Accounts in this organizational directory only**
   - Redirect URI: `https://<your-server-hostname>/signin-oidc`  
     Add a second redirect URI: `https://<your-server-hostname>/signout-callback-oidc`
2. Under **Authentication** → enable **ID tokens**
3. Under **Manifest**, set `"groupMembershipClaims": "SecurityGroup"`
4. Under **Certificates & secrets** → create a **Client secret** and copy the value
5. Create 1–3 **Entra security groups** (or reuse existing):
   - `HciRead` — users who can view data
   - `HciOperate` — users who can perform operations (includes view)
   - `HciAdmin` — administrators (includes everything)
   - All three can be the **same group** if you only need one access level.
   - Get each group's **Object ID** from Entra Portal → Groups → [group] → Overview

---

## Step 3 — Edit appsettings.Production.json

`scripts\Install.ps1` (run in Step 5) copies `appsettings.template.json` to `AzureLocal.ClusterTool.Web\appsettings.Production.json` for you.
Open that file and fill in the placeholders:

```json
{
  "AzureAd": {
    "TenantId":     "<your-entra-tenant-id>",
    "ClientId":     "<your-app-registration-client-id>",
    "ClientSecret": "<client-secret-from-step-2>"
  },
  "Groups": {
    "HciRead":    "<object-id-of-hciread-group>",
    "HciOperate": "<object-id-of-hcioperate-group>",
    "HciAdmin":   "<object-id-of-hciadmin-group>"
  },
  "Database": {
    "Provider":         "PostgreSQL",
    "ConnectionString": "Host=127.0.0.1;Database=azlmgmt;Username=azlmgmt_app;Password=changeme;Keepalive=60;Connection Idle Lifetime=300;Timeout=30;Command Timeout=60"
  }
}
```

> **Database options:**
> - **PostgreSQL** — recommended; free, no row-count limits. Run `scripts\Setup-Prerequisites.ps1` or configure manually.
> - **SQL Server** — enterprise environments. Set `"Provider": "SqlServer"`. #noting this is not 100% tested recommend PostgreSQL 

---

## Step 4 — Configure Clusters

`scripts\Install.ps1` (run in Step 5) also creates `AzureLocal.ClusterTool.Web\clusters.json` from the example file.
Open it and add your cluster(s):

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

`credentialSource` options:
- `"gMSA"` — use the IIS app pool identity (recommended; no password)
- `"KeyVault"` — fetch credentials from Azure Key Vault (see [IIS Deployment Guide](IIS-Deployment.md))

---

## Step 5 — Install the App and Configure IIS (first time only)

On the app server, open **PowerShell as Administrator** and run:

```powershell
cd C:\temp\alm-release\scripts

# 1. Install prerequisites (IIS features, ANCM, PostgreSQL)
.\Setup-Prerequisites.ps1 -serviceAccount "DOMAIN\azlmgmt-svc$"

# Restart if prompted after feature installation, then re-run

# 2. Copy app binaries and place config files
.\Install.ps1
# This copies AzureLocal.ClusterTool.Web\ to C:\apps\azlmgmt\ and creates
# appsettings.Production.json and clusters.json from the templates.
# Edit those two files now (Steps 3 and 4 above) before continuing.

# 3. Create the IIS site and application pool
.\Setup-IIS.ps1 `
    -siteName      "AZLManagement" `
    -appPoolName   "AZLManagementPool" `
    -physicalPath  "C:\apps\azlmgmt" `
    -hostHeader    "azlmgmt.yourdomain.com" `
    -serviceAccount "DOMAIN\azlmgmt-svc$"
```

`Setup-IIS.ps1`:
- Validates IIS features and ASP.NET Core Module are installed
- Creates the IIS site and application pool running as the service account
- Sets NTFS permissions on the app and data directories
- Configures ASPNETCORE_ENVIRONMENT and SignalR-safe recycle settings

> **Service account:** Replace `DOMAIN\azlmgmt-svc$` with your gMSA or standard service account.
> The gMSA must be created in AD and given local admin rights on each cluster node.
> For a standard account, also pass `-serviceAccountPassword "yourpassword"`.

---

## Step 6 — Add HTTPS Binding (recommended)

```powershell
.\Add-HttpsBinding.ps1 -SiteName "AZLManagement" -HostHeader "azlmgmt.yourdomain.com"
```

A self-signed certificate is created automatically. Replace with a CA-signed certificate or
Let's Encrypt certificate for production. Entra ID requires HTTPS for redirect URIs.

---

## Step 7 — First Sign-In

1. Navigate to `https://azlmgmt.yourdomain.com` in a browser
2. Sign in with your Entra ID account (must be a member of one of the groups you configured)
3. You will be redirected to the home page showing your registered clusters
4. Click a cluster to connect and begin managing it

**If sign-in fails:** Verify the redirect URIs in your Entra app registration exactly match
the URL you are browsing to (including `https://` and no trailing slash).

---

## Upgrading

Download the latest release ZIP, extract it, and from the `scripts` folder run:

```powershell
# Default app pool name (AZLManagementPool)
.\Install.ps1 -Upgrade

# If your pool has a different name
.\Install.ps1 -Upgrade -AppPoolName "HCIPortalPool" -WinAuthPoolName "HCIPortalWinPool"
```

This stops the app pool(s), replaces the binaries, and restarts. `appsettings.Production.json`
and `clusters.json` are preserved. No database changes are needed for minor/patch upgrades.
For major upgrades, check the [Changelog](../CHANGELOG.md) for any required migration steps.

> **Developer workflow:** If you are building from source, use `Deploy-ToIIS.ps1` instead —
> it publishes, compiles, and deploys in one step directly from the source tree.

---

## Common Issues

| Problem | Solution |
|---|---|
| Sign-in redirect loop | Verify `groupMembershipClaims: SecurityGroup` in the Entra app manifest |
| "Access denied" after sign-in | Your account is not in a configured Entra security group |
| No clusters listed | No clusters registered, or your RBAC role doesn't include `Clusters → View` |
| WinRM errors on cluster pages | Verify port 5985 is open from the app server to cluster nodes; check the gMSA has WinRM access |
| Blank VM / Node pages | gMSA may not be in the cluster's local Administrators group on the nodes |

For more detail, see the [IIS Deployment Guide](IIS-Deployment.md) troubleshooting section.

---

## Getting Help

- **Bug reports / feedback:** [GitHub Issues](../../issues) — use the provided templates
- **User guide:** [docs/USER-GUIDE.md](USER-GUIDE.md)
- **Deployment guide:** [docs/IIS-Deployment.md](IIS-Deployment.md)
