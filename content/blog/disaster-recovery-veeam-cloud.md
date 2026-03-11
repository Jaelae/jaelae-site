---
title: "Disaster Recovery Without a DR Site: Using Veeam and Cloud Storage"
date: 2019-09-11
tags: ["VMware", "Veeam", "Disaster Recovery", "Cloud", "Azure", "AWS"]
---

Traditional disaster recovery required a physical secondary site — a co-location facility or branch office with matching hardware that could run your workloads if your primary site failed. For small and mid-size businesses, the cost and complexity of a traditional DR site is often prohibitive. Cloud-based DR, using Veeam with Azure or AWS as the recovery target, has made meaningful DR achievable for environments that previously had none.

## The Cloud DR Model

The core idea: back up your VMware VMs to the cloud (Azure Blob, AWS S3, or a cloud-hosted Veeam repository) and use cloud compute to run those workloads if your primary site is unavailable. In a declared disaster, your VMs boot in the cloud and users access them over VPN or redirected DNS.

This isn't zero-downtime failover — RTO is typically measured in hours, not seconds. But for organizations whose alternative is "no DR at all," hours of recovery time is substantially better than days of rebuilding from scratch.

## Veeam Cloud Connect

Veeam Cloud Connect is a service layer that lets Veeam Backup & Replication at your primary site send backup data to a Veeam service provider's infrastructure (or your own cloud-hosted Veeam Cloud Connect gateway). The backup traffic is WAN-optimized and encrypted in transit.

For SMB use cases, you typically engage a Veeam Cloud Connect service provider — several specialize in this model and include the gateway infrastructure in their service. The economics are often better than building your own cloud infrastructure for DR.

Alternatively, Veeam Backup for Microsoft Azure (or AWS) lets you restore Veeam backups directly to cloud VMs from S3 or Azure Blob repositories.

## RPO and RTO Reality

**RPO** (how much data you lose): Determined by your backup frequency. If you back up to cloud daily, your RPO is up to 24 hours. Hourly offsite copies bring RPO to 1 hour but consume significantly more bandwidth and cloud storage.

**RTO** (how long recovery takes): For cloud restore, RTO depends on how much data needs to be restored before VMs can start. Veeam's Instant VM Recovery to cloud (starting VMs directly from the backup copy while data is hydrated in the background) significantly reduces RTO for critical VMs.

For a 50-VM environment with RPO of 4 hours and RTO of 4 hours: expect to pay approximately $500-800/month in cloud storage (depending on deduplication ratios and data change rates). This is substantially less than co-location costs for equivalent hardware.

## The Network Consideration

Cloud DR is most useful when users can access cloud-hosted workloads. This requires either:

- **VPN from all user locations to the cloud DR environment**: Viable if your users are in offices that have stable internet and VPN infrastructure
- **Published applications through a cloud-hosted gateway** (Citrix, Remote Desktop Gateway, etc.): Applications presented through a browser or client, not requiring full network connectivity
- **Hybrid access for SaaS-first environments**: If most workloads are already SaaS (Office 365, Salesforce), DR only needs to cover the subset of on-premises workloads that remain

Plan the user access model as part of DR design, not as an afterthought.

## Testing DR Annually

Cloud DR that's never been tested is cloud DR that may not work. Veeam supports DR testing via sandboxed restore — bringing up recovered VMs in an isolated network environment to verify they boot and application functions correctly, without affecting production or requiring actual VPN connectivity.

Run this test at least annually. Document the results: which VMs came up cleanly, which required intervention, actual recovery time for your critical workloads. Use the test results to refine your DR plan and close gaps before they matter in a real event.

The organizations I've seen handle real disasters well were the ones who treated their DR plan as a living document tested regularly — not a document filed away after initial creation.
