---
title: "Rubrik Backup for VMware: Why We Replaced Veeam"
date: 2018-06-11
tags: ["VMware", "Rubrik", "Backup", "Veeam", "Data Protection"]
---

Rubrik entered the backup market around 2014-2015 with a fundamentally different architecture than traditional backup software: a purpose-built hardware/software appliance with a policy-driven, SLA-based management model. When I first deployed it for a client in 2017, it solved problems I'd been working around in Veeam for years. It also introduced some new operational considerations.

## The SLA Policy Model

Traditional backup software (Veeam included) requires you to define jobs: which VMs are included, when the job runs, how many restore points to keep, what storage target to use. Managing dozens of backup jobs across hundreds of VMs is administratively heavy and creates subtle coverage gaps when new VMs get deployed and nobody remembers to add them to the right job.

Rubrik's model is SLA policies. You define policies at the cluster or tag level: "Gold tier gets hourly backups with 7-day retention, replication to DR, and 30-day archival." VMs assigned to the Gold SLA policy automatically comply with those terms. A new VM deployed into a vSphere folder assigned to Gold is automatically protected without manual job modification.

This sounds simple, and it is — once you've designed your SLA tiers thoughtfully. The design phase requires actually thinking about what your organization's recovery point and recovery time objectives are for different workload categories. Most organizations I've worked with hadn't formally defined those requirements before; Rubrik's model forces the conversation.

## The vCenter Integration

Rubrik integrates with vCenter to discover and protect VMs, leveraging VMware's VADP (vStorage APIs for Data Protection) framework. Backups are application-consistent for Windows VMs via VSS integration and for Linux VMs via pre/post scripts or application-specific connectors.

The integration means Rubrik sees your entire vSphere environment through the vCenter API. Newly deployed VMs appear in the Rubrik console automatically. VMs that are deleted from vCenter are automatically removed from protection schedules after a configurable grace period.

One thing to configure early: exclude template VMs and VMs in specific folders from SLA assignments if they don't need protection. Without explicit exclusions, discovery can attach SLA policies to VMs that don't need backup, consuming capacity on test/dev workloads.

## Instant Recovery vs. Live Mount

Rubrik offers two rapid recovery options. **Instant Recovery** restores a VM back to vSphere by migrating data from the Rubrik appliance to your production datastores — VMs come online quickly and storage migration happens in the background. **Live Mount** presents a VM directly from the Rubrik appliance's storage, running it from the backup copy immediately without any data movement. Live Mount is faster for immediate access but performs at Rubrik appliance I/O speeds (adequate for testing and short-term recovery, not ideal as a permanent state).

For compliance and DR testing, Live Mount is genuinely impressive: a 500GB VM can be running from its backup copy in under five minutes. This changes how you think about DR exercises — instead of a quarterly all-hands test that disrupts production, you can do monthly Live Mount tests of individual applications with minimal impact.

## What Veeam Still Does Better

Rubrik isn't a clear winner across every dimension. Veeam has stronger agent-based backup support for physical machines and cloud workloads. Veeam's restore granularity for individual files and items from agent backups is more mature. And Veeam's cost model is more flexible for organizations with small VM counts or varying workload types.

For environments that are heavily VMware-centric with 100+ VMs, strong SLA compliance requirements, and DR replication needs, Rubrik's model consistently outperforms Veeam operationally. For smaller environments or those with significant physical server or cloud workload components, Veeam remains competitive.

The decision isn't universal — it depends on your environment composition and how much operational overhead you want to spend managing backup jobs vs. paying for an appliance that manages itself.
