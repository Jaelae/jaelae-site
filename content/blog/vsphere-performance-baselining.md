---
title: "Performance Baselining in vSphere: What to Measure and When to Act"
date: 2020-01-15
tags: ["VMware", "vSphere", "Performance", "Monitoring", "Operations"]
---

"The VMs are slow" is the most common call I get about VMware environments, and it's almost never useful information on its own. What I need to know — and rarely have if nobody's been tracking it — is what "slow" means relative to baseline. Performance baselining is the practice of capturing normal behavior metrics so that abnormal behavior is recognizable as such.

## What to Baseline

For a vSphere environment, the metrics worth baselining fall into four categories:

**Host CPU**: Average ready time per host (time VMs spend waiting for CPU to be available) and per-host CPU utilization during peak business hours. Healthy environments show CPU ready times consistently below 5% per VM. Sustained CPU ready above 10% indicates overcommitment — more vCPUs are demanding execution than physical cores can serve simultaneously.

**Host Memory**: Available memory per host and memory balloon/swap activity. Ballooning (guests returning memory to the hypervisor under pressure) in small amounts is normal. Consistent high ballooning or any disk-based swap means your environment is genuinely memory-constrained.

**Datastore VMFS Performance**: Read and write latency per datastore, and IOPS consumed. The target thresholds: latency below 10ms average for production workloads, with capacity to absorb peaks. Datastores consistently at 20ms+ average latency under normal load need attention.

**Network Throughput**: Transmit and receive rates per host, per pNIC. Establishing a traffic pattern baseline helps identify anomalies — a sudden spike in outbound traffic from a normally quiet VM is worth investigating.

## Building the Baseline

vCenter's built-in performance charts (Monitoring > Performance in the vSphere Client) let you review historical metrics for hosts, datastores, VMs, and networks. The default retention in vCenter is approximately five minutes granularity for recent data, rolling up to hourly and daily averages as data ages.

For building a genuine baseline, I export daily averages for peak business hours over a rolling two-to-four week period. PowerCLI makes this scriptable:

```powershell
$stat = Get-Stat -Entity (Get-VMHost "esxi-host-01") -Stat "cpu.usage.average" -Start (Get-Date).AddDays(-30) -IntervalMins 30
$stat | Export-Csv baseline-cpu.csv
```

After four weeks, you have enough data to characterize what Monday morning looks like versus Saturday afternoon, what the normal variation range is during business hours, and where the current environment sits relative to a comfort threshold.

## Setting Meaningful Alerts

Most VMware environments either have no alerting or have the default vCenter alarms that trip constantly (because they're not calibrated to the environment's actual behavior). Neither is useful.

The approach I prefer: set alert thresholds at 80% of your headroom ceiling, not at 100% of capacity. If your hosts normally run at 40% CPU average and you have capacity for 70% before performance degrades meaningfully, alert at 65% — giving you a warning window before you're actually impacted.

For datastores, alert on free space percentage (not absolute space). A 10TB datastore at 5% free is a worse situation than a 500GB datastore at 10% free, even though the latter has less absolute space remaining.

## Capacity Planning from Baseline Data

Baseline data enables capacity planning. If you export monthly average utilization for your cluster over a year and plot it, you can project when you'll hit resource boundaries — usually 12-18 months ahead with reasonable confidence.

For an SMB client with a four-host cluster, a simple monthly export of host CPU and memory utilization and datastore capacity trends, plotted in a spreadsheet, gives a defensible answer to "do we need to add hardware?" before the environment is actually struggling.

This kind of data is also invaluable when something does go wrong. When you get "the VMs are slow," you can show whether CPU ready time has increased, whether a datastore's latency has trended upward over the past month, or whether a specific VM's resource consumption changed significantly. Data beats speculation every time.
