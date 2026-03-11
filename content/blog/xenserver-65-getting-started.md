---
title: "Getting Started with XenServer 6.5: The VMware Alternative I Actually Liked"
date: 2015-02-10
tags: ["XenServer", "Virtualization", "Citrix", "SMB"]
---

In 2015, I was working with a mid-size client who needed to virtualize roughly 20 servers on a budget that made VMware's licensing difficult to justify. They'd looked at Hyper-V but didn't have strong Windows Server Datacenter licensing already in place. XenServer 6.5 was free (for the base edition), mature enough for production use, and had a reasonable management tool in XenCenter. It became a genuine option worth taking seriously.

## Why XenServer Made Sense Here

XenServer is Citrix's hypervisor, built on the open-source Xen project with Citrix's management layer on top. The free edition gave you live migration (XenMotion), resource pools for clustering hosts, and a Windows-based management console — feature parity with much of what you'd get from VMware for basic virtualization needs.

For a 20-server consolidation onto 4 physical hosts with shared iSCSI storage, XenServer was legitimately adequate. The licensing cost difference versus VMware Essentials Plus (at the time) was significant enough that the client could put the savings toward better hardware.

## Installation: What the Docs Don't Mention

XenServer installs as a Linux-based hypervisor (CentOS derivative under the hood). The installer is straightforward, but a few things caught me off-guard on the first deployment:

**RAID configuration should happen before install.** The XenServer installer has a basic RAID option but it's limited and poorly documented. If your server has a hardware RAID controller, configure your RAID array in the RAID BIOS before booting the XenServer installer. XenServer will then see a single logical disk and proceed cleanly.

**NIC bonding is easier post-install.** The installer offers NIC bonding during setup, but the configuration there is more limited than what XenCenter gives you post-install. Install on a single NIC, then add your bond interfaces through XenCenter — you get mode selection (active-passive vs LACP), VLAN assignment, and proper management network assignment.

**Set a correct hostname and DNS before joining a pool.** Pool operations use hostnames for node communication. A server named `localhost.localdomain` in a resource pool will cause confusing behavior when pool commands run. Set the hostname correctly at install time or immediately after via `xe host-param-set uuid=<uuid> hostname=<name>` in the CLI.

## XenCenter vs. The CLI

XenCenter is the Windows-based GUI and covers about 80% of what you'll need day-to-day. The xe CLI (accessed via SSH to any XenServer host) handles the rest and is necessary for scripting, bulk operations, and some configuration options the GUI doesn't expose.

Get comfortable with a few basic xe commands early:

```bash
xe vm-list              # List all VMs
xe host-list            # List pool members
xe sr-list              # List storage repositories
xe vbd-list             # List virtual block devices (disks)
```

The CLI is verbose but consistent. Most operations have a `--help` flag that shows the required parameters, and the syntax is uniform enough that you can usually guess the right command format for unfamiliar operations.

## First Impressions as a VMware Admin

Coming from vSphere, XenServer felt slightly rougher around the edges — XenCenter lacked some of the polished reporting and alerting in vCenter, and live migration (XenMotion) had more constraints than vMotion in terms of networking and storage prerequisites. But for what this client needed, it worked reliably.

The biggest operational difference: pool master failure is a real concern in XenServer that doesn't have a clean equivalent in vSphere. I'll cover that separately in a dedicated post, because it deserves its own treatment.
