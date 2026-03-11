---
title: "Exchange 2013 Performance Monitoring: What to Watch and When to Worry"
date: 2014-11-15
tags: ["Exchange", "Exchange 2013", "Performance", "Monitoring"]
---

Exchange 2013's new Information Store architecture — where each database runs in its own worker process (`Microsoft.Exchange.Store.Worker.exe`) — changes what you monitor compared to Exchange 2010. Problems are more isolated, which is good, but your monitoring approach needs to account for per-database process tracking rather than a single monolithic store.

## The Counters That Actually Matter

Performance Monitor has hundreds of Exchange-related counters. Most of them are noise. The ones that consistently tell you something actionable:

**MSExchange Database\I/O Database Reads Average Latency** and **MSExchange Database\I/O Database Writes Average Latency**: These measure how long disk operations are taking against your databases. Microsoft's threshold is 20ms average for reads and 50ms average for writes. Sustained values above these indicate storage performance problems that will eventually manifest as slow OWA, disconnected Outlook clients, and Managed Availability-triggered restarts.

**MSExchange RPC\RPC Requests**: The number of active RPC requests being served by the Information Store. Healthy environments typically show this well under 100. Spikes above 200 during normal business hours indicate either client storms or server resource exhaustion.

**MSExchange ActiveSync\Requests/sec**: Useful for spotting mobile device storms — usually triggered by a mobile OS update that causes all devices to re-sync simultaneously, or a policy change that pushes a wipe-and-reconfigure cycle.

**Processor\% Processor Time (w3wp.exe)**: IIS worker process CPU consumption. Exchange 2013's CAS role runs as IIS. If w3wp.exe is consistently above 50% CPU, your CAS layer is undersized for the request load.

## Memory: The Counter People Check Wrong

Total server memory utilization isn't the right metric for Exchange 2013. The platform is designed to use all available memory for the database cache — high memory utilization is expected and healthy. What you want to watch is **Memory\Available Mbytes**. As long as this stays above 200-300MB, the server has headroom. If it consistently drops below 100MB, you have a memory pressure situation.

Also watch **MSExchange Database\Database Cache Size (MB)**. This should grow over time as Exchange caches frequently accessed database pages. A cache that's been evicted (size dropped suddenly) often correlates with the server paging heavily under memory pressure — which dramatically impacts I/O performance.

## Building a Baseline

Monitor without a baseline is guesswork. The first thing I do when taking on a new Exchange environment is set up a Performance Monitor data collector set to run continuously and save hourly averages for the key counters above.

After two to four weeks of baseline data, you know what "normal" looks like: typical CPU during business hours, average I/O latency, normal RPC request depth. From that baseline, you can set meaningful alerts that catch real problems and don't cry wolf every time there's a Monday morning mail rush.

## The Information Store Worker Process Split

One operational consequence of per-database worker processes: a runaway database (perhaps due to a large move request, a search indexer issue, or a software bug) will consume resources in a way that's visible per-process rather than globally. 

If Exchange feels slow but server-wide CPU and memory look fine, check the individual worker processes in Task Manager or via `Get-Process Microsoft.Exchange.Store.Worker`. You may find one database's worker consuming 40% CPU while everything else is idle — a targeted problem that's invisible if you're only watching aggregate counters.

Tracking down which database a given worker process serves: check the process command line arguments or cross-reference process creation time with `Get-MailboxDatabaseCopyStatus` events in the Crimson Channel log around the same time.
