---
title: "Exchange 2013 DAG Design for Mid-Size Organizations"
date: 2013-07-08
tags: ["Exchange", "Exchange 2013", "DAG", "High Availability"]
---

Database Availability Groups in Exchange 2013 are the right answer for most organizations that need mailbox redundancy. But "you should use a DAG" is where most guidance stops, and the actual design decisions — how many members, which witness server, how to handle network configuration — are where people run into trouble.

## Two Nodes vs. Three

For mid-size organizations typically running 200 to 1,500 mailboxes, the question is almost always whether to go two nodes or three. Two nodes is cheaper and simpler. Three nodes gives you something important: the ability to lose a node and still have an automatic failover quorum.

With a two-node DAG, you're dependent on a File Share Witness (FSW) on a third server for quorum. Lose the FSW and you have a two-node cluster that can't reliably arbitrate failover. This isn't academic — I've seen it bite environments where the FSW was sitting on a CAS server that went down during the same event that took out one DAG member. Result: both database copies fenced themselves off to avoid a split-brain, and nobody could access mail until an admin manually intervened.

Three-node DAGs with a FSW are more resilient because you retain majority quorum even after losing one full node. For environments where mail availability is business-critical, the hardware cost of a third mailbox server is almost always justified.

## Lagged Database Copies

One DAG feature that's underused in smaller deployments is the lagged copy. A lagged copy is a database replica that deliberately trails the active copy by a configured time period — typically one to four hours. The value proposition is simple: if a logic error, software bug, or runaway script corrupts your active database, you have a window to recover from a clean point before the corruption replicated everywhere.

For organizations running without traditional tape or disk backup (relying purely on DAG redundancy), a lagged copy is essential. Configure one lagged copy per DAG with a 2-4 hour lag and you have a recovery option that doesn't require a lengthy restore from backup.

## Network Design

Exchange DAG documentation consistently recommends dedicated MAPI and replication networks. In practice, many smaller deployments put everything on a single network and it works fine for low message volumes. But once your databases grow past 50GB per copy and replication traffic starts competing with client traffic on the same interface, you'll feel it.

The setup I prefer for 3-node DAGs: one dedicated 1GbE or 10GbE replication network connecting all DAG members directly (or through a dedicated VLAN), completely separate from the MAPI/client network. Tag the replication network interface so the DAG knows to use it for log shipping. Takes 30 minutes to configure and eliminates an entire class of performance complaints.

## Witness Server Placement

Put the File Share Witness on your CAS array or on a separate server in a management subnet — not on a domain controller, not on a file server that gets rebooted frequently for patches, and definitely not on a DAG member itself. I've seen all three of those configurations cause quorum issues at exactly the wrong moment.

Document where your FSW lives and make sure your change management process includes checking DAG quorum status before rebooting anything it depends on. That one-page runbook has saved multiple late-night escalations.
