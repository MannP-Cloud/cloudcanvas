# 04 · Storage

Storage is where data lives - and how you protect it, organise it, and control who can access it is one of the most practically important skills an Azure admin needs. Every workload generates data. Every workload needs somewhere to put it. Getting storage wrong means either paying too much (keeping everything in hot tier forever), exposing data you shouldn't (misconfigured access levels), or losing data you can't recover (no soft delete, no backups).

This domain is deceptively broad. The exam covers blob tiers, lifecycle policies, SAS tokens, file shares, firewalls, and private endpoints - and expects you to know not just what they are but when to use each one and why.

**AZ-104 exam weight:** 15–20%  
**Environment:** cloudcanvasstore storage account with blob containers, Azure Files, lifecycle policy, SAS token, storage firewall, and private endpoint  
**Case study context:** Contoso Ltd needs structured blob storage for raw data ingestion, a public-facing asset container, a cold archive for compliance data, a shared file server replacement, and enterprise-grade network security to ensure storage is never accessible from the public internet.

---

## What I built

| # | Resource | Purpose | Screenshot |
|---|---|---|---|
| 1 | cloudcanvasstore (Storage Account) | Central storage for all Contoso workloads | [01-storage-account.png](screenshots/01-storage-account.png) |
| 2 | Blob containers (raw-data, public-assets, archives) | Organised blob storage with different access levels | [02-blob-containers.png](screenshots/02-blob-containers.png) |
| 3 | Access tiers (Cool on archives blob) | Cost-optimised storage based on access frequency | [03-access-tiers.png](screenshots/03-access-tiers.png) |
| 4 | Lifecycle policy (archive-old-data) | Automatic tier transitions and deletion | [04-lifecycle-policy.png](screenshots/04-lifecycle-policy.png) |
| 5 | SAS token | Time-limited secure access without exposing account keys | [05-sas-token.png](screenshots/05-sas-token.png) |
| 6 | Azure Files (contoso-share 100 GiB) | Managed SMB file share replacing on-premises file server | [06-azure-files.png](screenshots/06-azure-files.png) |
| 7 | Storage firewall (spoke-vnet/subnet-data) | Restricts storage access to VNet only | [07-storage-firewall.png](screenshots/07-storage-firewall.png) |
| 8 | Private Endpoint (blob) | Private IP for storage inside the VNet | [08-private-endpoint.png](screenshots/08-private-endpoint.png) |

---

## The Storage Account - cloudcanvasstore

**Configuration:**
- Kind: StorageV2 (General Purpose v2) - the current standard
- Performance: Standard
- Redundancy: LRS (Locally Redundant Storage)
- Soft delete: 7 days for blobs and containers
- Versioning: Enabled
- Region: Canada Central

**Why StorageV2?** It supports all storage services - blobs, files, queues, tables - and all access tiers. Older account types like BlobStorage or StorageV1 have limitations. Always use StorageV2 unless you have a specific reason not to.

**Why LRS for a lab?** LRS keeps 3 copies of data within a single datacenter. It's the cheapest redundancy option. For production, Contoso would use GRS (Geo-Redundant Storage) which replicates to a second Azure region - essential for disaster recovery. For a lab environment spending credits, LRS is the right call.

**Redundancy options you need to know:**
```
LRS  → 3 copies, 1 datacenter, 1 region
ZRS  → 3 copies, 3 availability zones, 1 region
GRS  → LRS + async replication to secondary region
GZRS → ZRS + async replication to secondary region
```

**Why soft delete?** Accidents happen. A developer accidentally deletes a blob container with production data inside - without soft delete, it's gone forever. With 7-day soft delete, you have a week to restore it. This is a no-cost safety net that should always be enabled.

**Why versioning?** Every time a blob is overwritten, the previous version is retained. If someone overwrites a file with corrupt data, you can restore the previous version. Essential for any storage account holding important documents or application data.

---

## Blob Containers - Three access levels

```
raw-data        → Private   (internal data ingestion, no public access)
public-assets   → Blob      (anonymous read of individual blobs)
archives        → Private   (compliance data, no public access)
$logs           → Private   (auto-created by Azure for diagnostic logs)
```

**Why three different containers instead of one?**
Because access control is at the container level for anonymous access. You can't make one blob public and another private within the same container using access levels - you'd need SAS tokens for that. Separating by access requirement is cleaner and easier to audit.

**The three access levels:**
- **Private** - No anonymous access. Requires authentication for everything.
- **Blob** - Anonymous read access to individual blobs only. You need the full URL.
- **Container** - Anonymous read access to the container listing AND individual blobs. Anyone can browse what's in the container.

**Why Blob and not Container for public-assets?** Because we don't want anonymous users to be able to list everything in the container. They can access a specific asset if they know the URL, but they can't browse the full list. Slightly more secure.

