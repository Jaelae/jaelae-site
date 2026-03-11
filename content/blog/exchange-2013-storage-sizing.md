---
title: "Exchange 2013 Storage: Disk Layout, Sizing, and What Goes Wrong"
date: 2014-05-09
tags: ["Exchange", "Exchange 2013", "Storage", "Sizing"]
---

Exchange 2013 was designed to tolerate slower, cheaper storage better than any previous version. Microsoft optimized it heavily for JBOD (Just a Bunch of Disks) in large enterprise deployments. For mid-size organizations, that changes the calculus around storage architecture — but only if you understand what the optimization actually buys you.

## Separate OS, Database, and Logs — Always

The most fundamental storage rule in any Exchange deployment: keep the operating system, mailbox databases, and transaction logs on separate volumes. This isn't optional.

Transaction logs are write-sequential and grow continuously until truncated by backup (or by circular logging, though circular logging disables point-in-time recovery and should only be used when you have a DAG). If logs share a volume with the OS and that volume fills up, Exchange will dismount affected databases to protect data consistency. Logs on their own dedicated volume keeps this failure contained.

Databases are read/write-intensive and generate significant random I/O during peak hours. On the same volume as the OS, database I/O competes with pagefile activity and OS operations. On dedicated storage, you can properly size the volume and IOPS independently.

The common "do it wrong" configuration I encounter in small businesses: SQL, Exchange databases, logs, and the OS all on C:\. It works until it doesn't.

## Right-Sizing Exchange Storage

Microsoft provides the Exchange Server Role Requirements Calculator — a downloadable spreadsheet that takes your mailbox count, average mailbox size, message profile, and required database copies as inputs and outputs disk sizing, IOPS requirements, and hardware recommendations. Use it. Don't guess.

For typical mid-size organizations (300-800 mailboxes, 2-3 GB average mailbox size, 3-year retention), you'll end up with somewhere between 2TB and 8TB of database storage, depending on database copy count. The IOPS requirement is often surprisingly modest — Exchange 2013's caching and I/O optimization means you need far less spindle performance than Exchange 2010 required for equivalent user loads.

## RAID vs. JBOD

Large Microsoft deployments use JBOD because the DAG provides redundancy at the database level, making disk-level RAID redundant (and expensive). For smaller organizations without the server count to run large DAGs, RAID 10 on the database volumes remains appropriate. It protects against single-disk failure without requiring database-level failover.

RAID 5 was common in older Exchange designs and is specifically not recommended by Microsoft. The write penalty on RAID 5 impacts Exchange I/O patterns disproportionately. If you're inheriting an older environment still on RAID 5 for the Exchange databases, it's worth documenting as a risk.

## The Transaction Log Accumulation Problem

Transaction logs in Exchange 2013 are 1MB files, generated continuously. With circular logging disabled (the correct configuration for any environment with backups or DAG copies), they accumulate until truncated. Truncation happens after a successful backup that covers the log stream.

I've seen environments where a backup had been silently failing for weeks and nobody noticed until the log drive filled up and took down every database on the server simultaneously. Monitor your log drive space and your backup job success daily. Alert at 70% and 85% capacity. By the time you're at 95%, you're already in an incident.

Set up a regular `Get-MailboxDatabase -Status | Select Name, DatabaseSize` check as part of your weekly operations review. Trend the numbers. Databases that grow faster than expected are a signal worth investigating — usually runaway archive mailboxes or a retention policy that isn't working as intended.
