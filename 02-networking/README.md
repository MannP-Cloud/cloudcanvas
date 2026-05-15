# 02 · Virtual Networking

If Identity is the foundation of *who* can access Azure, Networking is the foundation of *how* resources talk to each other - and more importantly, how they don't talk to things they shouldn't.

This is the domain that separates people who have read about Azure from people who have actually built on it. You can memorize CIDR notation and NSG rule priorities all day, but until you've watched your own policy block a Network Watcher deployment, it doesn't fully click.

**AZ-104 exam weight:** 15–20%  
**Environment:** hub-vnet + spoke-vnet + NSGs + Bastion + Private DNS  
**Case study context:** Contoso Ltd has one non-negotiable security requirement - no VM should ever have a public IP address. All access must go through a controlled gateway. All resources must communicate privately. This entire domain exists to make that happen.

---

## Why hub-spoke topology?

Before anything else - why build two VNets instead of one?

In a real company, you don't put everything in one flat network. You separate concerns:

- **The hub** is where your shared infrastructure lives - the things every workload needs but nobody should own individually. Bastion for secure access. DNS for name resolution. In larger environments, a firewall.
- **The spokes** are where your workloads live - isolated from each other, connected to the hub, never talking directly to other spokes without going through the hub first.

This gives you security isolation, clean billing separation, and a network you can actually reason about when something goes wrong at 2am.

For Contoso, the hub hosts Bastion and DNS. The spoke hosts the storage private endpoint. When we add VMs in Domain 3, they'll live in the hub subnets - connected to shared services, isolated from the data tier.

---

## What I built

| # | Resource | Purpose | Screenshot |
|---|---|---|---|
| 1 | hub-vnet (10.0.0.0/16) | Central hub with 3 subnets | [01-hub-vnet.png](screenshots/01-hub-vnet.png) |
| 2 | spoke-vnet (10.1.0.0/16) | Data tier spoke with subnet-data | [02-spoke-vnet.png](screenshots/02-spoke-vnet.png) |
| 3 | VNet Peering (both directions) | Connects hub and spoke privately | [03a-vnet-peering-hub.png](screenshots/03a-vnet-peering-hub.png) · [03b-vnet-peering-spoke.png](screenshots/03b-vnet-peering-spoke.png) |
| 4 | NSG for subnet-admin | Controls RDP access to Windows VM subnet | [04a-nsg-subnet-admin.png](screenshots/04a-nsg-subnet-admin.png) |
| 5 | NSG for subnet-dev | Controls SSH access to Linux VM subnet | [04b-nsg-subnet-dev.png](screenshots/04b-nsg-subnet-dev.png) |
| 6 | Azure Bastion | Secure browser-based RDP/SSH - no public IPs on VMs | [05a-bastion-deploying.png](screenshots/05a-bastion-deploying.png) · [05b-bastion-complete.png](screenshots/05b-bastion-complete.png) |
| 7 | Private DNS Zone (cloudcanvas.internal) | Internal name resolution for VNet resources | [06-private-dns.png](screenshots/06-private-dns.png) |
| 8 | Network Watcher | Network diagnostics and monitoring | [07-network-watcher.png](screenshots/07-network-watcher.png) |

---

## hub-vnet - The network backbone

**Address space:** 10.0.0.0/16 - 65,536 addresses. More than we'll ever use, but planning generously avoids painful address exhaustion later.

**Three subnets:**

```
AzureBastionSubnet    10.0.0.0/26    64 addresses
subnet-admin          10.0.1.0/24    256 addresses  → Windows VM (Domain 3)
subnet-dev            10.0.2.0/24    256 addresses  → Linux VM (Domain 3)
```

**Why /26 for Bastion?** Microsoft requires AzureBastionSubnet to be at least /26 - that's 64 addresses minimum. The name must be spelled exactly `AzureBastionSubnet` with that exact casing or Bastion won't deploy into it.

**Why separate subnets for admin and dev?** Because NSGs are applied at the subnet level. If both VMs were in the same subnet, you couldn't write different firewall rules for them. Separating them gives you granular control - the Windows admin VM allows RDP from Bastion, the Linux dev VM allows SSH from Bastion, and never the other way around.

**Azure reserves 5 IPs in every subnet:** The first 4 addresses and the last one are reserved by Azure for internal use. A /24 gives you 256 total addresses but only 251 usable - that's what you see in the Portal.

---

## spoke-vnet - The data tier

**Address space:** 10.1.0.0/16

**One subnet:**
```
subnet-data    10.1.0.0/24    256 addresses  → Storage Private Endpoint (Domain 4)
```

