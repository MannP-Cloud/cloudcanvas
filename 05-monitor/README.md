# 05 · Monitor & Maintain

This is the domain that separates reactive admins from proactive ones. Building infrastructure is one thing - knowing when something is wrong before users notice is another skill entirely. Azure Monitor, Log Analytics, and Recovery Services Vault are the tools that give you eyes and ears across everything you've built.

This domain also ties the entire CloudCanvas environment together. The Log Analytics Workspace receives data from the VMs we built in Domain 3. The alerts fire based on metrics from those VMs. The Recovery Vault backs up the same Windows VM that runs IIS. Everything connects.

**AZ-104 exam weight:** 10–15%  
**Environment:** Log Analytics Workspace + VM Insights + Alert Rules + Recovery Services Vault + VM Backup  
**Case study context:** Contoso Ltd's IT manager has one standing request - don't let me find out about problems from users. That means proactive monitoring, automated alerts, and verified backups before anything goes to production.

---

## What I built

| # | Resource | Purpose | Screenshot |
|---|---|---|---|
| 1 | cloudcanvas-law (Log Analytics Workspace) | Central repository for all log and metric data | [01-log-analytics.png](screenshots/01-log-analytics.png) |
| 2 | VM Insights on contoso-win-vm | Performance monitoring and health visibility | [02a-vm-insights.png](screenshots/02a-vm-insights.png) |
| 3 | VM Insights on contoso-linux-vm | Performance monitoring on Linux VM | [02b-vm-insights.png](screenshots/02b-vm-insights.png) |
| 4 | Alert rule - High-CPU-Windows-VM | Email alert when CPU exceeds 80% for 5 minutes | [03-alert-rule.png](screenshots/03-alert-rule.png) |
| 5 | cloudcanvas-vault (Recovery Services Vault) | Backup storage for VM recovery points | [04-recovery-vault.png](screenshots/04-recovery-vault.png) |

---

## Log Analytics Workspace - cloudcanvas-law

**Configuration:**
- Name: cloudcanvas-law
- Resource group: rg-monitor
- Region: Canada Central
- Retention: 30 days (default)

**What it is:** Log Analytics Workspace is the central data store for monitoring in Azure. Resources send their logs and metrics here. You query the data using KQL (Kusto Query Language). Azure Monitor, Microsoft Sentinel, VM Insights, and Defender for Cloud all use Log Analytics as their backend.

**Why rg-monitor?** All monitoring resources live in their own resource group with a CanNotDelete lock. This is intentional - if you accidentally delete your monitoring infrastructure, you lose visibility into everything else. The lock prevents that.

**The 30-day default retention** means log data is queryable for 30 days. After that it's deleted unless you configure longer retention or archive to a storage account. Compliance requirements often mandate longer retention - financial services typically require 7 years, healthcare may require similar. Extended retention costs extra but is non-negotiable in regulated industries.

**KQL - the query language you need to know:**

```kql
// Find all VM heartbeats in the last hour
Heartbeat
| where TimeGenerated > ago(1h)
| summarize count() by Computer

// Show CPU usage over time
Perf
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart

// Find failed login attempts on Windows
SecurityEvent
| where EventID == 4625
| project TimeGenerated, Computer, Account, IpAddress
```

KQL is used by every Azure ops role. It's not just a monitoring skill - it's the foundation of Microsoft Sentinel (SIEM), Defender for Cloud, and Application Insights. Learning it properly is one of the highest-value investments you can make after AZ-104.

---

## VM Insights

**What it shows:**
- VM availability status
- CPU utilization over time
- Memory usage
- Disk read/write performance
- Network in/out
- Azure outages affecting the VM
- Health events

**Why this matters:** Before VM Insights, understanding VM performance required manually querying Log Analytics or pulling metric charts one by one. VM Insights provides a pre-built dashboard that gives you the full picture in one view - availability, performance, and connectivity.

**The Map feature** (visible in the tabs) shows network connections from the VM - what it's talking to, which ports, and the volume of traffic. This is invaluable during incident investigation. If a VM starts behaving unexpectedly, the Map shows you immediately if it's making unusual outbound connections.

**Diagnostic settings and the policy:** During this lab, enabling diagnostic settings through the Portal was blocked by our "Require Project Tag" policy because the diagnostic extension resource can't accept tags through the Portal UI. VM Insights was enabled through the Monitor blade which bypasses this limitation. This is a real-world scenario - complex governance policies sometimes require alternative deployment paths for specific resources.

---

## Alert Rule - High-CPU-Windows-VM

**Configuration:**
- Signal: Percentage CPU (metric)
- Condition: Greater than 80%
- Aggregation: Average over 5 minutes
- Severity: 2 - Warning
- Target: contoso-win-vm
- Action: contoso-alert-group (email notification)
- Status: Enabled

**Why CPU > 80%?** This threshold is a common starting point. If a VM's CPU averages above 80% for 5 consecutive minutes, something significant is happening - either a legitimate workload spike that needs investigation, or a runaway process consuming resources. Below 80% is normal fluctuation. Above 80% sustained warrants attention.

**The 5-minute aggregation window** prevents alert noise. A single spike to 95% for 30 seconds is normal - a process starting up, a scheduled task running. Averaging over 5 minutes filters out transient spikes and only fires when there's a sustained issue.

**Severity levels:**
- 0 - Critical: Service is down, immediate action required
- 1 - Error: Service degraded, urgent investigation needed
- 2 - Warning: Approaching threshold, investigate soon
- 3 - Informational: Notable event, no immediate action
- 4 - Verbose: Diagnostic information

**Action Groups** are reusable. The `contoso-alert-group` we created can be assigned to multiple alert rules - CPU alerts, disk space alerts, VM availability alerts - all sending to the same email. One change to the action group updates all alerts that use it.

