---
title: "NSX-v to NSX-T Migration: The Playbook Nobody Gives You"
date: 2026-03-05
description: "Having led this migration at enterprise scale, here's the real playbook — including the gotchas that aren't in VMware's documentation."
tags: ["NSX", "VMware", "Networking", "Migration"]
---

If you're still running NSX-v, you already know the clock is ticking. VMware ended general support, and NSX-T (now NSX) is the path forward. But the migration documentation from VMware reads like it was written for a lab environment, not a production enterprise with hundreds of distributed firewall rules and thousands of logical switches.

I led this migration at Johnson & Johnson across a global VMware environment. Here's the playbook that actually works.

## The Pre-Migration Audit Nobody Wants to Do

Before you touch a single configuration, you need a complete inventory of your NSX-v environment. And I mean complete:

- Every logical switch and which VMs are attached
- Every distributed firewall rule, including who created it and why
- Every edge services gateway and what it's doing
- Every load balancer VIP and its health check configuration
- Every NAT rule

You'll discover rules that nobody remembers creating, edges that are serving traffic nobody knew about, and firewall policies that contradict each other. **Clean this up before you migrate.** Migrating garbage into a new platform just gives you faster garbage.

## The Migration Coordinator Tool — Use It, But Don't Trust It

VMware provides a Migration Coordinator that's supposed to automate the transition. It works reasonably well for simple topologies. For complex environments, it becomes a starting point rather than a complete solution.

We used the coordinator to migrate about 70% of our configuration automatically, then manually handled the remaining 30% — which included our most critical and complex firewall policies.

The coordinator struggles with:
- Complex service insertion chains
- Guest introspection configurations  
- Multi-tier firewall rules with nested security groups
- Edge configurations with multiple interfaces in complex routing topologies

## Parallel Operation is Your Friend

The biggest risk in any network migration is the cutover. Our approach was to run NSX-v and NSX-T in parallel for an extended period. New workloads went to NSX-T immediately. Existing workloads migrated in waves, grouped by application dependency.

This parallel operation period was longer than we planned — about four months instead of two — but it gave us the safety net to roll back individual application groups if something broke.

## The Firewall Rule Translation Problem

This is where most teams get burned. NSX-v and NSX-T handle distributed firewall rules differently. The rule processing order, the default behaviors, and the grouping constructs have all changed.

We built a validation framework that compared traffic flows before and after migration for each application group. Any deviation triggered an investigation before we proceeded. This caught several rule translation issues that would have caused production outages.

---

The NSX-T platform is genuinely better than NSX-v — better performance, better policy management, better integration with VCF. But the migration is a real project that deserves serious planning. Don't let anyone tell you it's a weekend task.