**Why a separate VNet for storage?** In enterprise architecture, you isolate your data tier from your compute tier. If something goes wrong in the compute VNet - a misconfigured NSG, a compromised VM - it doesn't automatically have access to storage. The spoke boundary forces all traffic to go through the hub, where you can inspect and control it.

In Domain 4, we'll create a Private Endpoint for the storage account that lives in subnet-data. This gives the storage account a private IP inside the VNet, meaning traffic from VMs to storage never touches the public internet.

---

## VNet Peering - Connecting hub and spoke

Peering connects two VNets so their resources can communicate using private IPs. Traffic stays entirely on Microsoft's backbone network - it never goes to the public internet and it never leaves the Azure region.

**Two peering links were created simultaneously:**
- `hub-to-spoke` - visible from spoke-vnet's peering list
- `spoke-to-hub` - visible from hub-vnet's peering list

Both show **Fully Synchronized** and **Connected** status.

**The non-transitive rule - critical for the exam:**

Peering is not transitive. If you have:
```
VNet A ←→ VNet B ←→ VNet C
```
VNet A cannot reach VNet C through VNet B. Each pair needs its own peering. In our environment this isn't an issue because we only have two VNets, but in enterprise hub-spoke with 10+ spokes, this matters - spokes cannot talk to each other directly, only through the hub.

**Why not just put everything in one VNet?** You could. But peering gives you organizational and security boundaries. Different teams can own different VNets, with different policies, different access controls, and separate blast radius if something goes wrong.

---

## NSGs - The subnet firewall

Network Security Groups are stateful firewalls. You define rules, and Azure tracks connection state - so if you allow inbound traffic, the response traffic is automatically allowed outbound without needing a separate rule.

**Rules are evaluated by priority number - lower number = higher priority.**

### nsg-subnet-admin (Windows VM subnet)

| Priority | Name | Port | Source | Action |
|---|---|---|---|---|
| 100 | Allow-RDP-From-Bastion | 3389 | 10.0.0.0/26 | Allow |
| 200 | Deny-RDP-Internet | 3389 | Any | Deny |
| 65000 | AllowVnetInBound | Any | VirtualNetwork | Allow |
| 65500 | DenyAllInBound | Any | Any | Deny |

**Why source IP 10.0.0.0/26?** That's the AzureBastionSubnet address range. By specifying this exact range, only traffic originating from the Bastion subnet can RDP into the Windows VM. If someone tries to RDP directly from the internet, it hits the Deny rule at priority 200 first.

**The default rules (65000, 65500)** are created by Azure automatically on every NSG. You can't delete them but you can override them with lower priority numbers.

### nsg-subnet-dev (Linux VM subnet)

Same pattern, but for SSH (port 22):

| Priority | Name | Port | Source | Action |
|---|---|---|---|---|
| 100 | Allow-SSH-From-Bastion | 22 | 10.0.0.0/26 | Allow |
| 200 | Deny-SSH-Internet | 22 | Any | Deny |

**In a real company:** NSGs are one of the first things a security team audits. Open RDP or SSH to the internet (0.0.0.0/0) is a critical finding that gets escalated immediately. Every public-facing VM with port 3389 or 22 open is actively being probed by automated scanners within minutes of deployment.

---

## Azure Bastion - The secure gateway

Bastion is how you connect to VMs in this environment. No VM has a public IP. No RDP port is open to the internet. Instead, you open a browser, navigate to the Azure Portal, click Connect on the VM, and a browser-based RDP or SSH session opens - all tunneled through the Bastion service over HTTPS port 443.

