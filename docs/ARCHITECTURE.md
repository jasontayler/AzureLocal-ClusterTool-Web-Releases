# Azure Local Cluster Tool — Architecture

This document describes the system architecture through a series of diagrams.  
All diagrams use [Mermaid](https://mermaid.js.org/) syntax (rendered by GitHub, VS Code Mermaid Preview, etc.).

---

## Contents

1. [Overall Architecture](#1-overall-architecture)
2. [Authentication & Authorisation Flow](#2-authentication--authorisation-flow)
3. [Azure ARM Connection Modes](#3-azure-arm-connection-modes)
4. [Cluster Communication & Page Caching](#4-cluster-communication--page-caching)
5. [Service Dependency Map](#5-service-dependency-map)
6. [Deployment Topology](#6-deployment-topology)

---

## 1. Overall Architecture

Shows every actor and system boundary — users, the app server, Entra ID, Azure ARM, and the on-premises cluster.

```mermaid
graph TB
    subgraph Users["Users"]
        UA["Admin / Operator\n(domain-joined browser)"]
        UB["Read-only User\n(Entra ID account)"]
    end

    subgraph EntraID["Microsoft Entra ID (cloud)"]
        OIDC["OIDC / OAuth2\nTenant: 647ab81b"]
        Groups["Security Groups\nHciRead · HciOperate · HciAdmin"]
    end

    subgraph AppServer["App Server — azlmgmt.yourdomain.com"]
        IIS["IIS (InProcess)\nAZLManagementPool"]
        Blazor["Blazor Server (.NET 10)\nInteractiveServer circuits"]
        DB[("SQLite / SQL Server\nClusters · AuditLog\nAppSettings · RBAC")]
        DPKeys["DPAPI Key Ring\nC:\\apps\\azlmgmt-data\\dp-keys"]
        gMSA["gMSA Identity\nDOMAIN\\azlmgmt-svc$"]
    end

    subgraph AzureARM["Azure Resource Manager (cloud)"]
        ARM["ARM REST API\nmanagement.azure.com"]
        KV["Azure Key Vault\n(optional credential store)"]
    end

    subgraph Cluster["Azure Local Cluster — AZ-NUC-CL01.jase.org"]
        CNS["Cluster endpoint\n(WinRM :5985)"]
        N1["Node 1\nWinRM :5985"]
        N2["Node 2\nWinRM :5985"]
        N3["Node N\nWinRM :5985"]
    end

    UA -->|"HTTPS + Negotiate/Kerberos\n(WinAuth site)"| IIS
    UB -->|"HTTPS + OIDC cookie\n(Entra ID site)"| IIS
    IIS --> Blazor
    Blazor -->|"OIDC sign-in redirect"| OIDC
    OIDC -->|"ID token + group claims"| Blazor
    Groups -.->|"OIDs embedded in claims"| OIDC
    Blazor <-->|"EF Core"| DB
    DB -.->|"encrypted secrets\nDPAPI unprotect on startup"| DPKeys
    Blazor -->|"OBO token or SPN creds\nArc / AKS / ARM pages"| ARM
    Blazor -->|"fetch cluster credentials\n(KeyVault source only)"| KV
    gMSA -.->|"Kerberos process identity\nused for WinRM + CIM auth"| IIS
    Blazor -->|"WinRM RunspacePool(1-5)\nmutations + cmdlets with\nno CIM equivalent"| CNS
    Blazor -->|"CIM (MSCluster + storage)\nmost reads — no wsmprovhost.exe"| CNS
    Blazor -->|"CIM per-node (Win32 + storage)\nVM queries, node stats"| N1
    Blazor -->|"CIM per-node"| N2
    Blazor -->|"WinRM direct Runspace\nGet-NetAdapter only"| N1
```

**Key points**

| Concern | Detail |
|---|---|
| App server | Single Windows Server, IIS InProcess hosting, self-contained .NET 10 executable |
| Identity for WinRM / CIM | gMSA `DOMAIN\azlmgmt-svc$` — no passwords to rotate; Kerberos ticket obtained from process identity |
| Transport hierarchy | Most reads: C# CIM API (no PS process on cluster nodes). Hyper-V / FailoverClusters mutations: local RSAT PS (`_localPool`) with `-CimSession`/`-Cluster`. WinRM RunspacePool retained for mutations lacking a local-PS equivalent. `Get-NetAdapter` remains per-node WinRM (module not on app server). |
| RSAT required | App server must have `RSAT-Hyper-V-Tools` and `RSAT-Clustering-PowerShell` installed (`Setup-Prerequisites.ps1` handles this). |
| Database | SQLite by default (`C:\apps\azlmgmt-data\app.db`); swap to SQL Server via `appsettings.json` with no code changes |
| Secret storage | Sensitive settings (SPN secrets, `AzureAd:ClientSecret`) stored AES-encrypted in DB via DPAPI; decrypted at startup before MSAL registers |

---

## 2. Authentication & Authorisation Flow

Two parallel authentication modes exist on separate IIS sites.

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant IIS as IIS (azlmgmt.yourdomain.com)
    participant Blazor as Blazor Server
    participant Entra as Microsoft Entra ID
    participant RBAC as RbacService (DB snapshot)

    User->>Browser: Navigate to protected page
    Browser->>IIS: GET /clusters/x/vms

    alt No auth cookie (Entra ID site)
        IIS->>Blazor: Forward unauthenticated request
        Blazor-->>Browser: 302 → /MicrosoftIdentity/Account/SignIn
        Browser->>Entra: OIDC Authorization Request
        Entra-->>User: Sign-in prompt (MFA if configured)
        User->>Entra: Credentials
        Entra-->>Browser: Authorization code
        Browser->>Blazor: POST /signin-oidc (code)
        Blazor->>Entra: Token exchange → access_token + id_token
        Entra-->>Blazor: Tokens + group Object ID claims
        Blazor-->>Browser: Encrypted auth cookie (sliding expiry)
    else Windows Auth site (azlocalmgmt-win.jase.org)
        IIS->>IIS: Kernel-mode Kerberos/NTLM challenge
        Note over IIS: Browser sends ticket automatically\n(domain-joined machine)
        IIS-->>Blazor: HttpContext.User = WindowsIdentity\ngroupsid claims = AD group SIDs
    end

    Blazor->>Blazor: FallbackPolicy — RequireAuthenticatedUser ✓
    Blazor->>Blazor: Named policy check\nHciRead / HciOperate / HciAdmin group OID or SID

    Blazor->>RBAC: HasViewAccess(userGroupIds, ResourceType)
    Note over RBAC: 5-minute snapshot cache of\nCustomRoles + Claims + Assignments
    RBAC-->>Blazor: true / false

    alt Access denied
        Blazor-->>Browser: Render .alert-error block
    else Access granted
        Blazor->>Blazor: FilterByPattern(items, nameSelector)\nGetAllowedOps(name) per row
        Blazor-->>Browser: Page rendered with RBAC-filtered data\nand per-item action buttons
    end
```

**Access tiers**

| Policy | Satisfied by | Typical assignment |
|---|---|---|
| `HciRead` | HciRead, HciOperate, or HciAdmin group | All staff who need view access |
| `HciOperate` | HciOperate or HciAdmin group | Operators who start/stop VMs, drain nodes |
| `HciAdmin` | HciAdmin group only | IT admins — cluster CRUD, audit log, RBAC roles |

> **HciAdmin users bypass all custom RBAC** — they always have full access regardless of role definitions.

---

## 3. Azure ARM Connection Modes

Used by Arc, AKS, logical networks, and extension pages. Controlled by `Azure:ArmAuthMode` in `appsettings.json` / Admin → Settings.

```mermaid
graph TB
    subgraph Blazor["Blazor Server — AzureArmService (scoped)"]
        ArmSvc["AzureArmService\nGetAsync · PostAsync\nPatchAsync · DeleteAsync"]
        TokenSel{"GetTokenAsync\nisWriteOperation?"}
    end

    subgraph OBO["Mode: OBO — On-Behalf-Of (default)"]
        OBOFlow["ITokenAcquisition\n.GetAccessTokenForUserAsync()\nscope: management.azure.com/user_impersonation"]
        UserToken["Bearer = signed-in user identity\nARM audit trail shows real UPN\nRequires: AzureAd:ClientSecret\n+ user_impersonation delegated permission"]
    end

    subgraph SPN["Mode: SPN — Service Principal"]
        SPNCred["ClientCredential\nAzure:Spn:TenantId\nAzure:Spn:ClientId / ClientSecret"]
        SPNToken["Bearer = service principal identity\nRequires ARM Reader RBAC\non each subscription"]
    end

    subgraph ActionSPN["Action SPN — optional write override"]
        ActEnabled{"Azure:ActionSpn\n:Enabled = true?"}
        ActCred["ClientCredential\nActionSpn TenantId/ClientId/Secret\nLeast-privilege write SPN"]
        ActToken["Bearer = action SPN identity\nApplied to POST / PUT / PATCH / DELETE only\nRead calls still use primary mode"]
    end

    subgraph ARM["Azure Resource Manager"]
        ReadAPI["GET reads\nArc machines · Arc extensions\nAKS clusters · Logical networks\nCustom locations · Resource bridge"]
        WriteAPI["POST / PATCH / DELETE\nExtension upgrades\nRetry-intent operations"]
    end

    ArmSvc --> TokenSel
    TokenSel -->|"read"| OBO
    TokenSel -->|"read"| SPN
    TokenSel -->|"write"| ActEnabled
    OBO --> OBOFlow --> UserToken --> ReadAPI
    SPN --> SPNCred --> SPNToken --> ReadAPI
    ActEnabled -->|"yes"| ActCred --> ActToken --> WriteAPI
    ActEnabled -->|"no — fall through"| OBO
    ActEnabled -->|"no — fall through"| SPN
    SPNToken --> WriteAPI
    UserToken --> WriteAPI
```

**Choosing a mode**

| Scenario | Recommended mode |
|---|---|
| Enterprise: user_impersonation allowed on app registration | OBO — audit trail maps to individuals |
| Enterprise: SPN required, writes are rare | SPN for reads + Action SPN for writes |
| Lab / single admin | OBO is simplest |

---

## 4. Cluster Communication & Page Caching

Every cluster page follows a stale-while-revalidate pattern so repeat navigation is instant.

```mermaid
graph TB
    subgraph Page["Blazor Page — LoadAsync()"]
        Start(["OnParametersSetAsync"])
        CacheCheck["ClusterPageCache.Get(cluster, page)"]
        CacheHit{"Cache hit\nand within TTL?"}
        ShowCached["ApplySnapshot() immediately\n_loading = false\nStateHasChanged()"]
        BgRefresh["Task.Run — background fetch\nCache.Set(snap)\nInvokeAsync(StateHasChanged)"]
        BlockFetch["Await FetchSnapshotAsync()\nshow skeleton while waiting"]
        StoreFresh["Cache.Set(snap)\nApplySnapshot()"]
    end

    subgraph Pool["ClusterConnectionPool (singleton)"]
        SemLock["SemaphoreSlim(1) per cluster\nprevents double-connect race"]
        SvcMap["ConcurrentDictionary\ncluster → WebHyperVService"]
    end

    subgraph Svc["WebHyperVService (one per cluster)"]
        RP["RunspacePool(1-5)\ncluster-level mutations\n(ATC, ECE, Storage, Arc,\nClusterInfo, Get-HealthFault)"]
        LocalPool["_localPool (local RSAT PS)\nHyper-V + FailoverClusters\nwith -CimSession / -Cluster"]
        CimCluster["_cimSession (CIM to cluster)\nMSCluster_Node / ResourceGroup\nMSCluster_Network / CSV\nMSFT_Storage classes"]
        NodeCim["_nodeCimSessionCache\nCIM per node (Win32, Hyper-V)\nFetchNodeStats, VM step 2"]
        NodeWinRM["GetOrCreateCachedRunspace\nWinRM per-node (5-min TTL)\nGet-NetAdapter only"]
        Log["InvokeLogged() -> ps.Invoke()\nPsCallLog ring buffer (500)\n/admin/debug-perf\nBadge: WinRM / CIM / ARM / Local"]
    end

    subgraph WinRM["Remote endpoints"]
        CEP["Cluster WSMan\nHTTP :5985 (default)\nHTTPS :5986 (optional)"]
        N1["Node 1 :5985\ndirect WinRM\n(Get-NetAdapter only)"]
        N2["Node N WMI (CIM)\nno PS shell spawned"]
    end

    Start --> CacheCheck --> CacheHit
    CacheHit -->|"yes"| ShowCached --> BgRefresh --> Pool
    CacheHit -->|"no"| BlockFetch --> Pool
    BlockFetch --> StoreFresh
    Pool --> SemLock --> SvcMap --> Svc
    Svc --> RP --> Log --> CEP
    Svc --> LocalPool --> Log
    Svc --> CimCluster --> CEP
    Svc --> NodeCim --> N2
    Svc --> NodeWinRM --> N1
```

**Transport hierarchy**

| Pool / Session | What uses it |
|---|---|
| `_pool` (WinRM RunspacePool) | Mutations needing a PS session on a cluster node: `Get-HealthFault`, `Get-NetIntent`, ATC, ECE, Arc, `GetClusterInfoAsync`, `GetSolutionUpdatesAsync` |
| `_cimSession` (C# CIM to cluster) | Reads: cluster nodes, roles, networks, CSVs, storage pools, virtual disks, physical disks |
| `_localPool` (local RSAT PS) | Hyper-V ops (Get-VM, Start/Stop/Etc, snapshots, VHD resize, CPU/memory config) and FailoverClusters mutations (Move-ClusterSharedVolume, Start/Stop-ClusterGroup, Suspend/Resume-ClusterNode, Move-ClusterVirtualMachineRole) |
| `_nodeCimSessionCache` (CIM per node) | `FetchNodeStats` (Win32_OperatingSystem, Win32_Processor, StdRegProv), VM step 2 (Get-VM -CimSession) |
| `GetOrCreateCachedRunspace` (WinRM per-node) | `GetNetworkAdaptersAsync` — NetAdapter module is only on cluster nodes |

**Why direct per-node WinRM is now rare**  
DCOM/RPC from a non-cluster machine has much higher per-call overhead than a persistent WinRM
session (`GetClusterInfoAsync` showed 14–34s with `-Cluster` vs <500ms via WinRM pool).
CIM sessions reuse a single TCP connection and bypass `wsmprovhost.exe` entirely.
Local RSAT PS with `-CimSession`/`-Cluster` uses the CIM transport internally for queries
and the existing WinRM session for mutations — no extra process on cluster nodes.

**Cache TTL: 2 minutes**  
Mutating actions (Start VM, Drain Node, etc.) call `Cache.Evict(clusterName)` after success so the next load always fetches fresh data.

---

## 5. Service Dependency Map

DI lifetime: **Singleton** = one instance for app lifetime; **Scoped** = one per HTTP request/circuit.

```mermaid
graph LR
    subgraph Singleton
        Pool["ClusterConnectionPool"]
        Cache["ClusterPageCache"]
        PsLog["PsCallLog"]
        Rbac["RbacService"]
        Registry["ClusterRegistry"]
        AppSettings["AppSettingsService"]
        Snapshot["SnapshotService"]
        Collector["CollectorTypeScheduler"]
        Alerts["AlertEngine"]
        Maintenance["MaintenanceWindowService"]
    end

    subgraph BackgroundServices
        Poller["ClusterPollerService\n(BackgroundService)"]
    end

    subgraph AlertSenders
        Composite["CompositeAlertSender"]
        Smtp["SmtpEmailSender"]
        Teams["TeamsWebhookSender"]
    end

    subgraph Scoped
        HyperV["WebHyperVService\n(created by Pool)"]
        ArmSvc["AzureArmService"]
        AuditLog["AuditLogService"]
        CurrentUser["CurrentUserService"]
    end

    subgraph Infrastructure
        DbCtx["IDbContextFactory\nAppDbContext\n(EF Core)"]
        DP["IDataProtector\n(DPAPI)"]
        TokenAcq["ITokenAcquisition\n(MSAL - OBO mode)"]
        HealthCheck["DatabaseHealthCheck"]
    end

    Pool --> HyperV
    Pool --> PsLog
    HyperV --> PsLog
    Rbac --> DbCtx
    Registry --> DbCtx
    AuditLog --> DbCtx
    AppSettings --> DbCtx
    AppSettings --> DP
    ArmSvc --> TokenAcq
    ArmSvc --> AppSettings
    CurrentUser --> AuthState["AuthenticationStateProvider\n(bUnit fake in tests)"]

    Poller --> Pool
    Poller --> Snapshot
    Poller --> Collector
    Poller --> Alerts
    Poller --> DbCtx
    Snapshot --> DbCtx
    Alerts --> Maintenance
    Alerts --> Composite
    Composite --> Smtp
    Composite --> Teams
    Maintenance --> DbCtx
    HealthCheck --> DbCtx

    Pool -.->|"EvictAsync on cluster delete"| Cache
    Rbac -.->|"RefreshAsync after RBAC mutations"| DbCtx
```

---

## 6. Deployment Topology

```mermaid
graph TB
    subgraph DevMachine["Developer machine"]
        VS["VS Code / Visual Studio"]
        PS1["Deploy-ToIIS.ps1\ndotnet publish → robocopy\n→ iisreset AZLManagementPool"]
    end

    subgraph AppServer["azlmgmt.yourdomain.com (Windows Server)"]
        direction TB
        IIS1["IIS Site: AZLManagement\nPort 443 (HTTPS)\nEntra ID OIDC auth\nApp pool: AZLManagementPool\nIdentity: DOMAIN\\azlmgmt-svc$"]
        IIS2["IIS Site: AZLManagementWinAuth\nPort 80 or 443\nWindows Authentication\nApp pool: AZLManagementWinPool\nIdentity: DOMAIN\\azlmgmt-svc$\nASPNETCORE_ENVIRONMENT=WinAuth"]
        AppFiles["C:\\apps\\azlmgmt\\\n(app binaries — replaced on deploy)"]
        DataFiles["C:\\apps\\azlmgmt-data\\\napp.db (SQLite)\ndp-keys\\ (DPAPI ring)\n(survives deploys)"]
        IIS1 --> AppFiles
        IIS2 --> AppFiles
        AppFiles --> DataFiles
    end

    subgraph AD["Active Directory — jase.org"]
        gMSA2["DOMAIN\\azlmgmt-svc$\n(Group Managed Service Account)\nAuto-rotating password\nHTTP SPN registered"]
        ADGroups["AD Security Groups\n(synced to Entra ID)"]
    end

    subgraph AzureCloud["Azure / Entra ID"]
        EntraApp["App Registration\nClient ID: 036fce98\nOIDC redirect URIs\ngroupMembershipClaims: SecurityGroup"]
        EntraGroups["Entra Security Groups\nHciRead / HciOperate / HciAdmin\nObject IDs in appsettings.json"]
    end

    VS --> PS1 --> AppServer
    gMSA2 -.->|"assigned to both app pools"| AppServer
    ADGroups -.->|"synced"| EntraGroups
    EntraApp -.->|"group OIDs in token claims"| EntraGroups
    AppServer -->|"OIDC sign-in"| EntraApp

    subgraph Cluster["Azure Local Cluster"]
        HCI["AZ-NUC-CL01.jase.org\nWinRM :5985"]
        Nodes["Cluster Nodes\nWinRM :5985 each"]
    end

    AppServer -->|"Kerberos (gMSA ticket)"| HCI
    AppServer -->|"Kerberos (gMSA ticket)"| Nodes
```

**Two IIS sites, one app binary**

| Site | Hostname | Auth | Use case |
|---|---|---|---|
| AZLManagement | `azlmgmt.yourdomain.com` | Entra ID OIDC | External / non-domain browsers, cloud users |
| AZLManagementWinAuth | `azlmgmt-win.yourdomain.com` | Windows Auth (Kerberos/NTLM) | Domain-joined browsers — single sign-on, no login prompt |

Both sites run the same published binary in `C:\apps\azlmgmt\`. The `ASPNETCORE_ENVIRONMENT=WinAuth` environment variable on the WinAuth app pool causes `appsettings.WinAuth.json` to be loaded, switching `Authentication:Scheme` to `Windows` and disabling all Entra ID MSAL registration.

---

*Last updated: March 2026*
