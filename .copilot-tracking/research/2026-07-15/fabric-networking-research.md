# Microsoft Fabric Networking Options — Explainer

## What "networking" means in Fabric

Microsoft Fabric is a **SaaS platform**. Unlike PaaS services (where you deploy compute into
your own virtual network), Fabric runs inside Microsoft's cloud on the Microsoft backbone
network. Every request is authenticated with **Microsoft Entra ID**, and all traffic between
Fabric experiences travels over the internal Microsoft network. Traffic between your clients
(browser, SSMS, etc.) and Fabric is encrypted with **TLS 1.2+** (negotiating to 1.3 when possible).

Because Fabric is SaaS, its networking controls split into two directions:

- **Inbound** — controlling and securing traffic **coming into** Fabric (users/clients reaching your tenant).
- **Outbound** — controlling how Fabric **reaches out** to your data sources (which often sit behind firewalls or in private networks).

---

## Inbound networking — securing access *into* Fabric

### 1. Microsoft Entra Conditional Access

**What it is:** Identity-based access policies applied at authentication time. Built on the
Zero Trust model — identity is the security perimeter, not the network.

**What you can enforce:**
- Allow-list of IPs for inbound connectivity to Fabric
- Require Multifactor Authentication (MFA)
- Restrict by country/region, device compliance, or device type
- Block or limit access from risky locations, devices, or networks

**Requirements / caveats:**
- Requires **Microsoft Entra ID P1** licenses (often already owned via Microsoft 365).
- Policies apply broadly to Fabric *and* related Azure services (Power BI, Azure Data Explorer,
  Azure SQL Database, Azure Storage) — can be too coarse-grained for some customers.
- Fabric does not support account keys or SQL authentication (username/password) — Entra only.

**When to use it:** The default, lightweight choice for most organizations that want to
restrict who/where/how users authenticate without re-architecting their network. Good when you
want MFA, device compliance, or IP-range restrictions but don't need to fully remove Fabric from
the public internet.

---

### 2. Private Links

**What it is:** Uses Azure Private Link + private endpoints to assign Fabric a **private IP
address** from your virtual network. Traffic tunnels from your VNet subnet into Fabric over the
Microsoft backbone, and Fabric is **no longer reachable over the public internet**. All
communication — including viewing a Power BI report in a browser or connecting via SSMS to a
`...datawarehouse.fabric.microsoft.com` string — must go through the private network.

**Two levels:**

| Level | Scope | Use when |
|---|---|---|
| **Tenant-level private links** | Applies network policy to the **entire tenant** — every workspace and resource | You want blanket, organization-wide isolation from the public internet |
| **Workspace-level private links** | Maps a **specific workspace** to a specific VNet (1:1 private link service per workspace) | You want to isolate only sensitive workspaces while leaving others public, without tenant-wide changes |

**Key details:**
- One private link service maps to one workspace, but a workspace can have multiple private
  endpoints (multiple VNets can reach it), and one VNet can reach multiple workspaces.
- You can restrict a workspace's inbound public access with or without a private link. Restricting
  without a link makes the workspace unreachable from all networks (until an admin changes the
  inbound rule via the communication policy API).
- Workspace FQDNs are constructed from the workspace ID, e.g.
  `https://{workspaceId}.z{xy}.onelake.fabric.microsoft.com`.
- Enabling tenant Private Link and running the first Spark job triggers creation of a **managed
  virtual network** (see below).

**Trade-offs / caveats:**
- **Bandwidth & latency:** All traffic routes through the private endpoint's region. Global static
  assets (CSS/HTML/images) also load from that region, so users far from the endpoint (e.g.,
  Australian users with a US endpoint) see slower load times.
- **On-premises reach:** Extend on-prem networks to Azure via ExpressRoute or site-to-site VPN.
- **Cost:** Private Link pricing plus potential ExpressRoute bandwidth increases.
- Comes with a notable list of considerations and limitations because you're closing Fabric off
  from the public internet.

**When to use it:** Regulated or high-security environments that must guarantee Fabric is only
reachable from approved corporate networks. Choose **workspace-level** when only some workspaces
need this; **tenant-level** for organization-wide lockdown.

**Inbound decision rule of thumb:** Start with Conditional Access (identity-based, simpler). Move
to Private Links when you must remove public internet access entirely.

---

## Outbound networking — Fabric reaching *out* to data sources

### 3. Trusted workspace access

**What it is:** Lets Fabric workspaces securely reach **firewall-enabled ADLS Gen2 storage
accounts** using the workspace's **workspace identity**, without opening the storage account to the
public. You create a **resource instance rule** on the storage account that trusts a specific
Fabric workspace ID.

**How it's used across Fabric:** OneLake shortcuts, pipelines, the T-SQL `COPY` statement
(Warehouse ingestion), import-mode semantic models, and AzCopy loads.

**Requirements / caveats:**
- Requires an **F SKU capacity** (not Trial, not My workspaces); needs a **workspace identity**
  with Contributor access.
- The authenticating principal needs Storage Blob Data Reader/Contributor/Owner (or Delegator) RBAC.
- Resource instance rules must be created via **ARM template or PowerShell** — not the Azure portal UI.
- Max 200 resource instance rules; not compatible with cross-tenant requests.
- For securing storage access **from Spark**, use managed private endpoints instead.

**When to use it:** You need Fabric (shortcuts, pipelines, warehouse COPY, import models) to read
from firewalled ADLS Gen2 without deploying private endpoints, and you want to scope access to
specific workspaces.

---

### 4. Managed private endpoints

**What it is:** Private endpoints that **Fabric creates and manages** to reach data sources behind
a firewall (Azure SQL Database, Azure Storage, and many more) — without exposing them publicly or
building complex networking yourself. A workspace admin creates them from workspace settings by
supplying the target resource ID, subresource, and a justification.

