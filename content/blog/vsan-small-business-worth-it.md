---
title: "vSAN for Small Business: Is the Complexity Worth It?"
date: 2018-07-31
tags: ["VMware", "vSAN", "Storage", "HCI", "SMB"]
---

VMware vSAN entered the mainstream around 2016 as a hyperconverged storage option — using local drives in ESXi hosts as a shared storage pool, eliminating the need for a dedicated SAN or NAS. For small businesses, the pitch is appealing: consolidate compute and storage, reduce hardware complexity, and lower TCO. The reality is more nuanced.

## How vSAN Works

vSAN pools the local disks across cluster members into a shared datastore. Each ESXi host contributes its locally attached disks to the pool, and vSAN distributes VM data across hosts according to a storage policy that specifies redundancy (number of failures to tolerate).

With a standard policy of FTT=1 (Failure to Tolerate = 1), each VM's data is striped across at least two hosts with a witness component on a third. Losing one host doesn't lose data — the other hosts have copies. This mirrors what a SAN provides, but without dedicated storage hardware.

## The Minimum Requirements

vSAN requires a minimum of three hosts for an FTT=1 policy (two data copies, one witness). Each host needs at minimum one SSD for the cache tier and one additional disk for the capacity tier (All-Flash vSAN uses only SSDs; hybrid uses SSDs for cache, HDDs for capacity).

The minimum configuration for All-Flash vSAN: 3 hosts × (1 NVMe/SSD cache + 1 SSD capacity) = 6 drives total. This provides adequate performance for most SMB workloads but is the absolute floor — real deployments generally have more capacity drives per host.

Network requirements are strict: 10GbE minimum for the vSAN network (not shared with management or VM traffic), low latency, and consistent connectivity across all hosts. vSAN is more sensitive to network quality than traditional iSCSI or NFS storage because writes must be acknowledged by multiple hosts before returning success.

## Where vSAN Makes Sense for SMB

**Remote office/branch office (ROBO)**: A two-node vSAN cluster with a witness VM at headquarters is purpose-built for small sites. Provides local storage redundancy without a dedicated storage array at each location.

**HCI refresh cycles**: Organizations replacing aging separate compute and storage hardware. If you're buying new servers anyway, the incremental cost of vSAN-capable drives may be less than buying a separate storage array.

**Environments with 3-4 hosts and minimal storage complexity**: Simple shared storage needs where a dedicated SAN is overkill but you want the HA capabilities that shared storage enables.

## Where vSAN Doesn't Make Sense

**Existing functional SAN/NAS**: If you have a storage array that's meeting your needs, replacing it with vSAN requires significant migration effort for questionable benefit.

**High storage-to-compute ratios**: vSAN storage scales with hosts. If you need 100TB of storage but only 2 hosts worth of compute, vSAN forces you to add more compute hosts than you actually need just to get storage capacity — inefficient and expensive.

**Environments without 10GbE**: Running vSAN over 1GbE is technically possible but produces poor performance and is not recommended for production. If your network infrastructure is 1GbE throughout, the networking upgrade cost needs to factor into the vSAN business case.

**Limited operational expertise**: vSAN has its own operational model, health monitoring, and troubleshooting tools. The vSAN Skyline Health plugin provides guidance, but there's a learning curve. Environments without dedicated VMware expertise may find vSAN operational complexity a burden.

My honest guidance: for SMB deployments with 3-4 hosts starting fresh and budget to do it right (10GbE, all-flash), vSAN is worth considering. For environments with existing storage infrastructure, the bar for switching is high. And regardless of deployment model, take the vSAN Design and Sizing Guide seriously — undersizing vSAN hardware produces a bad experience that's hard to attribute correctly.
