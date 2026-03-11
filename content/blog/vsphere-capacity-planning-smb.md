---
title: "Capacity Planning for Small VMware Environments Without Enterprise Tools"
date: 2020-03-09
tags: ["VMware", "vSphere", "Capacity Planning", "Operations", "SMB"]
---

Enterprise VMware environments have tools like vRealize Operations for capacity management. SMB environments running five to twenty ESXi hosts typically don't — and they don't need to. Good capacity planning is achievable with PowerCLI, a spreadsheet, and consistent measurement discipline.

## The Three Resources That Matter

For most VMware workloads, three physical resources constrain capacity: CPU, memory, and storage. Network rarely saturates in typical mixed-workload environments unless you have specific high-throughput applications.

**CPU capacity** is measured as a ratio of vCPUs to physical cores. A cluster of four dual-socket 16-core hosts gives you 128 physical cores. At a 4:1 overcommit ratio (typical for mixed workloads), you can comfortably run 512 vCPUs worth of VM allocation. Beyond that, CPU ready times start climbing and VMs feel sluggish.

**Memory capacity** is less forgiving. CPU overcommit is real — VMs that aren't actively computing aren't competing for physical cores. Memory overcommit is theoretical — every GB of VM guest RAM that's in use needs actual physical RAM behind it (or expensive disk-based swap). Size memory for your current loaded memory, not your total VM memory allocation.

**Storage capacity** has two dimensions: space and performance. Space is easy to measure. Performance (IOPS and latency) requires measurement under load. An environment that looks fine on available space but is saturating its storage IOPS budget will perform poorly in ways that are hard to diagnose without per-datastore performance metrics.

## The Baseline Spreadsheet

I maintain a monthly snapshot spreadsheet for client environments with these columns per host:

- Physical CPUs, total cores
- Total memory (GB)
- Average CPU utilization (30-day average at peak hours)
- Average memory usage (30-day average at peak hours)
- Memory balloon active (yes/no — any ballooning means memory pressure)
- Per-datastore: capacity, used, free %, average latency (ms)

One tab per month. After twelve months, trends emerge clearly. I add a simple linear projection column — "at current growth rate, we hit 80% CPU in approximately N months." This becomes the basis for hardware planning conversations.

## VM-Level Overprovisioning

A large percentage of SMB VMware capacity problems aren't actually capacity problems — they're overprovisioning problems. VMs allocated 8 vCPUs and 32GB RAM for workloads that actually use 2 vCPUs and 8GB RAM in normal operation.

Check actual resource usage versus allocation in PowerCLI:

```powershell
Get-VM | Select Name, NumCpu, MemoryGB, 
    @{N='CpuUsageMhz';E={($_ | Get-Stat -Stat cpu.usage.average -MaxSamples 30).Value | Measure-Object -Average | Select -Exp Average}},
    @{N='MemUsageGB';E={[math]::Round(($_ | Get-Stat -Stat mem.usage.average -MaxSamples 30).Value | Measure-Object -Average | Select -Exp Average * $_.MemoryGB / 100, 1)}}
```

Regularly reviewing this and right-sizing VMs that are significantly over-allocated recovers capacity without any hardware investment. I've recovered 30-40% of effective cluster capacity for clients by right-sizing a handful of over-allocated VMs.

## When to Add Hardware

The decision to add ESXi hosts or storage should be data-driven. My trigger points:

- Cluster CPU average at peak hours exceeds 65% sustained (not spike)
- Any host shows memory balloon activity during normal operation
- Any datastore sustained latency exceeds 15ms average during business hours
- Free datastore space drops below 25% on any production datastore

These thresholds give a comfortable lead time for hardware procurement and installation without cutting it close. Waiting until you're at 90% of a resource limit means you're adding hardware reactively, under pressure, with limited options.

Planning 6-12 months ahead based on trend data is the goal. It's not complicated — it just requires consistent measurement.
