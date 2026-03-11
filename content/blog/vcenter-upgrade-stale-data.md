---
title: "Stale Data After a Failed vCenter Upgrade: How to Clean It Up"
date: 2017-04-03
tags: ["VMware", "vCenter", "Upgrades", "Troubleshooting"]
---

A failed vCenter upgrade is among the more stressful VMware events you'll deal with. The VMs are still running, HA and DRS are still functioning on cached state, but vCenter itself is in an inconsistent state between versions and things are subtly broken in ways that compound over time. Here's what to do.

## How Failed Upgrades Create Stale Data

vCenter upgrades (particularly 5.x to 6.x or 6.0 to 6.5 paths) involve multiple sequential stages: database schema migration, certificate store migration, SSO configuration updates, and service reconfiguration. If the upgrade process fails partway through, you can end up with:

- A vCenter database partially migrated to the new schema
- VECS certificate stores in a mixed state (old and new cert formats)
- SSO configurations referencing both old and new service registrations
- Extension registrations in the database pointing to services that no longer exist

The common symptom set: vCenter starts but certain functions don't work, you see "connection refused" or "service unavailable" errors for specific features, and the vSphere Client shows a subset of your inventory correctly while other objects appear missing or grayed out.

## First: Restore vs. Repair

Before attempting any in-place repair, assess whether restoring from your pre-upgrade snapshot or backup is more practical. If you took a VCSA snapshot before the upgrade (you should always do this), rolling back to it should be the first option unless you've made configuration changes since the snapshot was taken.

Snapshot rollback is clean, predictable, and fast. In-place repair of a partially upgraded vCenter is time-consuming, requires careful following of VMware KB procedures, and has its own risk of making things worse. For most SMB scenarios, the snapshot restore is the right call.

## When You Can't Roll Back

If there's no clean snapshot, or the snapshot was taken too long ago to be useful, in-place repair is the path.

**Step 1: Identify the failure point.** Check `/var/log/vmware/` on the VCSA. The upgrade logs (`upgrade-runner.log`, `ldf_upg.log`) will show where the process stopped. Understanding what completed vs. what didn't determines the repair approach.

**Step 2: Check service health.** On the VCSA, run `service-control --status --all`. Services in a stopped or failed state tell you which components need attention. Authentication issues (SSO/vmdir) show as `vmware-sts` or `vmafdd` failures.

**Step 3: Address certificate issues.** Use `vecs-cli store list` to see all certificate stores and `vecs-cli entry list --store <store>` to inspect entries. Stale entries from old solution users or services that no longer exist can prevent service startup. Removing them (with appropriate VMware KB guidance — don't wing this) allows services to re-register cleanly.

**Step 4: Re-register extensions.** Partially upgraded environments sometimes have extensions registered to old endpoints. `Managed Object Browser` (MOB) at `https://vcenter/mob` lets you inspect the extension manager and remove stale registrations. Access: `https://vcenter/mob/?moid=ExtensionManager`. This browser is disabled by default in newer versions and must be temporarily enabled in advanced settings.

## Preventing the Problem

The checklist before any vCenter upgrade:

1. Take a snapshot of the VCSA VM (or backup if snapshots aren't your primary backup method)
2. Check current certificate expiration dates — an upgrade won't fix expiring certs and may fail because of them
3. Run the pre-upgrade checker that VMware provides with newer VCSA versions
4. Verify you have vCenter and all ESXi hosts at supported interoperability levels for the target version
5. Confirm your DNS, NTP, and domain connectivity are clean — authentication issues during upgrade are often DNS-related

The upgrade itself should happen during a maintenance window with at least two hours of buffer. The VCSA upgrade is generally faster than Windows-based vCenter was, but complex environments with plugin-heavy configurations can take longer than expected.
