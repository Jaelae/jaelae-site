---
title: "Planning Your Exchange 2013 to Office 365 Migration: Lessons from the Field"
date: 2014-09-30
tags: ["Exchange", "Exchange 2013", "Office 365", "Migration", "Hybrid"]
---

By late 2014, Office 365 had matured to the point where the question wasn't "should we consider it" but "when do we move." Organizations that had just completed Exchange 2013 deployments found themselves weighing a cloud migration sooner than expected. Here's what a realistic transition looked like.

## The Hybrid Configuration: Necessary Complexity

Microsoft's recommended migration path for organizations with existing Exchange infrastructure is Hybrid Configuration — a bridge state where on-premises Exchange and Exchange Online coexist, sharing namespace, free/busy, and mail routing. The Hybrid Configuration Wizard handles most of the setup, but "handles most of it" is doing a lot of work.

Prerequisites for hybrid that catch people off-guard:

**Directory Synchronization (Azure AD Connect)**: All on-premises users must be synchronized to Azure AD before mailboxes can be migrated to Exchange Online. Azure AD Connect needs to be installed, configured, and running reliably. Any AD attribute inconsistencies that Azure AD Connect's sync rules don't like will prevent affected users from synchronizing — and a user who doesn't synchronize can't have their mailbox migrated.

**Certificates on the CAS servers**: Hybrid configuration uses the existing Exchange certificate infrastructure. Your on-premises Exchange certificate must cover the namespaces used for hybrid mail flow and Autodiscover. If you have a single-SAN certificate with minimal SANs, you may need a new certificate before proceeding.

**MX record considerations**: During hybrid, you can choose to route mail through on-premises (keeping your existing filtering appliance in the path) or centralize routing through Office 365. Plan this before starting — changing it mid-migration is disruptive.

## Cutover vs. Staged Migration

For smaller organizations (under 150 mailboxes), a cutover migration — moving everyone in a single event — is often simpler than running a hybrid for months. Cutover doesn't require Exchange Hybrid Configuration; it's a direct mailbox migration to Exchange Online.

The tradeoff: cutover migrations are faster and simpler but provide no gradual transition period. All mailboxes move in one migration event (typically overnight), Outlook clients reconfigure via Autodiscover, and you're done with on-premises Exchange. For organizations with uncomplicated configurations and low tolerance for extended hybrid complexity, cutover is appropriate.

## What Doesn't Migrate Cleanly

The items that require extra attention in any Exchange to Office 365 migration:

**Public folders**: Classic public folders exist in Office 365 but with limitations compared to on-premises. Large public folder hierarchies or heavy public folder users may require a separate assessment and migration strategy.

**Third-party integrations**: Any application that connects to Exchange directly (SMTP relays, conference room booking systems, HR systems, scan-to-email appliances) needs to be redirected to Office 365 SMTP endpoints after migration. Catalog these integrations before starting.

**Archive mailboxes**: Online Archive in Exchange Online has different limits and behaviors than on-premises archive mailboxes. Understand the differences before migrating heavily-archived users.

**Retention policies and compliance holds**: Legal holds and compliance policies applied on-premises don't automatically carry over. Map your compliance requirements to equivalent Office 365 features before migration.

## The Timeline Expectation

For a 300-user organization with Exchange 2013 on-premises, a realistic hybrid migration takes 3-4 months: one month for preparation (directory sync, hybrid setup, certificate verification), one to two months for phased mailbox migration (pilot users, then departments, then everyone), and two to four weeks of hybrid operation before decommissioning on-premises servers. Rushing this timeline creates problems that drag out longer than a careful migration would have taken.
