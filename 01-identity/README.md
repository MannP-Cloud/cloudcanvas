# 01 · Identities & Governance

This is the first domain I built for CloudCanvas - the foundation everything else sits on. Before you deploy a single VM or create a storage account, you need to know *who* can do *what* and *where*. That's what this section is about.

**AZ-104 exam weight:** 20–25%  
**Environment:** Microsoft Entra ID (formerly Azure AD) + Azure RBAC + Azure Policy  
**Case study context:** Onboarding Contoso Ltd's 50-person team to Azure; setting up their identity, access, and governance before any workload is deployed.

---

## What I built

| # | Task | Screenshot |
|---|---|---|
| 1 | Created 5 resource groups (one per domain) | [01-resource-groups.png](screenshots/01-resource-groups.png) |
| 2 | Created 3 Entra ID users across departments | [02-entra-users.png](screenshots/02-entra-users.png) |
| 3 | Created 3 security groups with members assigned | [03-groups.png](screenshots/03-groups.png) |
| 4 | Assigned IT-Admins as Contributor at subscription scope | [04-rbac-subscription.png](screenshots/04-rbac-subscription.png) |
| 5 | Assigned Developers as Contributor scoped to rg-compute | [05-rbac-rg-compute.png](screenshots/05-rbac-rg-compute.png) |
| 6 | Assigned Finance as Reader scoped to rg-monitor | [06-rbac-rg-monitor.png](screenshots/06-rbac-rg-monitor.png) |
| 7 | Created Azure Policy requiring Project tag on all resources | [07-azure-policy-created.png](screenshots/07-azure-policy-created.png) |
| 8 | Tested policy enforcement - blocked resource creation without tag | [08-azure-policy-enforcement.png](screenshots/08-azure-policy-enforcement.png) |
| 9 | Applied CanNotDelete lock on rg-identity, rg-network, rg-monitor | [09-resource-lock-created.png](screenshots/09-resource-lock-created.png) |
| 10 | Tested lock enforcement - deletion attempt blocked | [10-resource-lock-enforced.png](screenshots/10-resource-lock-enforced.png) |
| 11 | Built management group hierarchy: Root → Production → Subscription | [11-management-groups.png](screenshots/11-management-groups.png) |
| 12 | Created $50/month budget with 80% and 100% alert thresholds | [12-cost-management-budget.png](screenshots/12-cost-management-budget.png) |

---

## Resource Groups

The first thing I did was create all 5 resource groups in Canada Central - one per AZ-104 domain. This keeps everything organized and makes RBAC scoping clean.

```
rg-identity   → Entra ID, RBAC, Policy, Locks, Management Groups
rg-network    → VNet, Subnets, NSG, Bastion, DNS
rg-compute    → VMs, VMSS, App Service, Containers
rg-storage    → Storage Account, Blob, Files, Private Endpoint
rg-monitor    → Log Analytics, Alerts, Recovery Vault, Advisor
```

**Why this matters in a real company:** Every enterprise organizes resources into resource groups by workload, environment, or team. Getting this structure right from day one avoids the mess of having everything in one group with no clear ownership.

---

## Entra ID Users & Groups

I created 3 users representing Contoso's three departments:

| User | Department | Job Title |
|---|---|---|
| Dexter McPherson | IT | Systems Administrator |
| Velma Dinkley | Developers | Software Developer |
| Squidward Tentacles | Finance | Financial Analyst |

Then created matching Security groups with Assigned membership:

- **IT-Admins** - contains Dexter McPherson
- **Developers** - contains Velma Dinkley  
- **Finance** - contains Squidward Tentacles

**Why groups instead of assigning access to individual users?**  
Because in a real company, people join and leave teams constantly. If you assign access directly to users, every change requires finding and updating individual role assignments. With groups, you just add or remove the person from the group - their access updates automatically everywhere.

**Exam note:** Know the difference between Assigned and Dynamic group membership. Dynamic groups automatically add members based on attributes like department or job title - useful at scale but requires Entra ID P1 license.

---

## RBAC - Role-Based Access Control

This is where least-privilege gets implemented. The idea is simple: give people the minimum access they need to do their job, nothing more.

Here's what I configured and why:

### IT-Admins → Contributor at Subscription scope
IT needs to manage infrastructure across all resource groups. Contributor lets them create and manage resources but not assign roles to others. Scoped to the full subscription so they can work anywhere.

### Developers → Contributor at rg-compute only
Developers need to deploy and manage compute resources (VMs, containers, app services) but have no business touching networking, storage configuration, or monitoring dashboards. Scoping to rg-compute enforces this boundary.

### Finance → Reader at rg-monitor only
Finance needs visibility into costs and resource health but should never be able to modify anything. Reader on rg-monitor gives them access to dashboards and advisor recommendations without any risk of accidental changes.

**The scope hierarchy - important for the exam:**
```
Management Group
    └── Subscription          ← IT-Admins assigned here
            └── rg-compute    ← Developers assigned here
            └── rg-monitor    ← Finance assigned here
```

Roles assigned at a higher scope are inherited downward. So IT-Admins' Contributor at subscription level means they automatically have Contributor in every resource group - including rg-compute and rg-monitor.

