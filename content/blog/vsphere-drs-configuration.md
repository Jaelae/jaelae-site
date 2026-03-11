---
title: "vSphere DRS: From Manual to Fully Automated and the Lessons in Between"
date: 2017-09-14
tags: ["VMware", "vSphere", "DRS", "Load Balancing", "Clustering"]
---

Distributed Resource Scheduler (DRS) is vSphere's automated workload balancing engine. When configured appropriately, it silently keeps your cluster balanced without admin intervention. When misconfigured, it generates a flood of migration recommendations that overwhelm administrators or triggers constant VM movements that impact performance. Here's how to dial it in.

## The Three DRS Automation Levels

**Manual**: DRS generates recommendations but doesn't act on them. You review and approve each suggested vMotion. Good for learning what DRS is recommending before trusting it with automation, or for environments where unplanned VM movements are operationally problematic.

**Partially Automated**: Initial placement of VMs when powered on is automated; ongoing balance recommendations are manual. Useful transition state.

**Fully Automated**: DRS both places VMs initially and moves them to rebalance the cluster without prompting. The standard production configuration for most environments.

I start new cluster deployments in Manual for the first two weeks to understand DRS behavior, then move to Fully Automated once the recommendation pattern looks sensible. Most SMB environments are adequately sized that DRS recommendations are infrequent and appropriate.

## The Migration Threshold

The migration threshold slider (1 through 5, Conservative to Aggressive) controls how imbalanced the cluster needs to be before DRS acts. At level 1 (Conservative), DRS only migrates when the imbalance is very significant. At level 5 (Aggressive), it migrates frequently to keep everything tightly balanced.

The default (3) is appropriate for most environments. Moving toward Aggressive causes excessive VM migrations that consume network bandwidth and CPU during migration and can actually impact VM performance. Moving toward Conservative allows more imbalance to persist.

Common mistake: setting aggressive migration thresholds in an environment where migrations themselves are expensive (slow storage, constrained network) hoping to keep the cluster balanced. The migrations then become the problem.

## DRS Rules: Use Sparingly

DRS affinity and anti-affinity rules let you specify that certain VMs must or should run together, or that certain VMs must not run together. These are powerful but frequently misused.

**Anti-affinity rules** (VMs should run on different hosts) make sense for: domain controller pairs (never let both DCs run on the same host), SQL Always On pairs, application servers in an HA pair.

**Affinity rules** (VMs should run on the same host) make sense for: clustered workloads that communicate intensively and benefit from local communication, licensing-constrained software.

The mistake: creating too many affinity rules and inadvertently constraining DRS's ability to balance. If you have rules specifying that VMs A, B, and C must stay together, and VMs D, E, and F must stay together, and VM G must separate from VMs A and D, you've created a constraint problem that DRS may not be able to satisfy while also balancing load. Keep rules minimal and review them periodically.

## Resource Pools: An Aside

Resource pools let you allocate cluster resources in proportional shares or absolute limits. They're frequently set up and then never touched, which is mostly fine — but misconfigured resource pools can cause confusing performance behavior.

The common issue: a resource pool with a hard CPU limit set during initial deployment to prevent a VM from consuming too much, and then nobody noticed when the workload grew. The VM hits the CPU limit, its performance degrades, and nobody connects it to the resource pool configuration.

If VMs in your cluster are performing poorly despite adequate host resources, check resource pools. `Get-ResourcePool` in PowerCLI shows limits and shares across all pools.

## DRS and Maintenance Mode

When you put an ESXi host into maintenance mode (for patching, hardware work, etc.), DRS automatically migrates VMs off that host to make the evacuation seamless. This is one of the most operationally valuable DRS behaviors and something you quickly come to rely on.

The thing to know: if DRS is in Manual mode, putting a host in maintenance mode will generate recommendations instead of automatically evacuating VMs. If you're manually patching hosts in a Manual DRS cluster, you need to approve the migration recommendations for each VM before the host will complete entering maintenance mode.
