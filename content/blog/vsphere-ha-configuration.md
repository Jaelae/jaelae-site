---
title: "vSphere HA: Configuring It Right for Small Environments"
date: 2017-01-17
tags: ["VMware", "vSphere", "HA", "High Availability", "Clustering"]
---

vSphere High Availability is one of the primary reasons to run VMware over standalone hypervisors. When configured correctly, it automatically restarts VMs on surviving hosts when a host fails — transparently, without requiring administrator action. But "configured correctly" is doing a lot of work in that sentence.

## What HA Actually Does (and Doesn't)

HA monitors host availability through both network heartbeats (between hosts via the management network) and datastore heartbeats (via shared storage). When a host stops sending heartbeats, HA waits through a configurable failure detection window, then attempts to restart the affected VMs on other hosts in the cluster.

What HA does not do: protect against guest OS failures (that's vSphere Application Monitoring or third-party tools), protect against storage failures (if your datastore goes offline, the VMs on it can't restart elsewhere), or provide zero-downtime failover (VMs restart from their last checkpoint, which means a brief outage during host failure events).

It also doesn't protect against total cluster failure — if all your hosts go down simultaneously, HA can't help you.

## Admission Control: Don't Disable It

Admission control is the mechanism that ensures the cluster maintains enough capacity to restart VMs from the number of failed hosts you've specified. The most common misconfiguration I encounter: admission control disabled because it was blocking VM power-ons, and nobody knew how to fix it properly.

When admission control blocks a VM from powering on, it's telling you the cluster doesn't have enough headroom to guarantee restarts if a host fails. The right fix is either to add resources or to explicitly accept a lower HA guarantee — not to disable the check entirely.

For a three-host cluster, the standard configuration is to tolerate one host failure. That means HA reserves enough capacity to restart all VMs from one host on the remaining two. If your cluster is running at 90% utilization across all three hosts, you don't have 33% spare capacity — admission control will correctly prevent new VMs from powering on.

## Datastore Heartbeating

HA uses datastore heartbeating as a secondary signal to distinguish host failures from network isolation. If a host stops sending network heartbeats but continues writing to a datastore heartbeat file, HA considers the host isolated (not failed) and may not restart its VMs on other hosts.

In vSphere 6.x, HA automatically selects two datastores for heartbeating from available shared datastores. You can override this selection in the cluster settings. Best practice: ensure the heartbeat datastores are on the most reliable storage in your environment.

For clusters with only one shared datastore: configure HA to "Leave powered on" for the datastore heartbeat policy, otherwise a brief storage hiccup can cause HA to incorrectly classify running hosts as isolated and trigger unnecessary VM restarts.

## VM Restart Priorities

Not all VMs are equally important. HA lets you assign restart priorities — typically High, Medium, Low, and Disabled — to individual VMs or groups. VMs with High priority restart first on surviving hosts, before Medium and Low priority VMs consume capacity.

In practice, map this to your RTO requirements:

- **High**: Domain controllers, critical application servers, DNS servers
- **Medium**: File servers, general application servers
- **Low**: Development/test VMs, non-critical workloads
- **Disabled**: VMs that shouldn't auto-restart (intentionally stopped for maintenance, or dependencies that make auto-restart harmful)

Get the restart priority assignments right before an incident, not during one.

## Testing HA

Testing HA before you need it is not optional. The process: take a host that's running some VMs, put it in maintenance mode (this forces VMs off to other hosts, which is different from a failure), then power it off physically, then remove maintenance mode. Now the host is genuinely unreachable. Watch whether HA detects the failure and restarts the VMs on other hosts within your expected timeframe.

Do this during a maintenance window with non-critical test VMs first. Verify that your restart priority assignments are working as expected. Document the observed restart times.

An HA configuration you've never tested is a configuration you don't actually understand.
