# AKS Arc SPN Setup Guide

This guide covers the Azure configuration required to enable live AKS data
(namespaces, pod counts, unhealthy pods) on the AKS Overview page when the
cluster is Arc-connected (Azure Kubernetes Service on Azure Local).

---

## Overview

The portal uses an Azure Service Principal (SPN) configured in **Admin > Settings**.
For Arc-connected AKS clusters the SPN must be able to:

1. Call `listClusterUserCredential` on the connected cluster resource (ARM API).
2. Acquire a Proof-of-Possession (PoP) token via `kubelogin`.
3. List namespaces and pods through the Arc API server (k8s RBAC backed by Azure RBAC).

Each step requires a separate Azure role assignment.

---

## Prerequisites

Gather these values before starting. You will use them in Steps 1 and 3.

| Item | Where to find it |
|---|---|
| Subscription ID | Azure Portal → Subscriptions → [subscription] → Overview |
| Resource group | The resource group containing your Arc-connected cluster |
| Connected cluster name | Azure Portal → Azure Arc → Kubernetes clusters → [cluster name] |
| SPN Application (Client) ID | Entra Portal → App registrations → [your SPN] → Overview |
| SPN Object ID (enterprise app) | Entra Portal → Enterprise applications → [your SPN] → Overview |

> **Object ID vs Client ID** — Every app registration has both. The JWT token
> that kubelogin acquires contains the `oid` claim which is the **Object ID**
> from the Enterprise application (not the App registration). Azure RBAC and
> the Arc API server use the OID for all access checks, not the client ID.
> Use the Object ID from **Enterprise applications** in all `New-AzRoleAssignment` calls.

---

## Step 1 — Required Azure Role Assignments

All three roles must be assigned on the **connected cluster resource** scope
(not the resource group or subscription, though inherited assignments work too).

```
Scope: /subscriptions/<your-subscription-id>
       /resourceGroups/<your-resource-group>
       /providers/Microsoft.Kubernetes/connectedClusters/<your-cluster-name>
```

### Assign via PowerShell

```powershell
$spnOid = "<your-spn-object-id>"       # Enterprise application Object ID (not Client ID)
$scope  = "/subscriptions/<your-subscription-id>/resourceGroups/<your-resource-group>/providers/Microsoft.Kubernetes/connectedClusters/<your-cluster-name>"

# Role 1: get the kubeconfig from ARM (listClusterUserCredential)
New-AzRoleAssignment -ObjectId $spnOid `
    -RoleDefinitionName "Azure Arc Enabled Kubernetes Cluster User Role" `
    -Scope $scope

# Role 2: get AKS kubeconfig (AKS on Azure Local)
New-AzRoleAssignment -ObjectId $spnOid `
    -RoleDefinitionName "Azure Kubernetes Service Arc Cluster User Role" `
    -Scope $scope

# Role 3: perform k8s API calls (list namespaces, pods, etc.)
New-AzRoleAssignment -ObjectId $spnOid `
    -RoleDefinitionName "Azure Kubernetes Service Arc Cluster Admin Role" `
    -Scope $scope

# Role 4: read AND write Flux GitOps configurations (Force Resync requires write)
New-AzRoleAssignment -ObjectId $spnOid `
    -RoleDefinitionName "Flux Configurations Contributor" `
    -Scope $scope
```

### What each role does

| Role | What it enables | Without it |
|---|---|---|
| Azure Arc Enabled Kubernetes Cluster User Role | Call `POST .../listClusterUserCredential` to get the kubeconfig | ARM returns 403 AuthorizationFailed |
| Azure Kubernetes Service Arc Cluster User Role | Get AKS credentials for Arc-hosted AKS clusters specifically | ARM returns 403 |
| **Azure Kubernetes Service Arc Cluster Admin Role** | List/read k8s resources (namespaces, pods) through the Arc API server | k8s API returns 403 Forbidden — *this is the one most commonly missed* |
| **Flux Configurations Contributor** | Read and write `Microsoft.KubernetesConfiguration/fluxConfigurations` — required for Force Resync (PATCH); read-only operations (GitOps tab view) work with Reader but Force Resync needs Contributor | Force Resync returns 403 AuthorizationFailed |

> **Note:** The first two roles only grant the right to retrieve credentials.
> They do **not** grant any access inside the Kubernetes cluster. The third role
> is required for actual k8s API operations via the Arc relay.

---

## Step 2 — kubelogin on the App Server

The portal calls `kubelogin get-token` to acquire a PoP (Proof-of-Possession)
token. The binary must be present on the IIS app server and accessible to the
gMSA app pool process.

### Install kubelogin

```powershell
winget install Microsoft.Azure.Kubelogin
```

### Make it available to the IIS gMSA process

The IIS app pool identity uses the **machine-level PATH** only —
user-specific directories (e.g. `%LOCALAPPDATA%\Microsoft\WinGet\...`) from
a winget install are not visible to the gMSA (or service account) process.

Copy the binary to a machine-level PATH location:

```powershell
# Find where winget installed it
$src = (Get-Command kubelogin).Source

# Copy to System32 (in machine PATH by default)
Copy-Item $src "C:\Windows\System32\kubelogin.exe"
```

