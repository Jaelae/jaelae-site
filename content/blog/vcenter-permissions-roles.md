---
title: "vCenter Permissions and Roles: Getting It Right From Day One"
date: 2017-02-28
tags: ["VMware", "vCenter", "Security", "Permissions", "Roles"]
---

vCenter's role-based access control is one of those configuration areas that gets set up once during initial deployment and then rarely revisited — until someone has more access than they should, or can't do something they need to, and tracing why requires untangling permissions that accumulated over years.

## How vCenter Permissions Work

vCenter permissions are applied as a combination of a **role** (a collection of privileges) and an **identity** (a user or group) at a specific **inventory object** (a datacenter, cluster, folder, VM, or other object). Permissions are inherited downward through the inventory hierarchy — a role applied at the datacenter level propagates to all objects within it, unless a more specific permission overrides it at a lower level.

The administrator role, applied at the root vCenter level, grants full access to everything. This is appropriate for your vCenter admin team and nothing else. Every other use case should have a more restricted role.

## The Built-In Roles and Their Limits

vCenter ships with several built-in roles: No Access, Read-Only, View, VM Power User, VM User, Resource Pool Administrator, VMware Consolidated Backup User, Datastore Consumer, Network Consumer, and Administrator.

Most of these built-in roles are too coarse for real-world use cases. The gaps I encounter most often:

- A backup service account that needs read/write access to VMs for VADP backups but shouldn't be able to delete VMs or change network configurations — no built-in role covers this cleanly
- A Tier 1 support role that can power VMs on/off and take snapshots but not modify hardware or network settings
- A database team that needs to manage specific VM folders without visibility into the broader infrastructure

These use cases require custom roles.

## Custom Roles

Custom roles are created in vCenter Administration → Roles. The privilege list is granular — hundreds of individual privileges covering every vCenter action. Build custom roles by starting with the closest built-in role as a reference, then adding or removing specific privileges.

For service accounts (backup software, monitoring tools, automation scripts), build the minimum-privilege role that allows the tool to function. Test it thoroughly in a non-production environment. Document the role purpose and the privileges it includes — this documentation is invaluable when someone audits access six months later and needs to know why a service account has datastore write access.

## Common Permission Mistakes

**Global administrator accounts for automation**: Running backup software or monitoring agents with full vCenter administrator credentials is a security anti-pattern. When those credentials are exposed (and they eventually are), the blast radius is the entire infrastructure.

**Permissions at the root rather than at the datacenter or folder**: Applying access at the root vCenter level means it applies to all datacenters, all clusters, all VMs — often more than intended. Apply permissions at the lowest inventory level that covers the intended scope.

**AD group membership drift**: Granting vCenter access to an AD security group and then not maintaining that group means former employees retain vCenter access after leaving the organization. Audit vCenter-connected AD groups against HR records at least quarterly.

**Forgotten local accounts**: vCenter's `vsphere.local` identity source contains local accounts. Check `Administration → Users and Groups` for `vsphere.local` accounts beyond the default Administrator. Unknown local accounts are a red flag.

## Service Account Credential Management

For automation and integration accounts (Veeam, Rubrik, Zabbix, etc.), document each service account in a credential vault or password manager, with the associated vCenter role, the systems that use it, and the last rotation date. When it's time to rotate the password, you know exactly where it's used and can update it everywhere without a service outage.