**Note:** Anonymous access had to be explicitly enabled on the storage account first - it's disabled by default on new accounts, which is the correct secure default. We enabled it only because we have a legitimate public-assets container that needs it.

---

## Access Tiers - Hot, Cool, Archive

Azure charges you differently based on how often data is accessed:

| Tier | Storage cost | Access cost | Minimum storage |
|---|---|---|---|
| Hot | Highest | Lowest | None |
| Cool | Lower | Higher | 30 days |
| Archive | Lowest | Highest + rehydration | 180 days |

**The rule is simple:** data you access frequently goes in Hot. Data you access occasionally goes in Cool. Data you almost never access but must keep goes in Archive.

**Archive is special:** blobs in Archive cannot be read directly. They must first be *rehydrated* - moved back to Hot or Cool. Standard rehydration takes up to 15 hours. High-priority rehydration takes up to 1 hour but costs more. Plan accordingly.

**Early deletion fees:** if you delete a Cool blob before 30 days, you're charged for the full 30. Same for Archive at 180 days. Don't move data to these tiers unless you're sure it'll stay there.

**In Contoso's environment:**
- raw-data stays Hot - actively ingested data
- public-assets stays Hot - frequently accessed by users
- archives blobs set to Cool - compliance data accessed rarely

---

## Lifecycle Management Policy - archive-old-data

Instead of manually moving blobs between tiers, the lifecycle policy does it automatically based on the blob's last modified date.

**Rule configured:**
```
After 30 days  → Move to Cool storage
After 90 days  → Move to Archive storage  
After 365 days → Delete the blob
```

**Why this matters:** Without lifecycle policies, organisations accumulate years of hot-tier data that should have been archived. A storage account with 10TB of data that should be in Archive but is sitting in Hot can cost thousands of dollars more per month than necessary. This is one of the most common quick wins in Azure FinOps reviews.

**Policies run daily** - Azure evaluates your rules every day and moves or deletes blobs that meet the conditions. The change can take up to 24 hours to go into effect after you create or modify a rule.

**Filter options:** You can apply lifecycle rules to specific containers by prefix (e.g. only blobs in `raw-data/`) or by blob index tags. Our rule applies to all block blobs across the account.

---

## SAS Tokens - Shared Access Signatures

A SAS token is a URI that grants time-limited, permission-limited access to a storage resource without exposing the account key.

**SAS token configured:**
- Service: Blob
- Resource types: Container + Object
- Permissions: Read, List
- Protocol: HTTPS only
- Expiry: 24 hours

**Why not just use the account key?** The account key gives full access to everything in the storage account forever. Sharing it with a user or application means they can read, write, delete anything. A SAS token is scoped - Read only on blobs, expires tomorrow, HTTPS only. If it leaks, the blast radius is limited and it expires automatically.

**Three types of SAS - know these for the exam:**
- **Account SAS** - Access to multiple services (blob, file, queue, table) with specified permissions
- **Service SAS** - Access to a specific service (blob only, file only) with specified permissions  
- **User delegation SAS** - Most secure. Uses Entra ID credentials instead of account key. Requires Entra ID permissions.

**Stored access policy** - You can define a named policy on a container and reference it in the SAS. This lets you revoke the SAS before it expires by deleting the policy. Without a stored access policy, a SAS cannot be revoked before its expiry time.

---

## Azure Files - contoso-share

**Configuration:**
- Name: contoso-share
- Quota: 100 GiB
- Tier: Transaction optimized
- Protocol: SMB (Server Message Block)

**Why Azure Files?** Contoso has an on-premises file server that's aging out. Azure Files gives them the same experience - map a drive letter, browse folders, save files - but fully managed in Azure. No server to maintain, no storage hardware to replace, built-in snapshots and soft delete.

**Mounting on Windows:**
```powershell
net use Z: \\cloudcanvasstore.file.core.windows.net\contoso-share /user:Azure\cloudcanvasstore <account-key>
```

**SMB requires port 445** - some ISPs and corporate firewalls block outbound port 445. This is a common gotcha. If mounting fails, check that port 445 is open from the client to the storage account.

**Tier options:**
- **Transaction optimized** - For workloads with high transaction rates. Best for general file shares.
- **Hot** - For frequently accessed files with lower transaction rates.
- **Cool** - For archives and backup. Lowest storage cost, highest access cost.
- **Premium** - SSD-backed, sub-millisecond latency. Requires FileStorage account type (not StorageV2).

**Azure File Sync** - Not deployed in this lab due to cost, but worth knowing for the exam. File Sync extends on-premises Windows file servers to Azure. Files are cached locally for fast access but stored durably in Azure. When on-premises storage fills up, older files are tiered to Azure automatically - the local file is replaced with a pointer.

---

## Storage Firewall

**Configuration:**
- Public network access: Enabled from selected virtual networks only
- Virtual network: spoke-vnet
- Subnet: subnet-data (Endpoint Status: Enabled)
- Exceptions: Allow trusted Azure services

