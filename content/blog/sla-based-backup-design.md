---
title: "SLA-Based Backup Design for VMware Environments"
date: 2019-06-25
tags: ["VMware", "Backup", "SLA", "Data Protection", "RPO", "RTO"]
---

Most VMware backup configurations I inherit were built by someone who configured jobs, set a retention period, and moved on. There's no formal connection between what the backup policy does and what the business actually requires. SLA-based backup design makes that connection explicit — and it makes conversations about backup adequate or inadequate into evidence-based discussions instead of opinion debates.

## Start With RTO and RPO, Not Backup Schedule

Recovery Point Objective (RPO): how much data can the organization lose? If a system is recovered to a backup taken 24 hours ago and the RPO is 4 hours, you have a problem. RPO determines backup frequency.

Recovery Time Objective (RTO): how long can the system be unavailable? If a system restore from tape takes 6 hours and the RTO is 2 hours, you have a different problem. RTO determines backup technology (snapshot, deduplication appliance, tape, cloud) and recovery method.

These aren't technical questions — they're business questions. The answers come from conversations with application owners and business stakeholders, not from the IT team guessing what's acceptable. Document the answers for each application tier before designing any backup policy.

## Tier Your Applications

A common three-tier model for SMB environments:

**Tier 1 — Business Critical**: Domain controllers, primary database servers, core business applications (ERP, CRM). RPO: 1-4 hours. RTO: 2-4 hours. Policy: hourly backups, replication to secondary location, regular restore testing.

**Tier 2 — Business Important**: Secondary application servers, file servers, collaboration tools. RPO: 4-24 hours. RTO: 4-8 hours. Policy: daily backups, local retention 30 days, offsite copy.

**Tier 3 — Non-Critical**: Development VMs, test environments, archive systems. RPO: 24-48 hours or "rebuild from scratch acceptable". RTO: next business day. Policy: weekly backups, short local retention.

The tier assignments should be documented and agreed upon by application owners — not decided unilaterally by IT.

## Mapping Tiers to Backup Policy

Once tiers are defined, the backup policy design is straightforward:

- What backup tool covers this tier (Veeam, Rubrik, Commvault)?
- What backup frequency meets the RPO?
- What storage target (local, replicated, cloud, tape) meets the RTO given the restore technology available?
- How is compliance with the policy verified?

For Rubrik environments, this maps directly to SLA Domains — configure each tier's parameters in a named SLA Domain and assign VMs. For Veeam, create job templates per tier and document the job configuration in a runbook.

## The Restore Test Schedule

The backup policy isn't complete without a restore test schedule. At minimum:

- Tier 1 VMs: test restore quarterly, verify application comes up cleanly
- Tier 2 VMs: test restore semi-annually
- Tier 3 VMs: test restore annually or when technology changes

"Test restore" means actually restoring the VM to a separate environment (or using Live Mount) and verifying the application functions correctly — not just verifying the backup job completed. Application owners should participate in restore testing to confirm functionality from a business perspective, not just an IT perspective.

Document every restore test: date, VM restored, backup age used, restore time, pass/fail, and any anomalies. This documentation demonstrates due diligence and reveals trends (backups taking longer over time, restore success rate declining) before they become crises.

## Retention and Compliance

Retention requirements for some industries (healthcare, finance, legal) are legally mandated. Know your compliance requirements before setting retention periods. The default "30 days" configuration I find in most environments is rarely the result of a compliance analysis — it's a round number that sounded reasonable.

If your organization has documented compliance requirements, trace them to specific backup policy terms and ensure your backup system is configured to meet them. "We keep 30 days of backups" is not the same as "we meet our HIPAA data retention requirements" without having done that mapping explicitly.
