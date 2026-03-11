---
title: "XenServer Pool Master Failure: What Actually Happens and How to Recover"
date: 2015-05-18
tags: ["XenServer", "Citrix", "High Availability", "Troubleshooting"]
---

The pool master is XenServer's single management control plane for a resource pool. All XenCenter connections, all xe CLI commands that operate across the pool, all pool-level configuration changes — they all route through the pool master. Understanding what happens when it fails, and how to recover, is essential knowledge before you put XenServer into production.

## What Happens When the Pool Master Goes Down

When the pool master host goes offline unexpectedly, the other hosts in the pool enter a degraded state. VMs that are running on slave hosts continue running — this is the important part. Workloads don't stop. But the following become unavailable until a new pool master is elected or the original recovers:

- XenCenter connections to the pool (GUI is dark)
- xe CLI pool-level commands
- New VM starts, migrations, or storage operations
- Pool configuration changes of any kind

Hosts have a heartbeat timeout — by default around 60 seconds — after which they attempt to elect a new pool master. If High Availability is configured and the host genuinely failed (as opposed to a network partition), the election happens automatically and a slave host assumes the master role.

## Network Partitions Are the Dangerous Case

The scenario that causes the most grief: the pool master is alive but unreachable from pool slaves due to a network problem (switch failure, misconfigured VLAN, someone tripping over a cable). Now you have a master that thinks it's healthy, slaves that can't reach the master, and potentially a split-brain situation if HA tries to fence hosts.

XenServer's HA uses a storage heartbeat (via a dedicated heartbeat SR, typically a small LUN on shared storage) in addition to the network heartbeat. This reduces, but doesn't eliminate, split-brain risk. Hosts that can see the storage heartbeat but not the network heartbeat will delay fencing decisions — which is usually the right behavior but can extend the degraded window.

Before deploying HA in production, test your fencing behavior deliberately in a maintenance window. Know what your environment actually does under failure conditions, not just what the documentation says it should do.

## Manual Pool Master Promotion

If your pool master fails permanently and HA doesn't automatically elect a new one (or you don't have HA configured), you'll need to manually designate a new master. Connect to any running slave host via SSH:

```bash
xe pool-emergency-transition-to-master
```

Run this on the host you want to become the new pool master. It forces that host into the master role immediately. Then point XenCenter to the new master's IP and reconnect.

Once the pool is functional again, run:

```bash
xe pool-recover-slaves
```

This instructs the other slaves to re-synchronize with the new master and rejoin the pool properly.

## Preventing Avoidable Failures

Pool master placement matters. Put your pool master on the most reliable hardware in the cluster. If you're running mixed hardware (different server generations or configs), designate the newest, most stable host as pool master.

Keep the pool master's network simple — don't put complex VLAN trunking configurations on the management NIC that the pool relies on. Management network reliability is more important than management network sophistication.

Finally: know where your pool master is before an incident. I've seen admins in a panic unable to find which of four hosts is the master because nobody documented it. `xe pool-list params=master` shows the UUID, and `xe host-list` maps UUIDs to names. Run those commands and stick the output in your runbook.
