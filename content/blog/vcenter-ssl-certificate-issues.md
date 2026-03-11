---
title: "vCenter SSL Certificate Issues: The One That Bit Me Three Times"
date: 2016-10-04
tags: ["VMware", "vCenter", "Certificates", "SSL", "Troubleshooting"]
---

Certificate issues in vSphere environments are the most reliable source of annoying, hard-to-diagnose problems I've encountered. They show up at upgrade time, after hardware changes, when adding new hosts, and sometimes apparently at random. Here's what I've learned after dealing with these problems across multiple client environments.

## The VMCA and What It Does

Starting with vSphere 6.0, VMware introduced VMCA — the VMware Certificate Authority. VMCA is a built-in CA embedded in the vCenter platform. It automatically issues certificates for ESXi hosts, vCenter services (Solution Users), and the vCenter web interface.

By default, VMCA uses a self-signed root certificate. This means your browser will show certificate warnings when connecting to vCenter, and tools that validate certificate chains will flag the certs as untrusted. In environments with their own internal PKI (Active Directory Certificate Services is common), you can configure VMCA as a subordinate CA — signed by your internal root — which makes the certificates trusted throughout your organization.

## Three Configurations, Three Different Problems

**Default VMCA (self-signed)**: Browsers warn, but functionally everything works. The problem comes at upgrade time. If the VMCA root certificate has expired (default validity is 10 years, but shorter in some configurations), upgrades will fail cert pre-checks. I've seen this happen on environments that were deployed in 2016 and never had certificates touched.

**VMCA as subordinate CA**: The right long-term configuration for most organizations. Requires CSR generation, signing by your internal CA, and reimporting into VMCA. The complexity is manageable but the procedure is precise and unforgiving of mistakes. A wrong step in the certificate import sequence can break SSO authentication and leave you staring at a vCenter login page that bounces back with cryptic SSL errors.

**Custom certificates mode (all external)**: Every certificate replaced with externally signed certs. Maximum control, maximum maintenance burden. Not recommended for SMB environments without dedicated PKI management capabilities.

## Adding a Host and the Certificate Trust Problem

The scenario I hit most often in SMB environments: you rebuild an ESXi host, reinstall from ISO, and then try to add it back to vCenter. Error: "SSL exception — unable to get local issuer certificate."

What happened: the reinstalled ESXi host has a new self-signed certificate. vCenter's trust store still has the fingerprint of the old certificate. When vCenter tries to verify the new host's certificate against its trust store, the chain doesn't validate.

Resolution: In vCenter, when adding a host, there's typically an option to accept the host's thumbprint and add it to the trust store. If the error is appearing mid-reconnect rather than during the add workflow, you can reset certificate trust through the vSphere Host Client on the ESXi host (`/sbin/generate-certificates` to regenerate the default cert, followed by re-adding from vCenter).

The nuclear option if certificates are truly broken: setting `vpxd.certmgmt.mode` to `thumbprint` in vCenter's advanced settings temporarily allows vCenter to accept any certificate without full chain validation. This is a troubleshooting measure, not a production configuration — document it and revert to proper certificate management once the immediate problem is resolved.

## Stale Certificates After a Failed Upgrade

This is where it gets expensive. A vCenter upgrade that fails mid-process can leave certificate stores in an inconsistent state — partially migrated, with old and new certificate data coexisting in VECS (VMware Endpoint Certificate Store).

The symptoms: vCenter services fail to start after a failed upgrade, or start but can't authenticate to each other (solution user auth failures in the `vpxd.log`). The logs are terse and the error messages aren't immediately obvious as certificate problems.

The remediation: VMware KB articles exist for common certificate remediation scenarios. The `vecs-cli` tool on the VCSA lets you inspect and manipulate certificate stores directly. The general approach is: identify which certificate store is corrupted or contains a stale entry, remove it, and re-trigger the certificate provisioning workflow.

Before any major vCenter upgrade, snapshot the VCSA. It's the single most effective preparation for the certificate problems that upgrades sometimes introduce.