Verify it is accessible to all processes (not just your interactive session):

```powershell
# Run as SYSTEM or check machine PATH directly
$env:PATH -split ';' | ForEach-Object { Join-Path $_ "kubelogin.exe" } |
    Where-Object { Test-Path $_ }
```

---

## Step 3 — Admin > Settings Configuration

In the portal go to **Admin > Settings** and configure the SPN credentials:

| Setting | Value |
|---|---|
| ARM Auth Mode | `SPN` |
| SPN Tenant ID | Your Entra tenant ID |
| SPN Client ID | Your SPN Application (Client) ID |
| SPN Client Secret | The client secret you created for the SPN |

---

## How It Works (technical summary)

```
1. Portal calls ARM: POST .../connectedClusters/<cluster-name>/listClusterUserCredential
   → ARM returns a base64-encoded kubeconfig with exec plugin section
     (targeting Azure CLI / device-flow auth)

2. Portal deserializes kubeconfig YAML → K8SConfiguration object model

3. Portal extracts --server-id and --environment from the raw exec args

4. Portal runs kubelogin get-token directly (not via KubernetesClient's
   ExecuteExternalCommand) with explicit env vars set (TEMP, TMP,
   USERPROFILE, HOME) to work around the headless gMSA process context
   where Go binaries may produce empty output if these are missing:

   kubelogin get-token
     --login         spn
     --tenant-id     <your-tenant-id>
     --client-id     <your-spn-client-id>
     --client-secret <your-spn-client-secret>
     --server-id     6dae42f8-4368-4678-94ff-3960e28e3630  (fixed Azure KS app ID -- same for all tenants)
     --environment   AzurePublicCloud
     --pop-enabled
     --pop-claims    u=/subscriptions/<sub-id>/...connectedClusters/<cluster-name>

5. Portal parses the ExecCredential JSON response and extracts status.token
   (a PoP-bound JWT)

6. Portal sets UserCredentials.Token = <PoP token> and clears ExternalExecution,
   so KubernetesClient uses the static token for all k8s API calls — no
   further subprocess invocations

7. KubernetesClient calls:
   - CoreV1.ListNamespaceAsync()
   - CoreV1.ListPodForAllNamespacesAsync()
   → Arc API server validates the PoP token using Azure RBAC
   → Returns namespaces + pods if the SPN has Cluster Admin Role
```

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Arc: SPN lacks listClusterUserCredential/action` | Missing "Azure Arc Enabled Kubernetes Cluster User Role" | Add Step 1 Role 1 |
| `Arc: kubelogin not found` | kubelogin not in machine-level PATH | Copy binary to `C:\Windows\System32\` (Step 2) |
| `Arc: kubelogin get-token failed (exit N): ...` | Bad SPN credentials or kubelogin version mismatch | Check client secret hasn't expired; verify kubelogin >= v0.2.x |
| `403 Forbidden: cannot list resource "namespaces"` | Missing "Azure Kubernetes Service Arc Cluster Admin Role" | Add Step 1 Role 3 — this is the most commonly missed role |
| `Arc: AKS live data via Arc requires SPN auth mode` | Auth mode set to OBO | Set ARM Auth Mode = SPN in Admin > Settings |
| Token acquired but k8s calls still fail | Role assignment propagation delay | Wait 2-5 minutes after adding the role assignment |

---

## Standard Service Account vs gMSA

The Azure role assignments (Step 1) and kubelogin machine PATH (Step 2) are **identical** regardless of which IIS application pool identity you use.

The difference is how the process environment is populated when IIS launches the app:

### gMSA (`DOMAIN\account$`)

IIS never loads a user profile for a gMSA identity. This means the Go binary (`kubelogin`) starts with no `USERPROFILE`, `HOME`, or user-scoped `TEMP`. On any Windows system a Go binary writes temporary files to `%TEMP%` on startup — if that variable is missing the binary writes no stdout and the token fetch silently fails.

The app works around this by explicitly setting `TEMP`, `TMP`, `USERPROFILE`, and `HOME` to `Path.GetTempPath()` in the `ProcessStartInfo` environment before spawning `kubelogin`. No manual configuration is needed.

### Standard domain service account

IIS can load the user profile for a standard account if **Load User Profile** is enabled on the application pool (IIS Manager > Application Pools > [pool] > Advanced Settings > Load User Profile = True). When enabled, `USERPROFILE`, `HOME`, and `TEMP` are all set to the account's actual profile directory.

However, **Load User Profile is not enabled by default** for IIS application pools. If it is disabled, the same missing-env-vars problem applies as with gMSA.

The app sets the env vars explicitly using `Path.GetTempPath()` regardless, so it works correctly in both cases — you do not need to enable Load User Profile.

### Summary

| | gMSA | Standard service account |
|---|---|---|
| Azure role assignments | Same | Same |
| kubelogin PATH (machine-level) | Same | Same |
| USERPROFILE / HOME / TEMP set by IIS | Never | Only if Load User Profile = True |
| App sets env vars explicitly | Yes | Yes (as fallback) |
| Action required | None extra | None extra |