**Alert types you need to know for the exam:**
- **Metric alerts** - Near real-time, evaluated every minute. Used for CPU, memory, disk, network metrics.
- **Log alerts** - Run a KQL query on a schedule. Used for log-based conditions like failed logins or error patterns.
- **Activity log alerts** - Trigger on Azure control plane events. Used for things like "alert me when any VM is deleted" or "alert me when a new role assignment is made."

---

## Recovery Services Vault - cloudcanvas-vault

**Configuration:**
- Name: cloudcanvas-vault
- Resource group: rg-monitor
- Region: Canada Central
- Redundancy: GRS (Geo-Redundant Storage) - default
- Protected item: contoso-win-vm
- Policy: DefaultPolicy
- Backup Pre-Check: Passed

**Why GRS for the vault?** Unlike the storage account where we chose LRS to save costs, the Recovery Services Vault defaults to GRS - and for good reason. Backup data is your last line of defense. If the primary Azure region has a disaster, you need your backups in a secondary region. Changing redundancy after adding backup items is not allowed - it must be set before the first backup.

**DefaultPolicy schedule:**
- Frequency: Daily
- Time: 9:30 PM UTC
- Retention: 30 days of daily recovery points

**Backup Pre-Check: Passed** means Azure verified that the VM configuration is compatible with backup - the VM is running, the agent can be installed, and there are no blocking conditions.

**Initial backup pending** - the first backup hasn't run yet because it's scheduled for 9:30 PM UTC. You can trigger it manually:

```bash
az backup protection backup-now \
  --resource-group rg-monitor \
  --vault-name cloudcanvas-vault \
  --container-name "IaasVMContainer;iaasvmcontainerv2;rg-compute;contoso-win-vm" \
  --item-name "VM;iaasvmcontainerv2;rg-compute;contoso-win-vm" \
  --backup-management-type AzureIaasVM \
  --retain-until 25-05-2026
```

**Soft delete** is enabled by default - deleted backup items are retained for 14 days before permanent deletion. This protects against accidental or malicious deletion of backup data.

**Immutable vault** - not configured here but worth knowing for the exam. An immutable vault prevents backup data from being deleted or modified even by administrators. Required in some compliance frameworks.

**File-level restore** - you don't have to restore the entire VM to recover a single file. Azure Backup can mount the recovery point as a disk on the target VM, letting you browse and copy individual files. The mount script is valid for 12 hours.

---

## The monitoring architecture - how it all connects

```
contoso-win-vm ──→ Azure Monitor Agent ──→ cloudcanvas-law
contoso-linux-vm ─→ Azure Monitor Agent ──→ cloudcanvas-law
                                                  │
                                          KQL queries
                                          Alert evaluation
                                          VM Insights dashboards
                                                  │
                                    Percentage CPU > 80% for 5min
                                                  │
                                         contoso-alert-group
                                                  │
                                           Email to IT team

contoso-win-vm ──→ cloudcanvas-vault ──→ Daily backup at 9:30 PM UTC
                                     ──→ 30 days retention
                                     ──→ GRS replication to secondary region
```

---

## The policy lesson - a recurring theme

Throughout Domain 5 we hit the "Require Project Tag" policy multiple times - diagnostic settings extensions couldn't be tagged through the Portal, requiring the Monitor blade as an alternative deployment path.

This is the real-world lesson that runs through the entire CloudCanvas lab: **governance policies have unintended consequences**. A well-intentioned tag requirement designed to improve cost visibility can block legitimate resource deployments if not carefully scoped. In production, you'd use policy exemptions for specific resource types that can't accept tags, or modify the policy definition to exclude child resources like NICs and diagnostic extensions.

This understanding - that governance, security, and operations sometimes create friction that needs to be worked around intelligently - is what separates someone who has read about Azure from someone who has actually administered it.

---

## Key exam topics from this section

**Azure Monitor:**
- Metrics: numeric time-series, stored 93 days, evaluated near real-time
- Logs: structured text data in Log Analytics, queried with KQL
- Azure Monitor is the umbrella - Insights, Alerts, Workbooks all sit under it

**Log Analytics:**
- KQL is the query language - learn at least the basics
- Common tables: Heartbeat, Perf, Event, Syslog, AzureActivity, SecurityEvent
- Default retention: 30 days. Configurable up to 730 days.
- Multiple workspaces possible - centralised vs distributed is a design decision

**Alerts:**
- Three types: Metric, Log, Activity Log
- Action Groups are reusable across multiple alert rules
- Severity 0-4: Critical, Error, Warning, Informational, Verbose
- Alert states: Fired, Resolved

**Recovery Services Vault:**
- Set redundancy BEFORE adding backup items - cannot change after
- DefaultPolicy: daily at 9:30 PM UTC, 30-day retention
- Soft delete: 14-day retention of deleted backups
- Immutable vault: prevents deletion even by admins
- File-level restore: mount recovery point as disk, browse individual files
- Backup Pre-Check: validates VM is ready for backup before first run

**VM Insights:**
- Requires Azure Monitor Agent on the VM
- Map feature requires Dependency Agent
- Shows performance trends, availability, and network connections
- Data stored in Log Analytics Workspace

---

## Resources

- [Microsoft Learn - AZ-104 Monitor path](https://learn.microsoft.com/en-us/training/paths/az-104-monitor-backup-resources/)
- [KQL quick reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [Azure Monitor alerts overview](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
- [Azure Backup overview](https://learn.microsoft.com/en-us/azure/backup/backup-overview)

---

*Built by Mann Patel as part of the CloudCanvas AZ-104 lab series.*  
*[← Back to CloudCanvas](https://mannp-cloud.github.io/cloudcanvas)*
