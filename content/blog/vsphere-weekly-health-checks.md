---
title: "VMware vSphere Health Checks: Building a Weekly Operational Routine"
date: 2020-02-18
tags: ["VMware", "vSphere", "Operations", "Monitoring", "Health Checks"]
---

The best VMware environments I've managed didn't stay healthy by accident — they had a consistent operational review process that caught problems before they became incidents. A weekly health check routine doesn't need to be elaborate, but it does need to be consistent.

## The 15-Minute Weekly Review

Every Monday morning before business hours, I run through a standard set of checks that take about 15 minutes in a well-instrumented environment. Here's what's in it:

**Backup job status**: Check the previous week's backup jobs — did everything complete successfully? Are there any VMs not covered by a backup policy? Backup failures are the check that gets skipped most often in busy environments, and it's the most consequential one to miss.

**Datastore space**: Review free space across all production datastores. Alert if any are below 25% free. Look at the trend — if a datastore lost 5% capacity last week, understand why before it becomes critical.

**Snapshot inventory**: Check for any VMs with active snapshots older than 48 hours. Old snapshots are a consistent source of performance problems and unexpected capacity consumption.

**Host health**: Any hosts with hardware alarms (failed disk, degraded RAID, memory errors)? vCenter hardware status panels pull from iDRAC/iLO. A disk failure in the RAID backing an ESXi host's local storage needs attention before the second disk fails.

**vCenter and NSX alarms**: Review the current alarm list in vCenter. Acknowledge alarms that are known/expected, investigate anything new.

**DRS and HA status**: Confirm all hosts are connected, HA-enabled, and DRS is running. One disconnected host in a cluster is easy to miss in daily operations.

## The Monthly Review

Beyond the weekly checks, a monthly review covers:

**Certificate expiration**: Check all VMware certificates (vCenter, ESXi hosts, NSX components, backup software certificates). Anything expiring within 90 days goes on the action list.

**ESXi patch status**: Review vSphere Update Manager compliance for ESXi hosts. Are any critical patches outstanding? VMware releases security advisories that sometimes warrant expedited patching.

**Capacity trend review**: Export monthly averages for CPU, memory, and storage across the cluster. Update the capacity planning trend spreadsheet. Does the projection to 80% resource utilization change significantly from last month?

**Service account audit**: Review which accounts have vCenter access, verify service accounts are still needed, check that passwords haven't expired for service accounts with fixed credential configurations.

**Backup restore test**: Restore at least one Tier 1 VM from backup to verify restorability. Rotate through your critical VMs over the course of the year so everything gets tested.

## Turning the Routine Into a Runbook

Document the weekly and monthly checks as a runbook with explicit steps, expected outputs, and escalation criteria. A runbook enables the routine to survive personnel changes — a new team member can execute it correctly without institutional knowledge.

The runbook should include: what to check, how to check it (commands, URLs, menu paths), what "healthy" looks like, and what to do when something isn't healthy (escalation path, who to notify, initial troubleshooting steps).

For PowerCLI-based checks, script the data collection and output it to a standard format. A scheduled script that emails a health summary report every Monday morning means the routine happens even when someone is out sick.

## The Alert That Replaces the Check

A well-instrumented environment eventually replaces manual checks with automated alerts. But getting there requires knowing what to alert on — which comes from doing manual checks long enough to understand what normal looks like. Build the manual routine first, then automate the parts that are clearly defined and repeatable.

The goal is a state where your weekly health check primarily reviews automated alert outputs rather than manually running queries — and the occasional anomaly that automated checks miss gets caught by the human review pass.
