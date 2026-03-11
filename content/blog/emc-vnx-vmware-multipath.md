---
title: "EMC VNX with VMware ESXi: Multipath Configuration and What Goes Wrong"
date: 2017-12-05
tags: ["VMware", "ESXi", "EMC", "Storage", "SAN", "iSCSI"]
---

EMC storage arrays (VNX and VNXe in particular) were a common pairing with VMware in the 2015-2019 era for mid-market organizations. They're capable arrays with good VMware integration, but the integration requires correct multipath configuration on the VMware side to behave reliably. Getting it wrong produces intermittent storage errors that are difficult to reproduce and easy to blame on the wrong component.

## ALUA and Path Selection

EMC VNX arrays use ALUA (Asymmetric Logical Unit Access) for path management. ALUA allows arrays to present multiple paths to a LUN but designate some as "optimized" (through the owning storage processor) and others as "non-optimized" (through the non-owning processor).

The VMware path selection policy needs to be set correctly for ALUA arrays. The appropriate policy for EMC VNX is `Round Robin (VMware)` or the `EMC_ALUA_FAILOVER` PSP (Path Selection Plugin) if using EMC's PowerPath/VE. With the default `Most Recently Used` policy on an ALUA array, you may end up using non-optimized paths, which doubles the latency of every I/O operation (it has to cross the backend interconnect between storage processors).

Check and correct path selection policy per datastore in ESXi:

```
esxcli storage nmp device list
esxcli storage nmp psp roundrobin deviceconfig set --type=iops --iops=<n> --device=<naa.xxx>
```

For EMC arrays specifically, setting the Round Robin IOPS limit to 1 is often recommended — it routes I/O to the next path after each operation rather than waiting for a count threshold, which balances load more aggressively across paths.

## Multipath Verification

Every production ESXi host connecting to SAN storage should have at least two active paths to each LUN. Verify this:

```
esxcli storage nmp device list | grep -A 10 "naa.<your-lun-id>"
```

Look for multiple paths, each showing `State: active`. A single path means no failover protection — a NIC or switch port failure will cause I/O errors and potentially crash VMs.

For iSCSI specifically (as opposed to Fibre Channel): each ESXi host should have two VMkernel ports on the iSCSI network, binding to different physical NICs, connecting to different iSCSI target ports on the array. The software iSCSI initiator in ESXi needs to have both VMkernel ports bound to it. This is done in vCenter under the host's storage adapter configuration — a step that's easy to skip when you're in a hurry.

## The VNX LUN Trespassing Problem

On EMC VNX arrays without ALUA properly configured, you can encounter "LUN trespassing" — where I/O to a LUN arrives on the non-owning storage processor, causing the LUN to transfer ownership. Repeated trespassing causes performance degradation and generates EMC alerts.

The fix is ensuring VMware is using the correct path selection policy (optimized paths go to the owning SP), and that the array's host registration correctly identifies the host as ALUA-capable. Check the Unisphere management console for trespass event logs if you're seeing unexplained storage performance variability.

## EMC Unisphere Host Registration

When connecting ESXi hosts to VNX storage, register the host in Unisphere with its correct host type. For VMware hosts using the software iSCSI initiator, the host type should be "VMware" which configures the appropriate ALUA behavior. Registering as a generic UNIX or Windows host causes the array to miss the ALUA-specific optimizations.

If you inherit an environment where hosts were registered incorrectly, you can change the host type in Unisphere and the change takes effect on the next host reconnect — but the LUN configuration may need to be resynced. Coordinate a maintenance window for this kind of change.

## PowerPath/VE vs. Native PSPs

EMC's PowerPath/VE (Virtual Edition) is a premium multipathing plugin for VMware that replaces the native ESXi path selection with EMC's own management layer. It offers better load balancing, path health monitoring, and EMC support for path issues.

For environments with large numbers of LUNs or heavy I/O, PowerPath/VE is often worth the additional licensing. For smaller deployments with moderate I/O, the native VMware PSPs configured correctly produce adequate results. Know which one your environment is using — the troubleshooting paths are completely different.