**Supported workloads:** Fabric Data Engineering (Spark/Python notebooks, lakehouses, Spark job
definitions) and Eventstream (preview).

**Requirements / caveats:**
- Supported in Fabric **Trial** and all **F SKU** capacities.
- Requires Fabric Data Engineering (Spark) support in both the tenant home region and the capacity
  region. Creating one provisions a **managed virtual network** for the workspace.
- OneLake shortcuts don't yet support ADLS Gen2 / Blob connections via managed private endpoints.
- FQDN-based endpoints via Private Link Service are REST-API-only (not in the UX).
- Wait ~15 minutes after deleting one before recreating to the same resource.

**When to use it:** Spark/Data Engineering (or Eventstream) workloads need to reach private,
firewalled data sources like Azure SQL DB or Storage over a private connection.

---

### 5. Managed virtual networks

**What it is:** A dedicated virtual network that **Fabric creates and manages per workspace** to
give **network isolation for Spark workloads**. Spark clusters run in this dedicated network rather
than the shared VNet. It's the enabler for managed private endpoints and Private Link support for
Data Engineering / Data Science (Spark) items.

**How it gets provisioned (automatically):**
- When a **managed private endpoint** is added to the workspace (outbound), **or**
- When **Private Link** is enabled and the first Spark job / lakehouse operation runs (inbound).

**Trade-offs / caveats:**
- Once provisioned, **starter pools are disabled** (those are pre-warmed clusters in a shared VNet).
  Spark jobs then run on on-demand custom pools inside the managed VNet, adding roughly **3–5
  minutes** of session startup time.
- Not supported in Switzerland West and West Central US regions.

**When to use it:** You don't configure it directly — it's the isolation layer that turns on
automatically once you adopt managed private endpoints or Private Link with Spark. Understand it
mainly for the startup-latency and region trade-offs.

---

### 6. Data gateways

Bridges for connecting Fabric to sources that aren't publicly reachable:

- **On-premises data gateway** — Installed on a server inside your network; acts as a secure bridge
  so Fabric can reach on-prem sources without opening firewall ports.
- **Virtual network (VNet) data gateway** — Connects Microsoft cloud services to Azure data services
  **inside a VNet**, with no on-premises gateway software required.

**When to use it:** On-premises gateway for on-prem/private data sources; VNet gateway for Azure
data services locked inside a VNet.

---

### 7. Azure service tags

**What it is:** Use the `PowerBI`/Fabric **service tag** in Azure NSG or firewall rules to allow
traffic to/from Fabric without deploying data gateways. Lets you ingest from sources deployed in an
Azure VNet (Azure SQL VM, Azure SQL Managed Instance, REST APIs) or control VNet/Azure Firewall
egress.

**Example:** Allow a user on an Azure VM to reach Fabric SQL connection strings from SSMS while
blocking other public internet access.

**When to use it:** Azure-hosted data sources (SQL MI, SQL on VMs, REST APIs) that support service
tags, where you want secure connectivity without a gateway.

---

### 8. IP allowlists

**What it is:** For data **not in Azure**, allow-list Fabric's IP ranges on your network so traffic
can flow to/from Fabric. Fabric IPs come from the **Service Tags on-premises** JSON file (also
available via REST API, PowerShell, Azure CLI). Enables no-copy access (shortcuts, Lakehouse SQL
analytics endpoint, Direct Lake) from on-prem sources.

**When to use it:** On-premises or non-Azure data sources that **don't support service tags** and
where you want to connect without copying data into OneLake.

---

### 9. Connect to OneLake from an existing service

**What it is:** Reach Fabric/OneLake from existing Azure PaaS. For Synapse and Azure Data Factory,
use **Azure Integration Runtime** or **ADF managed virtual network + private endpoint**. Other
engines (Mapping Data Flows, Synapse/Databricks Spark, HDInsight) can use the **OneLake APIs**.

**When to use it:** You already run Synapse, ADF, Databricks, or HDInsight and want them to read/write
OneLake data securely from their existing network setup.

---

## Quick decision summary

| Goal | Feature |
|---|---|
| Restrict who/where/how users sign in (MFA, IP, device) | **Conditional Access** |
| Remove Fabric from the public internet entirely | **Private Links** (tenant or workspace level) |
| Isolate only sensitive workspaces | **Workspace-level private links** |
| Read firewalled ADLS Gen2 via shortcuts/pipelines/COPY/import | **Trusted workspace access** |
| Spark/Eventstream reaching private data sources (SQL DB, Storage) | **Managed private endpoints** |
| Network isolation for Spark clusters (auto-provisioned) | **Managed virtual networks** |
| Reach on-prem or in-VNet data sources | **Data gateways** (on-prem / VNet) |
| Allow Azure sources (SQL MI, SQL VM, REST) without a gateway | **Service tags** |
| Allow non-Azure/on-prem sources by IP | **IP allowlists** |
| Connect existing Synapse/ADF/Databricks to OneLake | **OneLake APIs / IR / ADF managed VNet** |

## Evidence log
- Security in Microsoft Fabric — https://learn.microsoft.com/en-us/fabric/security/security-overview
- Protect inbound traffic — https://learn.microsoft.com/en-us/fabric/security/protect-inbound-traffic
- Managed virtual networks overview — https://learn.microsoft.com/en-us/fabric/security/security-managed-vnets-fabric-overview
- Managed private endpoints overview — https://learn.microsoft.com/en-us/fabric/security/security-managed-private-endpoints-overview
- Trusted workspace access — https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access
- Workspace-level private links overview — https://learn.microsoft.com/en-us/fabric/security/security-workspace-level-private-links-overview
