---
title: "Starting with ESXi on Bare Metal: Hardware Selection for Small Business"
date: 2016-02-08
tags: ["VMware", "ESXi", "Hardware", "SMB", "Virtualization"]
---

The question I got asked most often when starting ESXi deployments for small and mid-size businesses: what hardware should we buy? The honest answer is "it depends," but there are enough consistent patterns across successful SMB deployments to give useful guidance.

## The VMware HCL Is Not Optional

The VMware Hardware Compatibility List (HCL) exists because ESXi relies on specific, validated driver combinations for hardware components. Running ESXi on hardware that isn't on the HCL means you may encounter missing drivers at install time, or worse — drivers that appear to work but have subtle I/O issues that manifest under load.

For servers, this narrows your practical options significantly. Dell PowerEdge, HPE ProLiant, Lenovo ThinkSystem, and Cisco UCS are the workhorses in VMware environments because their components are consistently on the HCL and the vendors maintain current driver VIBs (VMware Installation Bundles). Custom-built servers or consumer-grade hardware can work, but you're on your own for troubleshooting.

For SMB deployments, I've had consistently good results with Dell PowerEdge R630/R640 (2U rack) and R730/R740 (for more storage capacity). HPE DL360/DL380 Gen9/Gen10 are equally solid. Either platform has excellent ESXi HCL coverage and reliable iLO/iDRAC remote management.

## CPU, Memory, and Storage: Rough Sizing

For a typical 3-4 host SMB cluster virtualizing 30-80 VMs:

**CPU**: Dual-socket servers with current-generation Xeon processors (E5-2600 series in the 2016 era). vCPU overcommit ratios of 4:1 to 8:1 are typical for mixed workloads. Count your total vCPU requirements across VMs, divide by your overcommit target, and size accordingly.

**Memory**: Memory is usually the binding constraint in SMB virtualization. ESXi memory overcommit features (ballooning, transparent page sharing) are less reliable with modern security hardening. Size physical RAM to approximately 80% of the total guest RAM you intend to run. For 30 VMs averaging 4GB each, that's roughly 120GB target — two hosts at 128GB each gives you N+1 redundancy.

**Storage**: Covered in detail when I get to shared storage and datastores. The quick answer: use shared storage for clustered features (vMotion, HA, DRS), size your VMDK volumes conservatively initially and expand later, and always separate OS and data disks in your VMs.

## NICs: More Than You Think You Need

Every ESXi host in a clustered environment needs at minimum four network interfaces, and typically six to eight for proper network separation:

- VM network traffic
- VMkernel management
- vMotion (dedicated)
- vSAN or iSCSI storage (if applicable)

Running everything over two NICs is technically possible but creates a brittle configuration where any NIC failure impacts multiple services simultaneously. 10GbE NICs allow you to trunk multiple network functions over fewer physical ports with proper bandwidth separation, but 1GbE with 6-8 ports is still a solid choice for budget-conscious SMB deployments.

## The Parts People Cheap Out On

Remote management (iDRAC/iLO) enterprise licensing matters. The express/free tiers limit remote console functionality and often lack virtual media capabilities. When you need to reinstall ESXi on a host at midnight and the datacenter is 45 minutes away, iDRAC Enterprise pays for itself.

Redundant power supplies are not optional in a production environment. They're cheap relative to the server cost and protect against a failure class that would otherwise require downtime.

Finally: buy servers under vendor support contracts. ESXi patches, firmware updates, and hardware driver updates are the operational foundation of a stable virtualized environment. Running unsupported hardware makes all of that harder.
