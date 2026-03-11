---
title: "Exchange 2013 Certificate Management: The Part Nobody Documents Well"
date: 2014-02-18
tags: ["Exchange", "Exchange 2013", "Certificates", "SSL"]
---

Certificate issues are the most consistent source of Exchange 2013 headaches I've encountered across different organizations. The platform handles certificates in a specific, opinionated way — and when something goes wrong, the error messages are rarely helpful enough to point you directly to the fix.

## The Certificate Landscape in Exchange 2013

Every Exchange 2013 server needs a certificate for SMTP, IIS (OWA, EWS, ECP, OAB, ActiveSync), and POP/IMAP if those services are enabled. The certificate's Subject Alternative Names need to cover every name clients will use to connect: the primary namespace (`mail.company.com`), autodiscover (`autodiscover.company.com`), and any legacy namespaces still in use.

The internal certificate story is separate from the external one. Internally, clients connect to the CAS servers by FQDN — which may or may not match your external namespace. If you have internal DNS split-horizon and your internal clients resolve `mail.company.com` to an internal CAS IP, the internal certificate name coverage is the same as external and you're fine. If internal clients connect directly to the server FQDN, your internal certificate needs to cover those names too.

Plan your namespaces completely before you request your certificate. Certificate amendments after issuance are annoying and sometimes costly.

## Services Assignment

After importing a certificate, you need to explicitly assign it to Exchange services. This is done via `Enable-ExchangeCertificate` in EMS. Not doing this — or assigning it to the wrong services — is a common source of the baffling "there's a certificate error" calls that start happening after what seemed like a successful cert replacement.

The services you'll typically assign are SMTP, IIS, and optionally POP and IMAP. When you assign a new certificate to SMTP, Exchange prompts you to confirm overwriting the default self-signed certificate. Confirm, then verify: check `Get-ExchangeCertificate` and make sure your new cert shows the correct services in the `Services` column.

## The Autodiscover Certificate Problem

Autodiscover uses a specific URL (`https://autodiscover.company.com/autodiscover/autodiscover.xml`) that Outlook and mobile clients query to discover connection settings. This URL must resolve to a server presenting a valid, trusted certificate for `autodiscover.company.com`.

A common issue: the certificate covers `mail.company.com` but autodiscover was added as a separate SAN with a typo, or autodiscover resolves externally to a load balancer that's still presenting the old certificate. Outlook clients start prompting users with certificate warnings, or fail to configure mail accounts automatically.

Test your autodiscover endpoint with the Microsoft Remote Connectivity Analyzer (ExRCA) at testconnectivity.microsoft.com before and after any certificate change. It's free and will surface problems that are invisible from inside your own network.

## When Certificates Cause Managed Availability Failures

A subtler issue: if your internal certificate expires or has name mismatches, Exchange Managed Availability's self-test probes — which make internal HTTPS connections — will start failing. This triggers responders that restart IIS or other components, causing intermittent service disruptions that appear unrelated to certificates.

The tell is checking monitor health with `Get-ServerHealth` and seeing OWA, EWS, or ECP monitors in Unhealthy state, combined with certificate errors in the IIS logs. Replacing the certificate fixes both the direct certificate issue and the Managed Availability cascade that followed it.

## Documentation and Tracking

Keep a spreadsheet (or a proper CMDB entry) for every Exchange certificate in your environment: server, thumbprint, expiration date, assigned services, issuing CA, and renewal date reminder set 60 days before expiry. Certificate expiration causing an Exchange outage is entirely preventable and entirely embarrassing. Don't let it happen because nobody tracked when it was due.
