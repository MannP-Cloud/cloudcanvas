# 03 · Compute

If Networking is the skeleton of the Azure environment, Compute is where the actual work happens. This is where Contoso's servers live - the Windows machine the IT team RDPs into, and the Linux machine the developers SSH into. Both sit inside the network we built in Domain 2, both are completely unreachable from the public internet, and both are accessed exclusively through Azure Bastion.

**AZ-104 exam weight:** 20–25%  
**Environment:** Windows VM + Linux VM + Availability Set + NAT Gateway  
**Case study context:** Contoso Ltd needs two servers. One for IT administration running Windows Server, one for development work running Ubuntu Linux. The security requirement hasn't changed - no public IPs, no open RDP or SSH ports on the internet, zero attack surface.

---

## What I built

| # | Resource | Purpose | Screenshot |
|---|---|---|---|
| 1 | contoso-avset (Availability Set) | Protects both VMs from hardware failures | [01-availability-set.png](screenshots/01-availability-set.png) |
| 2 | contoso-win-vm (Windows Server 2022) | IT admin server in subnet-admin | [02-windows-vm.png](screenshots/02-windows-vm.png) |
| 3 | contoso-linux-vm (Ubuntu 24.04) | Developer server in subnet-dev | [03-both-vms.png](screenshots/03-both-vms.png) |
| 4 | Bastion RDP to Windows | Proved secure access with no public IP | [04-bastion-rdp-windows.png](screenshots/04-bastion-rdp-windows.png) |
| 5 | Bastion SSH to Linux | Proved secure SSH through browser | [05-bastion-ssh-linux.png](screenshots/05-bastion-ssh-linux.png) |
| 6 | IIS installed on Windows | Web server role via PowerShell | [06-iis-installed.png](screenshots/06-iis-installed.png) |
| 7 | nginx installed on Linux | Web server via apt after NAT Gateway fix | [07-nginx-installed.png](screenshots/07-nginx-installed.png) |
| 8 | contoso-nat-gateway | Outbound internet for VMs without public IPs | [08-nat-gateway.png](screenshots/08-nat-gateway.png) |

---

## Why an Availability Set first?

Before deploying a single VM, I created the availability set. This is intentional - you can't add an existing VM to an availability set after creation. It has to be configured at deployment time.

An Availability Set protects against two types of failure:

**Fault domains** - separate physical racks with their own power and network. We configured 2. If one rack's power supply fails, only the VMs on that fault domain go down. The others keep running.

**Update domains** - separate groups for maintenance. We configured 5. When Azure performs planned maintenance and needs to restart hosts, it only restarts one update domain at a time. With 5 update domains, at most 20% of your VMs are ever affected simultaneously.

**What this gives Contoso:** If Azure needs to restart the host running the Windows VM for maintenance, the Linux VM is in a different update domain and stays up. If a hardware failure takes out one rack, not both VMs are affected.

**The 99.95% SLA** only applies when you have 2+ VMs in an availability set or availability zone. A single VM gets 99.9% at best - that's 8.7 hours of potential downtime per year versus 4.4 hours. For a business running critical services, that difference matters.

---

## Windows VM - contoso-win-vm

**Specs:**
- OS: Windows Server 2022 Datacenter
- Size: Standard_B2s (2 vCPUs, 4 GB RAM)
- Disk: Standard SSD OS disk + 32 GiB data disk
- Network: subnet-admin (10.0.1.0/24) in hub-vnet
- Public IP: None
- Availability set: contoso-avset

**Why Standard_B2s?** The B-series are burstable VMs - they accumulate CPU credits when idle and spend them when busy. For a lab environment or lightly-used admin server, this is significantly cheaper than a dedicated compute VM while still handling bursts of activity. In production, you'd size based on actual workload requirements.

**Why a data disk?** Best practice is to separate OS and data. The OS disk runs the operating system. Application data, logs, and user files go on the data disk. This way if you need to rebuild the OS, you detach the data disk, rebuild, reattach - your data survives.

**Connecting via Bastion:** The Windows VM has no public IP and port 3389 is blocked from the internet by the NSG we created in Domain 2. The only way in is through `cloudcanvas-bastion` which proxies the RDP session over HTTPS port 443 through the browser. This is exactly how modern enterprise environments handle remote access - no VPN, no jump server, no open ports.

**IIS Installation:** Once connected, IIS was installed via PowerShell:
```powershell
Install-WindowsFeature -name Web-Server -IncludeManagementTools
```
Result: `Success: True, Restart Needed: No, Exit Code: Success`

This demonstrates that the VM is functional, PowerShell works correctly, and the Windows Server roles system is operational. In a real environment, this is how you'd enable web server capabilities before deploying an application.

---

## Linux VM - contoso-linux-vm

**Specs:**
- OS: Ubuntu 24.04 LTS
- Size: Standard_B1s (1 vCPU, 1 GB RAM)
- Disk: Standard SSD OS disk
- Network: subnet-dev (10.0.2.0/24) in hub-vnet
- Public IP: None
- Authentication: SSH key pair (contoso-linux-key.pem)
- Availability set: contoso-avset