**In a real company:** Incorrect RBAC is one of the most common cloud security findings. Giving everyone Owner or Contributor at subscription level is fast to set up but a serious security risk. Getting this right takes thought.

---

## Azure Policy

The business requirement: Contoso's finance team needs to track cloud spending by project. Without consistent tagging, they can't tell what's costing money or which team is responsible.

I created a policy that **requires a `Project` tag on every resource**. When someone tries to create a resource without that tag, Azure blocks it and explains why.

**How I tested it:** Tried to create a storage account without adding the Project tag. Got a policy violation error at deployment - the resource was blocked from being created. Added `Project: CloudCanvas` as a tag and it went through.

**The difference between Policy and RBAC - this comes up on the exam:**
- RBAC controls *who* can do something
- Policy controls *what* can be deployed

You can have RBAC permissions to create a VM but Policy can still block you from creating one in a region that's not approved, or without required tags, or using an expensive VM size.

**In a real company:** Untagged resources are a nightmare for FinOps teams. A policy like this enforces consistency without relying on people to remember. You'll also see policies that restrict deployments to approved regions, require encryption at rest, or prevent public IP creation.

---

## Resource Locks

I applied CanNotDelete locks on three resource groups:

- `rg-identity-lock` on rg-identity
- `rg-network-lock` on rg-network  
- `rg-monitor-lock` on rg-monitor

**Why these three?** They contain infrastructure that would be catastrophic to accidentally delete:
- rg-identity → all user accounts and access configuration
- rg-network → the network that everything else depends on
- rg-monitor → backup vaults and monitoring data

**How I tested it:** Tried to delete rg-identity after applying the lock. Azure returned an error: the resource is locked and cannot be deleted. The lock worked.

**Two things to know cold for the exam:**

1. `CanNotDelete` - you can still modify resources, just can't delete them
2. `ReadOnly` - you can't modify OR delete. More restrictive.

Locks apply to **all users including subscription Owners**. There is no way to override a lock without first removing it - and removing it requires the right RBAC permissions.

---

## Management Groups

Management groups sit above subscriptions in the Azure hierarchy. They're for organizations running multiple subscriptions who want to apply governance across all of them from one place.

I built this hierarchy:

```
Tenant Root Group (auto-created by Azure)
    └── CloudCanvas Root
            ├── Production       ← Azure subscription 1 lives here
            └── Dev-Test         ← Empty for now
```

Any policy or RBAC assignment made at CloudCanvas Root applies to both Production and Dev-Test. Any policy at Production applies only to subscriptions under it.

**Why this matters at enterprise scale:** A large company might have 50+ subscriptions :- one per team, per environment, per region. Managing governance individually on each subscription is impossible. Management groups let you set policy once at the top and have it cascade down to everything.

**Exam note:** You can nest up to 6 levels of management groups (not counting the root). A subscription can only belong to one management group at a time.

---

## Cost Management - Budget Alert

I created a `CloudCanvas-Budget` at $50 CAD per month with:
- Alert at 80% ($40) - early warning
- Alert at 100% ($50) - at limit
- Duration: May 2026 → April 2027

**Important to understand:** Budget alerts are *notifications only*. When you hit 100% of your budget, Azure does not stop your resources. It sends you an email. You still need to act on it manually or use automation (Azure Automation + Action Groups) to respond.

**In a real company:** Budget alerts are the first line of defense against surprise cloud bills. Paired with cost analysis (breaking down spend by resource group, service, or tag), this gives finance teams the visibility they need to hold teams accountable for cloud spending.

---

## Key Exam Topics from This Section

Here's what I'd focus on if you're studying for AZ-104:

**RBAC:**
- Scope hierarchy: Management Group → Subscription → Resource Group → Resource
- Built-in roles: Owner (full access + assign roles), Contributor (full access, no role assignment), Reader (read only)
- Custom roles require JSON definition and are assigned like built-in roles
- Deny assignments override allow; you can't work around a deny with a higher allow

**Azure Policy:**
- Policy ≠ RBAC. They work together but do different things
- Effect types: Audit (logs non-compliance), Deny (blocks deployment), DeployIfNotExists (auto-deploys remediation), Append (adds fields)
- Initiative = collection of policies applied together
- Compliance dashboard shows which resources are compliant vs not

**Resource Locks:**
- CanNotDelete vs ReadOnly
- Locks are inherited by child resources
- Even Owners cannot bypass locks; the lock must be removed first

**Management Groups:**
- Up to 6 levels deep (excluding root)
- Policy and RBAC assignments cascade downward
- One subscription = one management group at a time

**Entra ID:**
- Assigned vs Dynamic group membership
- B2B guest accounts (external users) vs Member accounts
- SSPR requires P1/P2 for on-premises writeback

---

## Resources

- [Microsoft Learn - AZ-104 Identity path](https://learn.microsoft.com/en-us/training/paths/az-104-manage-identities-governance/)
- [RBAC built-in roles reference](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Azure Policy built-in definitions](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies)

---

*Built by Mann Patel as part of the CloudCanvas AZ-104 lab series.*  
*[Back to CloudCanvas →](https://mannp-cloud.github.io/cloudcanvas)*
