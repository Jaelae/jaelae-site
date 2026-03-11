---
title: "vCenter SSO: The Most Misunderstood Part of vSphere"
date: 2017-06-22
tags: ["VMware", "vCenter", "SSO", "Authentication", "Troubleshooting"]
---

vCenter Single Sign-On (SSO) is the authentication backbone of the vSphere platform. It handles login to vCenter, manages identity sources, and issues tokens that vSphere services use to authenticate with each other internally. When it works, it's invisible. When it breaks, everything breaks simultaneously and the error messages are spectacularly unhelpful.

## What SSO Actually Is

SSO in vSphere is built on VMware's own LDAP-based directory service (`vmware-directory`, also called vmdir). This is separate from Active Directory — it's an internal directory that stores vSphere-specific objects like service accounts, vCenter roles, and licensing data.

Identity sources are configured in SSO to allow users from external directories (Active Directory, LDAP) to authenticate to vCenter. But the SSO admin account (`administrator@vsphere.local` by default) lives in vmdir and is independent of AD — which means if your AD is unreachable, the `vsphere.local` admin account can still log in.

This distinction matters for disaster recovery: if your AD is down, you can still manage your VMware environment through the `vsphere.local` admin account. Know this password and store it securely.

## The 90-Day Password Expiration Problem

By default, the `vsphere.local` administrator password expires every 90 days. This catches organizations off-guard when vCenter suddenly stops accepting logins and nobody knows why. The password expiration doesn't generate prominent warnings — it silently expires.

The fix: either change the password policy in SSO (Administration > Single Sign On > Configuration > Password Policy) to extend the expiration or disable it for service accounts, or set a calendar reminder to change the password before expiration.

For service accounts used by backup software, monitoring tools, or automation that authenticate to vCenter, password expiration is particularly disruptive. Either use AD-based service accounts with managed expiration, or configure the `vsphere.local` service accounts to not expire.

## SSO After vCenter Reinstall or Restore

When you restore a vCenter from backup or snapshot after a failure, SSO state is embedded in the VCSA. The vmdir database contains the configuration of all connected solutions and extensions.

If you restore a VCSA to a state before certain extensions (like Veeam, vRealize, or NSX) were registered, those extensions will no longer be registered in the restored vCenter's vmdir — you'll need to re-register them.

The worse scenario: restoring to a state that's inconsistent with what the ESXi hosts in the environment expect. Hosts authenticated against a vCenter SSO domain use certificates issued by that environment's VMCA. If VMCA state changes during restore, host trust may need to be re-established by disconnecting and reconnecting hosts.

## Active Directory Integration Issues

The most frequent SSO call I get from SMB clients: "We can log in with the vsphere.local admin account but not with our AD accounts." 

Diagnostic path:

1. Verify the AD identity source is configured and shows as healthy in SSO Administration
2. Check that the `vsphere.local` domain can reach domain controllers via DNS and LDAP (port 389, 636 for LDAPS)
3. Verify the service account used for the AD binding hasn't expired or had its password changed
4. Check the SSO logs at `/var/log/vmware/sso/` on the VCSA — look for LDAP bind failures or authentication rejections

The most common root cause: the service account password used for the AD identity source was changed during a routine security sweep and nobody updated the vCenter SSO configuration. A two-minute fix once you know that's the cause; a frustrating hunt if you don't.

## Locked Out Completely

If the vsphere.local admin password is expired and you're locked out: the VCSA has a built-in admin recovery path accessible via the DCUI (Direct Console User Interface) or the VCSA management interface on port 5480. Boot the VCSA in rescue mode and use the `passwd` command to reset the administrator@vsphere.local credential. The exact procedure is documented in VMware KB 2147144 and variations thereof for different versions. Know it exists before you need it.
