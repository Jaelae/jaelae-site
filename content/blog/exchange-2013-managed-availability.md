---
title: "Exchange 2013 Managed Availability: Friend or Enemy?"
date: 2013-12-03
tags: ["Exchange", "Exchange 2013", "Managed Availability", "Troubleshooting"]
---

Managed Availability is one of Exchange 2013's most significant operational changes — and one of the least understood. It's a built-in health monitoring and self-healing framework that watches Exchange components and takes corrective action when it detects problems. In theory, it means Exchange fixes itself. In practice, it can make troubleshooting significantly harder if you don't understand what it's doing.

## What Managed Availability Actually Does

The system operates through three components: probes that actively test functionality, monitors that track probe results over time, and responders that take corrective action when monitors trip.

Responders can do things like restart services, recycle application pools, failover databases, or bounce the entire Information Store. These actions happen automatically, without administrator intervention. On one hand, that means minor transient failures get cleaned up before users notice. On the other hand, it means you might open the event logs and find your database was failed over at 3am and you have no idea why.

## Reading the Crimson Channel Logs

Standard Application, System, and Security event logs don't tell the full Managed Availability story. The detail lives in the Crimson Channel logs under `Applications and Services Logs > Microsoft > Exchange`. 

The HighAvailability log channel is the most useful for understanding database-related actions. Look for entries around the time of any unexpected behavior. You'll find exactly which probe failed, which monitor tripped, which responder fired, and what action it took. This turns a mysterious database failover into a traceable event.

Filter the HighAvailability log by event ID 4115 (database copy state changes) and you'll build a complete picture of what happened and in what order.

## When Managed Availability Fights You

The scenario that causes the most confusion: Managed Availability keeps restarting a service or failing over a database, but the underlying problem hasn't been fixed. You restart Exchange Transport manually, watch it come up, and ten minutes later Managed Availability bounces it again because the probe is still failing.

The corrective action isn't "stop Managed Availability from doing this" — it's "figure out what probe is failing and why." Use `Get-ServerHealth -Identity <server>` in EMS to see the current health state of all monitors. Look for anything in `Unhealthy` state. That's your starting point.

If you absolutely need to suspend a responder while you work (e.g., to prevent it from interfering with maintenance), you can use `Add-GlobalMonitoringOverride` or `Add-ServerMonitoringOverride` to temporarily disable specific monitors. Document any overrides you create and remove them when done — orphaned overrides that suppress real health alerts are a future ops problem.

## Mailbox Server vs. CAS Health

Managed Availability monitors both the Mailbox and CAS roles separately. CAS health monitoring includes synthetic transaction probes that simulate OWA logins, EWS requests, and ActiveSync connections. If your CAS SSL certificate expires or an authentication configuration drifts, these probes will fail and start triggering responders.

I've seen environments where an expired internal certificate caused Managed Availability to repeatedly restart IIS components on CAS servers, which manifested as intermittent OWA disconnects for users. The event log trail through the Crimson Channel made it diagnosable in under an hour — but only once I knew where to look.

## The Bottom Line

Managed Availability is genuinely useful. It handles a large class of transient failures automatically. But when something is persistently wrong, it amplifies the noise — making what would have been a quiet service failure into a cascade of restarts and failovers. Learn to read the Crimson Channel logs early, before you need them.
