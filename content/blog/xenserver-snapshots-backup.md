---
title: "XenServer Snapshot Gotchas: Why Snapshots Aren't Backups"
date: 2015-11-30
tags: ["XenServer", "Citrix", "Snapshots", "Backup"]
---

Snapshots are the most misunderstood feature in virtualization. On XenServer, the confusion is compounded by the fact that snapshots look useful for backup purposes and are easy to take. Let me save you from the mistake.

## What a XenServer Snapshot Actually Is

A snapshot in XenServer freezes the state of a VM's virtual disk at a point in time. After the snapshot is taken, all new writes go to a differencing VDI (the snapshot delta), while the original VDI stays unchanged. To revert to the snapshot, XenServer discards the delta and the VM rolls back to its pre-snapshot state.

This is useful for: pre-patch checkpoints, pre-upgrade safety nets, testing configuration changes before committing.

This is not useful for: disaster recovery, ransomware recovery, accidental deletion recovery, anything you'd normally think of as "backup."

## Why Snapshots Fail as Backup

Snapshots exist only on the same storage as the VM they protect. If your storage array fails, you lose both the VM and all its snapshots. They offer zero protection against hardware failure.

More critically: snapshots consume space on the same datastore as the VM. Long-lived snapshots on a busy disk can grow to consume more space than the original VDI if the VM generates significant write activity. Running low on SR space while a large snapshot delta is open is a recipe for a very bad afternoon — XenServer will pause or crash VMs if the SR runs out of space.

XenCenter's default view doesn't make the space consumption of snapshot chains immediately obvious. Check it explicitly: `xe vdi-list snapshot-of=<parent-vdi-uuid>` to enumerate snapshot VDIs and `xe vdi-param-get uuid=<uuid> param-name=physical-utilisation` to check their actual disk consumption.

## The Backup Approaches That Actually Work

For XenServer, the backup approaches I've seen work reliably in production:

**Export to XVA**: XenServer's native export format creates a portable VM archive that includes all disks and metadata. You can export to a network share: `xe vm-export vm=<name-label> filename=\\server\share\vm-backup.xva`. It requires the VM to be shut down or uses the snapshot mechanism for running VMs. Slow for large VMs but reliable.

**Third-party backup software**: Veeam Backup & Replication supported XenServer for a period and could take application-consistent backups of running VMs. Unitrends and PHD Virtual (before it was acquired) were also options in the 2015-2016 timeframe. Check compatibility with your specific XenServer version before committing.

**XenServer's built-in VM protection**: XenServer 6.5 included a basic VM snapshot-based protection schedule through XenCenter. It's not enterprise backup, but it provides automatic periodic snapshots for VMs when you don't have a third-party solution. Better than nothing for non-critical VMs.

## Application Consistency

Taking a snapshot of a running Windows VM without coordinating with the guest OS creates a crash-consistent backup, not an application-consistent one. For databases, Exchange, or any application with open transactions, crash consistency means you get whatever state was on disk at snapshot time — potentially with incomplete transactions.

XenServer's VSS integration (via XenServer Tools installed in the guest) can coordinate with Windows VSS to create application-consistent snapshots. Always install XenServer Tools in your Windows VMs and verify VSS coordination is working before relying on VM snapshots for any application that matters.

The lesson I've repeated too many times: test your restores. A backup you've never restored is just optimism stored on disk.