**Why SSH key auth instead of password?** SSH keys are the industry standard for Linux VM access. A 2048-bit RSA key is exponentially harder to brute force than any password. The private key stays on your machine, the public key lives on the VM - even if someone intercepts your connection, they can't authenticate without the private key file.

**Connecting via Bastion SSH:** The browser-based SSH terminal connected to the VM using the downloaded `.pem` key file. The login source IP was `10.0.0.4` - that's the Bastion service's internal IP, confirming all traffic came through the Bastion subnet as intended.

---

## The NAT Gateway problem - and why it matters

When we tried to install nginx on the Linux VM, it failed with connection timeouts trying to reach `azure.archive.ubuntu.com`. The VM had no outbound internet access.

This is the correct security behavior - a VM with no public IP and no NAT Gateway cannot initiate outbound connections to the internet. But it also means you can't install software, pull updates, or download anything.

**The enterprise solution: NAT Gateway**

A NAT Gateway gives VMs outbound internet access without giving them inbound public IPs. Traffic flows like this:

```
Linux VM (10.0.2.x)
    → subnet-dev route table
        → contoso-nat-gateway
            → nat-pip (public IP)
                → internet (apt repositories, updates, etc.)
```

The VM still has no public IP. Nobody on the internet can initiate a connection to it. But the VM can initiate outbound connections - for software installation, OS updates, calling external APIs.

**Why not just give the VM a public IP?** Because then it becomes a target. Every VM with a public IP is actively scanned by automated bots within minutes of deployment, probing for open ports and weak credentials. A NAT Gateway gives you outbound connectivity without the inbound attack surface.

**The policy problem we hit:** Creating the NAT Gateway's public IP through the Portal was blocked by our "Require Project Tag" policy. Public IPs don't have a tags tab during NAT Gateway creation. The fix was using Azure CLI with the `--tags` flag to create the public IP and NAT Gateway separately with the required tag.

This is a real-world scenario. Blanket tag-enforcement policies need exemptions or workarounds for certain resource types. Understanding when to use CLI versus Portal - and why - is a genuine Azure admin skill.

After the NAT Gateway was created and associated to subnet-dev, nginx installed cleanly:

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl status nginx
# Active: active (running)
```

---

## VM sizing - what you need to know for the exam

Azure VM sizes follow a naming convention:

```
Standard_B2s
         │ │
         │ └── s = premium storage capable
         │
         └── 2 = number of vCPUs
  
B = Burstable series
```

**Common series for AZ-104:**
- **B-series** - Burstable, cost-effective for dev/test and light workloads
- **D-series** - General purpose, balanced CPU/memory for production
- **E-series** - Memory optimized for databases and in-memory workloads
- **F-series** - Compute optimized for CPU-intensive workloads

**Deallocating vs Stopping:** This distinction matters for costs and the exam.
- **Stop (OS level)** - VM is still allocated on a host, still charging for compute
- **Deallocate (Portal Stop button)** - VM is released from the host, no compute charges. Storage still charges.

Always use **Deallocate** in a lab environment when not using VMs.

---

## Key exam topics from this section

**Availability Sets:**
- Must be configured at VM creation - cannot add existing VM later
- Fault domains = separate power/network (physical separation)
- Update domains = separate maintenance windows
- 99.95% SLA requires 2+ VMs in availability set
- Availability Zones provide higher SLA (99.99%) but across different datacenters

**VM deployment:**
- VM size determines CPU, RAM, and storage throughput
- OS disk and data disk are separate - always use data disks for application data
- Authentication: password for Windows, SSH key recommended for Linux
- Public IP None = no inbound internet access

**Bastion:**
- Connects over HTTPS port 443 - works through most corporate firewalls
- No agent needed on the VM
- Basic SKU: browser only. Standard SKU: native client + file transfer
- Source IP on the VM shows the Bastion subnet IP, not your actual IP

**NAT Gateway:**
- Provides outbound internet for VMs without public IPs
- Fully managed - no maintenance, scales automatically
- Associated at subnet level
- Complements Private Endpoints and VNet isolation

**VM Extensions:**
- Run scripts or install software post-deployment
- Custom Script Extension: run any script as SYSTEM/root
- Can be deployed from Portal, CLI, ARM/Bicep
- Useful for bootstrapping VMs at scale without manual login

---

## Resources

- [Microsoft Learn - AZ-104 Compute path](https://learn.microsoft.com/en-us/training/paths/az-104-manage-compute-resources/)
- [VM sizes overview](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview)
- [Availability Sets vs Zones](https://learn.microsoft.com/en-us/azure/virtual-machines/availability)
- [NAT Gateway overview](https://learn.microsoft.com/en-us/azure/nat-gateway/nat-overview)

---

*Built by Mann Patel as part of the CloudCanvas AZ-104 lab series.*  
*[← Back to CloudCanvas](https://mannp-cloud.github.io/cloudcanvas)*
