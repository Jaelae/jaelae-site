---
title: "Exchange 2013 Database Troubleshooting: Dirty Shutdown and ESEUtil"
date: 2013-10-14
tags: ["Exchange", "Exchange 2013", "Database", "Troubleshooting", "ESEUtil"]
---

At some point, you will open the Exchange Admin Center or EMS and find a database in a dismounted state. If you're lucky, it's a managed availability-triggered failover that resolved cleanly. If you're not, you're staring at a database that won't mount because it's in a dirty shutdown state. Here's how to work through it without making things worse.

## Understanding Dirty Shutdown

Exchange uses the Extensible Storage Engine (ESE), also known as JET Blue, as its database engine. ESE uses write-ahead transaction logging — changes are written to transaction log files first, then committed to the database file (.edb). When Exchange shuts down cleanly, all pending transactions are committed to the database and it enters a "clean shutdown" state.

Dirty shutdown means the database was closed without all pending transactions being committed. This happens during unexpected shutdowns: power loss, server crash, storage failure, someone pulling the wrong cable. The database is in an inconsistent state and ESE refuses to mount it until the missing log files are replayed or the database is brought to a consistent state another way.

## Diagnosing the State

First, verify what state you're actually dealing with:

```
eseutil /mh "E:\ExchangeDatabases\Mailbox01\Mailbox01.edb"
```

This outputs the database header. Look for the `State:` line. A healthy, mountable database shows `Clean Shutdown`. A dismounted, inconsistent database shows `Dirty Shutdown`. You'll also see the log generation number that the database last committed — that's important for what comes next.

Also run `Get-MailboxDatabaseCopyStatus -Identity "Mailbox01\Server01"` in EMS to see the copy status across your DAG if you have one.

## The Repair Path Hierarchy

Before reaching for `eseutil /p` (hard repair), exhaust your other options in this order:

**1. Replay missing logs.** If you have the required transaction logs, ESE can replay them and bring the database to a clean shutdown state automatically. Copy any available logs from the correct log stream to the database's log directory and attempt to mount. Exchange will replay them.

**2. Soft recovery.** Use `eseutil /r E0n` where `E0n` is your log prefix (typically E00 for Exchange 2013). This replays available logs without mounting. Combine with `/l` to specify log path and `/d` to specify database path.

**3. Restore from backup or activate a DAG copy.** If you have a DAG with a healthy copy, this should be your primary path. Activate the passive copy: `Move-ActiveMailboxDatabase -Identity Mailbox01 -ActivateOnServer Server02`. Done in seconds, no data loss.

**4. Hard repair as last resort.** `eseutil /p` repairs the database by discarding corrupted pages. It is destructive — any data on damaged pages is gone. Always run it on a copy, never on your only copy. Always follow it with `eseutil /d` (defragmentation) and `isinteg -fix` to rebuild the Exchange-level database structure.

## The Mistake to Avoid

Running `eseutil /p` first, before checking for backups or healthy DAG copies, is a panic response that causes unnecessary data loss. I've seen it happen when someone reads a forum post at 2am and starts running commands without understanding the full picture. Slow down, assess your copies and backups, and choose the least destructive path.

If the database is critical and you're uncertain, open a Microsoft support case before running hard repair. The cost of a support incident is trivial compared to explaining to users why their email from the past six months is gone.

## After the Crisis

Once you've recovered, look at what caused the dirty shutdown. Storage failures, firmware bugs on HBAs, and Windows crash events (check System event log for critical errors around the time of dismount) are the most common culprits. Fix the root cause before the next incident, not after.
