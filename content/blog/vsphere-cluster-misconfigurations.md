---
title: "vSphere Clusters: Common Misconfigurations in SMB Deployments"
date: 2017-10-30
tags: ["VMware", "vSphere", "Cluster", "Configuration", "SMB"]
---

vSphere cluster misconfigurations don't always cause immediate problems. Sometimes they sit dormant for months before a host failure or upgrade event triggers a failure mode that the misconfiguration created. These are the ones I find most consistently when auditing SMB VMware environments.

## EVC Mode Not Configured

Enhanced vMotion Compatibility (EVC) sets a CPU compatibility baseline for the cluster, masking CPU features of newer processors to the level of the oldest CPU in the cluster. Without EVC, live migrating a VM from a newer CPU host to an older one may fail if the VM has been running on instructions the older CPU doesn't support.

In practice: you deploy a cluster with four identical hosts, EVC isn't configured because it seems unnecessary. Eighteen months later, you add two newer hosts with a different CPU generation. Now you have a mixed-CPU cluster without EVC, and vMotion failures start appearing when VMs try to migrate from newer to older hosts.

Enable EVC at the cluster level from the beginning, set it to the CPU family baseline of your oldest hardware generation, and it handles future hardware additions gracefully. Enabling EVC on an existing cluster requires all VMs to be powered off temporarily if any VMs are running on unsupported CPU features — plan for a maintenance window.

## HA/DRS Not Enabled on the Cluster (Only on Some Hosts)

This one seems obvious, but it shows up: HA and DRS are configured at the cluster level, not the host level. I've found environments where someone added a host to a cluster after initial setup, HA was enabled on the cluster, but the new host never completed the HA configuration step (usually because VIBs weren't installed correctly or the host had a network issue during joining). The host showed in the cluster but wasn't actually participating in HA.

Check each host's HA participation status periodically via PowerCLI:

```powershell
Get-Cluster | Get-VMHost | Select Name, @{N='HAStatus';E={$_.ExtensionData.Config.DasConfig.HaConnectedToMaster}}
```

## Datastore Clusters with DRS Thresholds Too Aggressive

Datastore clusters (Storage DRS) can automatically balance VMs across multiple datastores. The default threshold triggers migrations when space or I/O imbalance exceeds a configurable threshold. Set too aggressively, Storage DRS generates constant storage vMotion operations that consume I/O bandwidth and cause unnecessary VM performance variability.

For most SMB environments, I recommend setting Storage DRS to manual (generate recommendations but don't act automatically) until you understand the migration frequency the environment requires. Automated Storage DRS is appropriate in larger environments where storage balance is a consistent operational concern.

## VM Startup and Shutdown Not Configured

When a host reboots (for patching, hardware work, or after an unexpected failure and HA restart), VMs power on in an undefined order unless startup rules are configured. Domain controllers that come up after member servers that depend on them, DNS servers that lag behind clients — these sequencing issues cause a cascade of failures after every host reboot.

Configure VM startup policy per cluster: identify which VMs must start first (DCs, DNS, core infrastructure), set their startup priority and delay, and configure dependent VMs to start after the infrastructure tier has a chance to come up.

## Snapshot Retention Without Limits

This isn't directly a cluster misconfiguration, but it affects every environment I audit: VMs with old, forgotten snapshots consuming significant datastore space. A VM that had a snapshot taken "just in case" before a software update and was never deleted is now carrying around a large, constantly growing snapshot delta.

Set a cluster-level alert in vCenter for snapshots older than 72 hours. Review weekly. Delete snapshots when the change they were protecting against has been validated. Snapshot accumulation is silent capacity consumption that manifests as sudden "datastore full" events at the worst possible time.