**What we deployed:**
- Name: `cloudcanvas-bastion`
- Tier: Basic
- Subnet: AzureBastionSubnet (10.0.0.0/26) in hub-vnet
- Public IP: `bastion-pip` (this is Bastion's own public IP - the VMs themselves still have no public IP)

**Why does Bastion need a public IP if the VMs don't?** Bastion is the controlled entry point. It has one public IP that you can monitor, restrict, and audit. All access to all VMs goes through this single point. Compare this to giving every VM a public IP - you'd have dozens of attack surfaces to manage.

**Cost note:** Azure Bastion Basic tier costs approximately $0.19/hour while provisioned. For a lab environment, delete it when not in use and redeploy when needed. In production it runs 24/7.

**Basic vs Standard SKU:**
- Basic: Browser-based RDP/SSH only
- Standard: Native RDP client support, shareable links, IP-based connections, file transfers

For AZ-104 purposes, Basic covers everything you need to know.

---

## Private DNS Zone - Internal name resolution

**Zone name:** `cloudcanvas.internal`  
**Linked to:** hub-vnet with auto-registration enabled

Without this, VMs in the VNet can only refer to each other by IP address. With it, every VM that gets deployed into hub-vnet automatically registers its hostname - so you can SSH to `dexter-vm.cloudcanvas.internal` instead of remembering `10.0.1.4`.

**Auto-registration** means Azure automatically creates and removes DNS records as VMs are deployed and deleted. No manual DNS management needed.

**Why this matters for Private Endpoints (Domain 4):**
When you create a Private Endpoint for a storage account, the storage account's public DNS name (like `cloudcanvasstore.blob.core.windows.net`) needs to resolve to the private IP inside your VNet instead of the public IP. This requires a separate Private DNS Zone using the `privatelink.blob.core.windows.net` naming convention. We'll create that in Domain 4.

**The Portal glitch we hit:** Creating the VNet link through the Portal returned an undefined error. The fix was using Azure CLI with the `--tags Project=CloudCanvas` flag. This is a real-world lesson - the Portal doesn't always expose all options, and CLI often works when the UI fails. Knowing both is what makes you effective as an Azure admin.

---

## Network Watcher - Diagnostics

Network Watcher is a regional service that provides diagnostic tools for your Azure network. We created it for Canada Central.

**Key tools you'll use in Domain 3 when VMs exist:**

**IP Flow Verify** - answers the question "is this NSG blocking my traffic?" You specify a source IP, destination IP, port, and direction, and it tells you exactly which NSG rule is allowing or denying the flow. Saves hours of manually reading through rule lists.

**Connection Troubleshoot** - tests end-to-end connectivity between two endpoints. Useful when you can't figure out why a VM can't reach a resource.

**NSG Flow Logs** - logs every connection attempt that hits your NSGs to a storage account. Useful for security auditing and understanding traffic patterns.

**The policy lesson we learned:** Network Watcher couldn't be created through the Portal because our "Require Project Tag" policy blocked it. This is a real-world scenario - blanket tag policies need exemptions for system-managed resources that don't support tags. In production, you'd use a policy exemption or modify the policy scope to exclude the NetworkWatcherRG resource group.

---

## The network topology - fully built

```
Subscription
└── rg-network (Canada Central)
    │
    ├── hub-vnet (10.0.0.0/16)
    │   ├── AzureBastionSubnet (10.0.0.0/26)
    │   │   └── cloudcanvas-bastion ← entry point for all VM access
    │   ├── subnet-admin (10.0.1.0/24)
    │   │   └── nsg-subnet-admin (Allow RDP from Bastion only)
    │   │       └── Windows VM [deployed in Domain 3]
    │   └── subnet-dev (10.0.2.0/24)
    │       └── nsg-subnet-dev (Allow SSH from Bastion only)
    │           └── Linux VM [deployed in Domain 3]
    │
    ├── spoke-vnet (10.1.0.0/16) ←→ VNet Peering ←→ hub-vnet
    │   └── subnet-data (10.1.0.0/24)
    │       └── Storage Private Endpoint [deployed in Domain 4]
    │
    ├── cloudcanvas.internal (Private DNS Zone)
    │   └── link-to-hub-vnet (auto-registration enabled)
    │
    └── NetworkWatcher_canadacentral
```

---

## Key exam topics from this section

**VNet and subnets:**
- Address spaces cannot overlap between peered VNets
- Azure reserves 5 IPs per subnet (first 4 + last)
- AzureBastionSubnet must be minimum /26 and named exactly right
- Subnets in the same VNet can communicate by default

**NSGs:**
- Lower priority number = evaluated first
- Stateful - return traffic is automatically allowed
- Can be applied to subnet OR NIC - if both, both are evaluated
- Default rules: 65000 AllowVnetInBound, 65500 DenyAllInBound
- Cannot delete default rules but can override with lower priority

**VNet Peering:**
- Non-transitive - A→B and B→C does not mean A→C
- Both directions must exist (though Portal creates both simultaneously)
- Global peering works across Azure regions
- Address spaces cannot overlap

**Azure Bastion:**
- Requires AzureBastionSubnet (/26 minimum, exact name)
- Basic SKU = browser only; Standard = native client support
- VMs need zero public IPs when using Bastion
- Connects over HTTPS port 443

**Private DNS:**
- Auto-registration handles VM hostname records automatically
- Private Endpoints require privatelink.* zone naming
- VNet link required for each VNet that needs resolution

---

## Resources

- [Microsoft Learn - AZ-104 Networking path](https://learn.microsoft.com/en-us/training/paths/az-104-manage-virtual-networks/)
- [NSG default rules reference](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview#default-security-rules)
- [Azure Bastion FAQ](https://learn.microsoft.com/en-us/azure/bastion/bastion-faq)
- [VNet Peering overview](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)

---

*Built by Mann Patel as part of the CloudCanvas AZ-104 lab series.*  
*[← Back to CloudCanvas](https://mannp-cloud.github.io/cloudcanvas)*