**Why restrict to spoke-vnet only?** Contoso's storage account should never be accessible directly from the public internet. By restricting to subnet-data in spoke-vnet, only resources inside that subnet can reach the storage account over the network. A developer sitting at home cannot browse the storage account directly - they'd need to connect through the corporate VPN or Bastion first.

**The "Allow trusted Azure services" exception** is important. Services like Azure Backup, Azure Monitor, and Azure Site Recovery need to access your storage account even when the firewall is on. This checkbox allows Microsoft's trusted services to bypass the firewall while still blocking everything else.

**Service Endpoints vs Private Endpoints:**
This firewall configuration uses **Service Endpoints** - it extends the VNet's identity to the storage service, meaning traffic from subnet-data to storage stays on the Microsoft backbone. But the storage account still has a public IP address - it's just restricted to only accept traffic from subnet-data.

**Private Endpoints** go further - they give the storage account a private IP inside your VNet. The public endpoint can be completely disabled. Traffic never touches the public internet at all. That's what we built next.

---

## Private Endpoint - cloudcanvasstore-pe

**Configuration:**
- Name: cloudcanvasstore-pe
- Target sub-resource: blob
- Virtual network: spoke-vnet
- Subnet: subnet-data
- Connection state: Approved (Auto-Approved)

**How it works:** A Private Endpoint creates a Network Interface Card (NIC) inside subnet-data with a private IP address. When the Windows VM in subnet-admin wants to access blob storage, it resolves `cloudcanvasstore.blob.core.windows.net` - and with the Private DNS Zone configured correctly, that name resolves to the private IP (e.g. 10.1.0.5) instead of the public IP. Traffic stays entirely within the Microsoft network.

**Why both firewall AND private endpoint?** The storage firewall (Service Endpoint) restricts which networks can access the public endpoint. The Private Endpoint creates a completely separate private path. In enterprise environments you typically do both - the Private Endpoint handles normal traffic, and the firewall is a belt-and-suspenders control.

**The policy problem we hit:** Creating the Private Endpoint through the Portal failed because the auto-generated NIC resource couldn't accept tags through the Portal UI. The fix was using Azure CLI with the full subnet resource ID path. This is a recurring theme - complex resource deployments that create child resources (NICs, public IPs) often need CLI to properly pass tags through.

**Private DNS Zone for storage:** For the Private Endpoint to work correctly, DNS resolution must return the private IP. Azure creates a `privatelink.blob.core.windows.net` Private DNS Zone automatically when you create the endpoint through the Portal. Via CLI, you'd need to create and link this zone manually. The zone ensures that `cloudcanvasstore.blob.core.windows.net` resolves to the private IP when queried from within the VNet.

---

## Key exam topics from this section

**Storage account:**
- StorageV2 is the current standard - supports all services and tiers
- LRS/ZRS/GRS/GZRS redundancy - know the differences
- Soft delete and versioning are separate settings - both should be enabled
- Performance tiers: Standard (HDD) vs Premium (SSD)

**Blob access levels:**
- Private - no anonymous access
- Blob - anonymous read of individual blobs only
- Container - anonymous read of container listing and blobs
- Anonymous access must be enabled at account level first

**Access tiers:**
- Hot/Cool/Archive - know storage vs access cost trade-offs
- Archive requires rehydration before reading (up to 15 hours standard, 1 hour high priority)
- Early deletion fees: Cool 30 days, Archive 180 days
- Tier can be set at account level (default) or blob level (override)

**Lifecycle policies:**
- Run daily, take up to 24 hours to apply
- Conditions: last modified, last accessed, creation date
- Actions: tier to Cool, tier to Archive, delete
- Can filter by container prefix or blob index tags

**SAS tokens:**
- Account SAS, Service SAS, User Delegation SAS (most secure)
- Stored access policy enables revocation before expiry
- Never embed account keys in application code - use SAS or Managed Identity

**Azure Files:**
- SMB protocol, port 445
- Mount as network drive on Windows/Linux/macOS
- File Sync for hybrid on-premises + cloud scenarios
- Snapshots for point-in-time recovery

**Private Endpoints vs Service Endpoints:**
- Service Endpoint: extends VNet route to service, storage still has public IP
- Private Endpoint: gives storage a private IP in your VNet, public endpoint can be disabled
- Private Endpoint requires Private DNS Zone for correct name resolution

---

## Resources

- [Microsoft Learn - AZ-104 Storage path](https://learn.microsoft.com/en-us/training/paths/az-104-manage-storage/)
- [Storage redundancy overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Blob access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [SAS tokens overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Private Endpoints for storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-private-endpoints)

---

*Built by Mann Patel as part of the CloudCanvas AZ-104 lab series.*  
*[← Back to CloudCanvas](https://mannp-cloud.github.io/cloudcanvas)*
