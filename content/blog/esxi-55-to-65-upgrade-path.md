---
title: "ESXi 5.5 to 6.5: The Upgrade Path Nobody Documents Completely"
date: 2018-04-10
tags: ["VMware", "ESXi", "Upgrades", "vSphere", "Migration"]
---

ESXi 5.5 reached end of general support in September 2018, creating real urgency to move environments that had been running the same version for three to four years. The upgrade path to 6.5 (the stable target at the time) had some gotchas that weren't clearly documented in a single place. Here's what the actual upgrade looked like in practice.

## The Supported Upgrade Path

Direct upgrade from ESXi 5.5 to 6.5 is supported by VMware. You don't need to go through 6.0 first. However, vCenter has its own upgrade sequencing: vCenter must always be at or above the version of the ESXi hosts it manages. Upgrade vCenter to 6.5 before upgrading any ESXi hosts.

For environments on Windows-based vCenter 5.5, the upgrade path to VCSA 6.5 is not an in-place upgrade — it's a migration. VMware provides the vCenter Server Migration Tool that migrates configuration and data from Windows vCenter to a fresh VCSA. Plan time for this migration and verify it in a test environment first.

## vCenter Upgrade First

The VCSA 6.5 upgrade from VCSA 5.5 is an in-place upgrade via the VCSA Management Interface (port 5480). The process:

1. Take a snapshot of the VCSA VM
2. Access the management interface and upload the 6.5 upgrade package (or stage via ISO)
3. Run the pre-upgrade validation — review any warnings before proceeding
4. Execute the upgrade (30-60 minutes depending on database size)
5. Verify all services started correctly after upgrade

The most common issue: PSC (Platform Services Controller) complications. If you have external PSC VMs in your 5.5 environment, the PSC must be upgraded before vCenter. In 6.5, VMware recommends converging to embedded PSC if you're on external PSC — plan this as a separate step after the upgrade is stable.

## ESXi Host Upgrades

With vCenter at 6.5, upgrade ESXi hosts via Update Manager baselines. Create a 6.5 upgrade baseline (using the ESXi 6.5 update ISO), attach it to each host cluster, and remediate.

The pre-check that trips most environments: third-party VIBs (drivers) installed on hosts may not be compatible with ESXi 6.5. The upgrade pre-check in Update Manager will flag conflicting VIBs. Common offenders:

- Legacy NIC drivers for older NICs that aren't natively supported in 6.5
- Storage vendor driver VIBs that haven't been updated for 6.5
- Management agent VIBs from older HPE or Dell toolkits

For each flagged VIB: find the 6.5-compatible version from the vendor and stage it for installation after the ESXi upgrade, or determine if the VIB is still necessary (sometimes they're residue from previous drivers that are no longer needed).

## VM Hardware Compatibility Level

ESXi 6.5 supports VM hardware version 13 (vs. version 10 in ESXi 5.5). Upgrading VM hardware version unlocks new features (UEFI boot, improved paravirtual SCSI, etc.) but is not required for VMs to run on upgraded hosts — older hardware versions continue working.

For most environments, I recommend upgrading VM hardware versions on a scheduled basis after the host upgrade is stable and tested, not immediately during the migration window. Changing hardware version requires a VM reboot and is irreversible (you can't downgrade hardware version, only restore from snapshot).

## The Week After

The week following a major vSphere version upgrade is the time to watch carefully. Monitor your backup jobs (Veeam/Rubrik both require version compatibility with the vSphere API version — verify they still work). Check your monitoring system's connectivity to vCenter. Verify any custom scripts or automation are still functioning against the new API.

Infrastructure upgrades don't end at the successful installation — they end when all dependent systems have been verified against the new version.
