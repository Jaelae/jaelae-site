---
title: "When to Add vCenter: The Decision That Changes Everything"
date: 2016-07-11
tags: ["VMware", "vCenter", "vSphere", "SMB"]
---

Running standalone ESXi hosts is a common starting point for small businesses. It's simple, cost-effective, and covers a lot of ground. But there's a clear inflection point where standalone hosts start to limit you — and adding vCenter is the change that unlocks the rest of the platform.

## What You Can't Do Without vCenter

Standalone ESXi hosts are islands. Each one is managed independently, each has its own configuration, and there's no coordination between them. The list of features that require vCenter:

- **vMotion (live migration)**: Moving running VMs between hosts without downtime requires vCenter to orchestrate the migration.
- **vSphere HA**: Automatic restart of VMs on surviving hosts when a host fails.
- **vSphere DRS**: Automated load balancing across cluster members.
- **Distributed vSwitches**: Centralized, consistent network configuration across hosts.
- **vSphere Update Manager (VUM/VLCM)**: Centralized patch management for ESXi hosts.
- **Host profiles**: Template-based configuration consistency enforcement.

Without vCenter, manual live migration, automated failover, and any of those features simply don't exist. Host failures mean VM downtime until you manually power them on elsewhere.

## The Cost Reality for SMBs

vCenter historically had significant licensing costs, which pushed many SMBs to defer it. The VMware Essentials Plus kit offered a bundle of vCenter and ESXi licensing for smaller environments (typically 3-host limit) at a more accessible price point. For organizations beyond that scale, the discussion becomes about ROI — how much is automated failover worth versus the cost of the vCenter license?

For organizations with 20+ VMs running business-critical workloads, the answer is almost always that vCenter pays for itself the first time HA restarts a critical server automatically at 2am instead of someone getting paged.

## VCSA vs. Windows vCenter

By the time most SMB deployments I worked on in 2016 were adding vCenter, the vCenter Server Appliance (VCSA) was mature enough to recommend over the Windows-based installer. The VCSA is a purpose-built Linux appliance — no Windows license required, simpler deployment, and VMware was clearly investing more in the appliance going forward.

Deploy the VCSA on your existing ESXi infrastructure using the OVA/installer. Minimum resources: 2 vCPUs, 10GB RAM for the smallest supported deployments (though 4 vCPUs and 16GB RAM is comfortable for up to 100 hosts and 1000 VMs). The VCSA appliance includes its own embedded PostgreSQL database — no external SQL server required for small-to-medium deployments.

## The Single Point of Failure Discussion

vCenter itself is a potential single point of failure. If vCenter goes down, your VMs keep running, vMotion and HA continue operating (they have enough cached state to function independently for a while), but you lose centralized management visibility and the ability to make configuration changes.

For most SMBs, a single VCSA with regular snapshots or backups is acceptable. For environments where vCenter unavailability for even an hour is business-impacting, the VCSA supports a native High Availability configuration (vCenter HA, with active/passive failover) starting in vSphere 6.5.

My standard recommendation: deploy VCSA, back it up daily (snapshot or file-level backup of the VCSA VM), and document the manual recovery procedure for the worst case. For most SMBs, that's sufficient. Add vCenter HA when the organization matures to where its loss is a P1 incident.

## Don't Skip the Platform Services Controller Complexity

vSphere 6.0 introduced the Platform Services Controller (PSC) as a separate component handling Single Sign-On (SSO), certificate management, and licensing. Some guides recommended deploying it as a separate VM from vCenter. By 6.5 and especially 6.7, VMware strongly recommended the embedded PSC model — PSC functionality built into the VCSA itself.

If you're starting fresh, use embedded PSC/VCSA. If you're inheriting an environment with external PSC VMs, document it carefully and plan the converge-to-embedded migration when you have a maintenance window.
