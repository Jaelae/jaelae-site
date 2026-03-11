---
title: "Pure Storage Integration with vSphere: Getting the Most from All-Flash"
date: 2018-03-19
tags: ["VMware", "Pure Storage", "Storage", "All-Flash", "vSphere"]
---

When I first deployed Pure Storage in a VMware environment around 2017, the experience was different from anything I'd done with traditional SAN arrays. The management interface is genuinely simple, the performance headroom is substantial, and the VMware integration through the Pure vSphere Plugin changes some fundamental operational assumptions. Here's what to know going in.

## Why Pure Is Different

Traditional SAN arrays require significant upfront planning around RAID groups, disk pools, tiering policies, and LUN sizing — because you're managing spinning disk performance characteristics and the layout matters. Pure Storage abstracts all of that. The array handles data reduction, redundancy, and distribution internally. You present volumes of whatever size you need and the performance is uniform across the array.

This changes the conversation around storage planning from "how do I lay out my RAID groups" to "how much usable capacity do I need" and "what's my maximum IOPS budget for the array." The latter conversation is simpler and more aligned with how applications actually think about storage.

## The vSphere Plugin and VASA Provider

Pure provides a vSphere Plugin (available as an OVF deployed within your vCenter environment) that integrates array management directly into the vSphere UI. From vCenter, you can create and manage Pure volumes, see per-volume performance metrics, and manage snapshots without switching to the Pure GUI.

The VASA (vStorage APIs for Storage Awareness) provider integration enables Storage Policy-Based Management (SPBM) in VMware. You create VM Storage Policies in vCenter that reference Pure storage capabilities, and vCenter ensures VMs placed on datastores backed by Pure are compliant with those policies. For environments with mixed storage tiers, this is operationally valuable — VMs automatically land on the right tier based on policy.

## VMFS vs. vVols

Traditional VMFS datastores on Pure work well and are the simpler deployment path. Create a volume on Pure, present it as an iSCSI LUN or FC target to your hosts, format it as VMFS in vCenter, and provision VMs. Performance is excellent and management is familiar.

vVols (Virtual Volumes) on Pure are worth the additional complexity for environments that want per-VM snapshot management and data reduction reporting at the VM level. With vVols, each VM's virtual disk is a discrete object on the Pure array rather than a file within a shared VMFS volume. This enables per-VM snapshots at the array level (pure `pgroup` snapshots that are instantly available and have no performance impact), and per-VM data reduction metrics in the Pure UI.

The tradeoff: vVols require the VASA provider to be continuously available. If the vSphere Plugin VM (which hosts the VASA provider) has problems, vVol-backed VMs can become inaccessible. Keep the plugin VM highly available.

## Snapshot Integration

Pure's native snapshots are one of its genuinely compelling features — they're pointer-based, instant, and have essentially zero performance impact on the source volume. For backup workflows, Pure integrates with Veeam Backup & Replication via the Veeam Universal Storage API, allowing Veeam to orchestrate array-level snapshots as part of backup jobs.

The operational model: Veeam triggers a Pure snapshot at backup time (near-instantaneous), mounts the snapshot as a temporary datastore, reads the data, and unmounts it. The actual backup processing happens from the snapshot, not the live datastore, which eliminates the I/O impact of backup on production VMs. For environments with large VMs or high I/O sensitivity, this is transformative compared to traditional agent-based or VSS-based backup approaches.

## Common Integration Mistakes

**Host connectivity**: Ensure each ESXi host has two iSCSI paths (or two FC paths) to Pure. Pure supports both active-active paths, so both paths carry I/O simultaneously rather than one being a standby.

**Path selection policy**: Pure prefers Round Robin with IOPS=1 for iSCSI — same general principle as ALUA-capable arrays. Pure provides specific instructions for their supported path selection configuration in their vSphere best practices guide.

**Volume sizing**: Don't over-provision volumes out of fear of thin provisioning. Pure handles thin provisioning internally, and creating enormous volumes "just in case" doesn't waste array capacity. Create volumes sized for your expected growth and expand as needed — Pure volume expansion is online and takes seconds.
