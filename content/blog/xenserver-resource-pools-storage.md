---
title: "XenServer Resource Pools: Configuring Shared Storage for Live Migration"
date: 2015-07-14
tags: ["XenServer", "Citrix", "Resource Pool", "Storage", "XenMotion"]
---

XenMotion — XenServer's live migration capability — requires shared storage. If all your VMs live on local storage (the default after a fresh install), you can't migrate them between hosts. Setting up a resource pool with shared storage correctly is the step that unlocks the platform's high availability and flexibility features.

## Resource Pool Basics

A XenServer resource pool is a cluster of hosts managed as a unit. One host is the pool master; the others are slaves. All hosts in a pool must share at least one Storage Repository (SR) — this shared SR is what enables XenMotion and High Availability.

Requirements for pool membership: hosts must have the same CPU vendor (all Intel or all AMD — you can't mix) and ideally the same CPU model for live migration compatibility. Different models work for migration with CPU masking, but identical CPUs eliminate this constraint entirely.

## Setting Up Shared iSCSI Storage

The most common shared storage type for SMB XenServer deployments is iSCSI. The process:

1. Configure your iSCSI target (Dell SC Series, Synology, Starwind, or similar) with a dedicated LUN for XenServer shared storage
2. Set up a dedicated storage network on each XenServer host (dedicated VMkernel/bond for storage traffic, separate VLAN)
3. In XenCenter, right-click the pool and select New SR → iSCSI
4. Enter the target IP and IQN, discover LUNs, and create the SR

Once the SR is created and attached to all pool members, VMs stored on that SR can be live-migrated between hosts. VMs on local SRs cannot be XenMotion'd — they require storage migration first (moving the VM's disk to the shared SR) before live migration is possible.

## Pool Master Network Dependency

A subtle operational issue: pool management traffic routes through the pool master. If the management network on any slave host is unreachable, that host appears disconnected in XenCenter even if its VMs are running normally. VMs on disconnected hosts keep running — they just can't be managed until connectivity is restored.

This causes confusion when a management NIC fails on a slave host. VMs look gone from XenCenter's perspective, but they're actually fine. Verify using the DCUI on the host directly or by connecting to the host independently (`xe` CLI via the host's management IP).

## HA Configuration in Pools

XenServer High Availability requires shared block storage (not NFS) for the heartbeat SR. Create a small dedicated LUN (4GB is sufficient) for the HA heartbeat and configure it separately from your VM storage SR. This heartbeat SR enables the pool to detect host failures reliably and automatically restart VMs.

Enable HA at the pool level in XenCenter after the heartbeat SR is configured. Set the number of tolerated host failures — for a three-host pool tolerating one failure, the pool needs enough capacity on the remaining two hosts to run all VMs that were on the failed host. Plan your resource allocation with this headroom in mind.
