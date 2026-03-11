---
title: "VMFS Datastore Corruption: How It Happens and How to Recover"
date: 2019-05-14
tags: ["VMware", "VMFS", "Storage", "Datastore", "Recovery", "Troubleshooting"]
---

VMFS (VMware File System) is a clustered filesystem that lets multiple ESXi hosts access the same LUN simultaneously. It's reliable, well-tested, and handles concurrent access through distributed locking mechanisms. But VMFS corruption does happen — typically due to storage controller issues, firmware bugs, incorrect multipath configuration, or storage array problems — and knowing how to diagnose and recover from it is essential operational knowledge.

## How VMFS Corruption Happens

The most common causes in real environments:

**Incorrect multipath configuration**: As discussed in the EMC storage post, running multiple paths with an incorrect path selection policy can cause write conflicts that corrupt the VMFS metadata. Two paths writing to the same LUN with an active-active PSP on an ALUA array that expects active-passive is a recipe for filesystem corruption under write load.

**Storage array firmware bugs**: This happens more than vendors acknowledge. A firmware update on a storage array can introduce bugs in the write path that corrupt in-flight data. The symptoms are usually subtle — VMs that become unresponsive, VMDK files that appear corrupted, or datastores that randomly go offline.

**Sudden loss of storage connectivity**: If ESXi hosts lose access to a LUN while VMs are actively writing to it (power failure on the storage array, SAN fabric failure, iSCSI connectivity drop), the VMFS journal may not have time to write cleanly, leaving the filesystem in an inconsistent state.

**Guest OS I/O abuse**: A guest OS writing extremely aggressively to a thin-provisioned VMDK on a datastore that fills up can cause VMFS to encounter allocation failures mid-write. Always monitor datastore space and alert well before capacity is exhausted.

## Diagnosing VMFS Problems

When a datastore becomes unavailable or VMs on it start misbehaving, check in this order:

**Storage connectivity**: Can the ESXi hosts see the LUN? Check `esxcli storage core device list` for the device state. A device in an `off` or `degraded` state indicates connectivity problems that need to be resolved at the storage or network layer before VMFS can be repaired.

**VMFS metadata**: Once connectivity is confirmed, check the VMFS filesystem itself. On the ESXi host, `vmkfstools -V` verifies filesystem integrity. Look for "VMFS: Volume Metadata is corrupted" or similar messages in `/var/log/vmkernel.log`.

**vCenter alarm history**: Check what alarms fired and when. The sequence of events (which host reported the issue first, what storage paths changed around the same time) helps identify whether the root cause is a single host issue or cluster-wide.

## Recovery Options

**Resignature and remap**: If the datastore went offline and came back but vCenter isn't showing it, the VMFS volume may need to be resignatured or remapped. Use `esxcli storage vmfs snapshot list` to see unmounted VMFS volumes, then mount or resignature as appropriate.

**vmkfstools repair**: For VMFS metadata corruption, `vmkfstools --repair` can repair filesystem metadata in some cases. This requires the volume to be unmounted — which means VMs on it must be powered off. This is a recovery operation, not a maintenance operation, and should only be used when VMs are already down.

**Restore from backup**: When filesystem-level repair isn't sufficient and VMDKs themselves are corrupted, restore from backup is the only complete recovery path. This is why backup verification (actually test-restoring VMs periodically) matters — discovering your backups aren't usable during a VMFS corruption event is a very bad time.

## Prevention

The preventive measures that most consistently avoid VMFS corruption:

- Correct multipath configuration validated at deployment and verified after any storage changes
- Datastore space monitoring with alerts at 80% capacity
- Regular VMFS filesystem checks scheduled during maintenance windows (not the same as backup verification, but complementary)
- Storage firmware and driver updates tracked and applied on a regular cadence — not deferred indefinitely

Keep your storage vendor's recommended ESXi driver and firmware matrix current. Compatibility matrix drift between ESXi driver versions and storage firmware is a common source of subtle I/O issues that precede more serious corruption.
