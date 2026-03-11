---
title: "Exchange 2013: Migrating from 2010 and What We Didn't Expect"
date: 2013-04-12
tags: ["Exchange", "Exchange 2013", "Email", "Migration"]
---

Exchange 2013 launched with a significantly revamped architecture, and when the organization I was supporting decided to move off Exchange 2010, I expected a relatively clean migration path. We'd done the 2007-to-2010 move a few years prior without too much drama. Exchange 2013 turned out to be a different beast.

## The Architecture Shift That Changes Everything

The biggest adjustment was wrapping my head around the consolidated role model. Exchange 2013 reduced the traditional roles down to just two: Client Access Server (CAS) and Mailbox. Gone were the Hub Transport, Unified Messaging-as-separate-role, and Edge-as-internal-component patterns I'd grown used to. The CAS role in 2013 is stateless — it acts purely as a proxy, forwarding connections to whichever Mailbox server holds the active database copy for that user.

This sounds elegant in theory. In practice it means that the CAS server itself is largely irrelevant for troubleshooting most client issues. The action happens at the Mailbox server, which took some getting used to when reading event logs and performance counters.

## Coexistence: Longer Than You Think

We ran 2010 and 2013 side-by-side for about four months. Microsoft's documentation makes coexistence sound like a temporary bridge — spin up the new servers, migrate mailboxes, decommission old ones. What the docs underplay is how much you need to keep the 2010 environment healthy during that period. 

We had an incident mid-migration where a 2010 Hub Transport service started queuing messages because a connector was misconfigured after we added the 2013 infrastructure. Mail was flowing fine for 2013-homed mailboxes but backing up for anyone still on 2010. The coexistence routing through 2013 CAS was introducing an unexpected relay hop. Took three hours to trace.

## CAS Namespace Planning

If there's one thing to get right before you start, it's your namespace design. Exchange 2013 consolidated the Outlook Anywhere, EWS, ActiveSync, OWA, and autodiscover namespaces in a cleaner way, but you still need to think carefully about your SSL certificate Subject Alternative Names and your external DNS records before the first server goes in.

We ended up with a cleaner setup post-migration — one primary namespace, one autodiscover namespace, proper internal/external DNS split — but we had to revisit the certificate twice during the process because we hadn't fully mapped out all the service endpoints before procurement.

## The One Thing I'd Do Differently

Start the coexistence period with a complete audit of your existing connectors, public folder configuration, and any legacy Exchange attributes on Active Directory objects. Old AD objects from previous migrations — especially decommissioned servers or databases that weren't properly cleaned up — will surface at the worst possible times. We had ghost public folder database references on mailbox database objects that caused intermittent Outlook connectivity prompts weeks into the migration. Cleaned up in AD, never looked back.

Exchange 2013 is genuinely a better platform than 2010, but the migration itself requires more upfront design work than the documentation implies. Budget the time.
