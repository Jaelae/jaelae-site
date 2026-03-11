---
title: "Migrating VMs Between Datastores Without Downtime"
date: 2019-04-02
tags: ["VMware", "vSphere", "Storage vMotion", "Datastores", "Migration"]
---

Storage vMotion is the capability that makes VMware environments operationally flexible — you can move a VM's virtual disks from one datastore to another while the VM continues running, without any user-visible downtime. Understanding how it works and what can go wrong makes it a tool you can use confidently.

## How Storage vMotion Works

Storage vMotion mirrors the VM's VMDK files from the source datastore to the destination, tracking changes as the VM continues running. When the mirror is nearly synchronized, the VM is briefly paused (typically under one second), the final deltas are copied, and the VM resumes on the destination datastore. The source files are then cleaned up.

This is different from regular vMotion (which moves a running VM between hosts, keeping its storage in place) and full vMotion (which moves both simultaneously — compute and storage).

## When You Need It

Common use cases:

**Decommissioning storage**: Moving all VMs off a datastore before unmounting it (for hardware replacement, storage migration, or decommissioning an old LUN).

**Storage balancing**: Moving VMs from a nearly-full datastore to one with more available space, without downtime.

**Performance optimization**: Moving a high-I/O VM from shared HDD-backed storage to SSD-backed storage.

**Pre-host maintenance**: Moving VMs to a datastore accessible from multiple hosts before placing a host in maintenance mode.

## Performance Considerations

Storage vMotion consumes I/O bandwidth on both the source and destination datastores simultaneously. On busy datastores, running multiple concurrent Storage vMotions can significantly impact VM performance for other VMs sharing those datastores.

Best practices for production environments:

- Run Storage vMotions during off-peak hours when possible
- Limit concurrent Storage vMotions per datastore (one at a time is conservative, two is usually manageable on fast storage)
- Monitor datastore latency during migration — if latency spikes significantly, pause the migration

In vCenter, you can limit Storage vMotion migrations via DRS settings, or schedule them via vSphere scheduling (limited compared to third-party orchestration).

## Common Failure Scenarios

**Out of space on destination**: The most common failure — the destination datastore fills up mid-migration. Storage vMotion operations can fail partway through if space is exhausted, leaving partially migrated VMDKs in a state that needs cleanup. Always verify available space before starting, and ensure the destination has at least 110% of the source VMDK size available (the extra 10% accounts for snapshot overhead during migration).

**Datastore disconnect during migration**: If connectivity to either the source or destination datastore is interrupted during Storage vMotion, the VM may go into a failed state. If the source is still accessible, the VM continues running from the source datastore. If the source is also inaccessible, the VM may become unresponsive. This is why Storage vMotion between datastores on the same SAN, rather than between SANs, is lower-risk.

**Snapshot chain complexity**: VMs with multiple snapshot layers migrate more slowly because each snapshot layer must be consolidated into the migration. Long snapshot chains (multiple levels deep) can cause Storage vMotion to take significantly longer than expected. Consolidate snapshots before migrating when possible.

## Via PowerCLI

For bulk Storage vMotion operations (moving all VMs off a datastore), PowerCLI is faster than doing it VM-by-VM in the UI:

```powershell
$sourceDS = Get-Datastore "OldDatastore"
$destDS = Get-Datastore "NewDatastore"
Get-Datastore $sourceDS | Get-VM | Move-VM -Datastore $destDS
```

This queues the migration for all VMs on the source datastore. By default, vCenter limits concurrent Storage vMotions — they'll be throttled appropriately. Monitor progress in the vCenter task panel.
