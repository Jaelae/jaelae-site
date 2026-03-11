---
title: "Failed ESXi Upgrade: Recovering When Update Manager Goes Wrong"
date: 2019-08-06
tags: ["VMware", "ESXi", "Upgrades", "Update Manager", "Troubleshooting"]
---

ESXi host upgrades via vSphere Update Manager (VUM, now Lifecycle Manager in newer versions) are routine operations that usually go smoothly. When they don't, the failure modes can range from easily recoverable to genuinely painful. Here's what I've encountered and how to handle it.

## Before You Start

The things that should happen before any ESXi upgrade:

**Verify the upgrade path is supported**: Check the VMware upgrade compatibility matrix for your current and target versions. Not every upgrade is a direct path — some require an intermediate version.

**Evacuate the host**: Put the host in maintenance mode with DRS set to migrate VMs automatically. Verify all VMs have migrated off before proceeding. An upgrade on a host that still has powered-on VMs is a problem waiting to happen.

**Check driver and firmware compatibility**: The VMware HCL lists which driver versions are compatible with which ESXi versions. An upgrade that installs new ESXi kernel modules may conflict with existing third-party VIBs (drivers). The pre-upgrade hardware compatibility check in VUM will flag known conflicts, but not all conflicts are detected automatically.

**Have a rollback plan**: ESXi upgrade creates a rollback partition at install time. Know how to use it before you need it.

## The Most Common Failure: VIB Conflict

ESXi upgrades fail most often because of VIB (VMware Installation Bundle) conflicts. Third-party drivers — storage HBA drivers, network card drivers, vendor management agents — installed as VIBs may be incompatible with the target ESXi version.

The upgrade process detects conflicts and reports them in the VUM remediation report. The resolution is usually to remove the conflicting VIBs before upgrading and reinstall compatible versions afterwards, or to install the ESXi version that includes updated driver compatibility.

If you ignore a VIB conflict report and force the upgrade (possible with certain flags), you risk boot failures — the host upgrades but won't boot because a kernel module that was forcibly removed was load-critical.

## The Host Won't Boot After Upgrade

This is the scenario that requires the rollback partition. ESXi maintains a boot bank (the current version) and an alternate boot bank (the previous version). If the upgraded host won't boot into the new version, you can roll back:

During the DCUI boot process, watch for the option to boot from alternate boot bank. The timing varies by hardware — look for a boot menu option or keypress prompt. Selecting the alternate boot bank boots the previous ESXi version, returning the host to operational status while you investigate the upgrade failure.

Once rolled back, the host is operational again and VMs can be migrated back to it. Take time to understand why the upgrade failed before attempting again.

## Update Manager Database and Scan Issues

VUM maintains its own database of host states, patch baselines, and compliance status. In some environments (particularly those that have run for several years without VUM maintenance), this database accumulates stale entries that cause bizarre behavior: hosts that appear in compliance but fail remediation, updates that show as applicable but aren't actually installed, scan operations that hang indefinitely.

The fix is usually to rescan the host and force VUM to re-inventory the host state. If scanning hangs, check the VUM service health on the vCenter side and look at the `vmware-updatemgr.log` for errors. Sometimes a VUM service restart is necessary to clear stuck operations.

## The ESXi Boot Recovery Partition

Beyond rollback, ESXi's boot architecture includes a recovery/diagnostic partition accessible via the DCUI or via direct console access. If the upgrade corrupted the boot partition and neither boot bank is available, ESXi's built-in recovery options are limited. At that point, reinstalling from ISO and re-joining to vCenter (VMs remain intact on the datastore) is the path.

This is why ESXi hosts in production should have out-of-band management (iDRAC/iLO) with virtual media capability — so you can mount an ISO and boot to it remotely without physical data center access.
