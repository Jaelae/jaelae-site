---
title: "When Virtualization Goes Wrong: Lessons from Real Production Incidents"
date: 2020-05-27
tags: ["VMware", "Incidents", "Operations", "Lessons Learned", "vSphere"]
---

Production incidents in virtualized environments have a different character than physical infrastructure failures. The failure radius can be much larger (one host failure impacts many VMs), the failure modes are sometimes subtle, and the time pressure to recover can lead to decisions that compound the original problem. These are some of the patterns I've seen, and what I took from them.

## The Cascading Storage Failure

Scenario: A storage array controller failed and a brief (30-second) storage path failure occurred. ESXi hosts switched to alternative paths and I/O resumed. Managed Availability on several Windows VMs detected the I/O pause as a failure and triggered restarts. Those VMs came back up, but a few with open transaction log files for SQL Server came back in a crashed/dirty state.

The incident itself was recoverable — SQL instances ran their recovery processes and the databases came back clean. But the sequence of events revealed several configuration gaps: the storage multipath configuration wasn't optimized for ALUA (causing slower path failover than necessary), and the Managed Availability restart policy for the SQL VMs was set more aggressively than was appropriate for database workloads.

**Lessons**: Validate your multipath configuration under failure conditions before a real failure. Tune application-specific VM restart policies — SQL VMs that restart mid-transaction need database recovery time after the OS comes up, and aggressive HA restart policies don't account for that.

## The Snapshot That Ate the Datastore

Scenario: A test environment shared a datastore with some lower-priority production VMs. A developer took a snapshot of a large test VM "just for a few hours" before an upgrade. The upgrade took longer than expected, the snapshot wasn't deleted, and four days later the datastore was full. Several VMs were paused due to out-of-space conditions.

No data was lost, but four VMs were unavailable for approximately two hours while the snapshot was deleted and space was recovered.

**Lessons**: Don't share datastores between test and production environments if you can avoid it. Implement alerts at 75% and 85% datastore capacity. Implement a daily scheduled check for snapshots older than 24 hours and notify — or enforce — a deletion.

## The Hosts That Agreed to Disagree

Scenario: Two of four cluster hosts had a network issue that isolated them from the other two. HA correctly determined both pairs of hosts could see the storage heartbeat, so neither pair fenced the other — both continued running. VMs were running on all four hosts, but the two isolated hosts lost access to the NSX controller (which ran on the main two hosts) and their VM networking stopped working.

The VM workloads on the isolated hosts were unreachable for 45 minutes until the network was restored.

**Lessons**: NSX controller placement matters for network resilience. Ensure controllers are spread across hosts or placed on a separate, highly available host. Network partition scenarios should be explicitly tested before production deployment.

## The Upgrade That Broke Backup

Scenario: A vCenter upgrade from 6.0 to 6.7 proceeded without issues. Backup jobs ran as normal the following night and showed as successful in the backup software. Three weeks later, during a restore test, the backup images were corrupt — the backup software had been using a deprecated API path that wasn't fully compatible with vCenter 6.7, and while jobs completed "successfully," the resulting backups were not restorable.

**Lessons**: Test restores immediately after any vCenter or ESXi version upgrade. "Job successful" in backup software is not the same as "backup is restorable." Run a restore verification of at least one VM within 24 hours of completing a major upgrade.

## The Pattern Across Incidents

Looking at these and others: the root cause is rarely a single failure. It's usually a combination of a primary event (storage path failure, human action, upgrade) with a contributing configuration gap (aggressive restart policy, no datastore alerts, untested restore process) that turns a manageable event into a production outage.

Post-incident reviews should look for both the root cause and the contributing conditions. Fix both, not just the immediate trigger.
