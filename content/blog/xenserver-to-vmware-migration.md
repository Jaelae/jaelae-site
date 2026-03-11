---
title: "Migrating from XenServer to VMware ESXi: The Practical Guide"
date: 2016-05-09
tags: ["XenServer", "VMware", "ESXi", "Migration", "Virtualization"]
---

At some point the XenServer environments I deployed for cost reasons hit a wall — either the organization's needs grew beyond what XenServer's ecosystem could support cleanly, or a strategic decision to standardize on VMware made the migration necessary. Moving workloads between hypervisors isn't trivial, but with the right approach it's manageable.

## The Core Challenge: Disk Format Conversion

XenServer uses VHD (Virtual Hard Disk) format for VM disks. VMware uses VMDK. These are different disk formats and ESXi can't directly boot from a VHD file. Every VM migration requires disk format conversion.

Tools for the conversion:

**VMware vCenter Converter Standalone**: VMware's free conversion tool supports converting running VMs or disk images to VMDK format. For XenServer source VMs, you run the converter agent on the source VM and convert it to a target ESXi host. Works reasonably well for Windows VMs; Linux conversions sometimes require manual driver adjustment post-migration.

**Starwind V2V Converter**: A free alternative that handles a variety of disk format conversions including VHD to VMDK. Useful when vCenter Converter has compatibility issues with specific VHD configurations.

**Manual export/import**: Export from XenServer as OVA or XVA, convert disk format with qemu-img, import to ESXi. More steps but gives control over the conversion process. `qemu-img convert -f vpc source.vhd -O vmdk target.vmdk`

## Pre-Migration Checklist

Before converting any VM:

1. **Document the network configuration**: IP addresses, VLAN assignments, NIC teams. These won't transfer automatically — you'll reconfigure networking in the destination VM.

2. **Verify application health**: Migrate healthy VMs, not VMs that have existing problems. Post-migration troubleshooting is harder.

3. **Uninstall XenServer Tools**: The XenServer para-virtualized drivers must be removed from Windows VMs before migration. Running XenServer Tools on ESXi causes driver conflicts and boot failures. Uninstall via Add/Remove Programs, then shut down cleanly.

4. **Install generic drivers**: Ensure the Windows VM has the generic (non-paravirtual) IDE/SCSI driver enabled. XenServer uses paravirtual storage drivers; ESXi uses different ones. A VM that only has XenServer paravirtual storage drivers won't find its disk on ESXi without the generic driver path available.

## Post-Migration Steps

After a Windows VM is running on ESXi:

1. **Install VMware Tools**: This is the VMware equivalent of XenServer Tools — paravirtual drivers for networking and storage, memory balloon driver, quiesce support for backups. Install immediately.

2. **Convert to PVSCSI**: Once VMware Tools is installed, optionally convert the VM's storage controller from LSI Logic (the default after conversion) to PVSCSI for better I/O performance. This requires powering down the VM and changing the controller type with data intact.

3. **Verify networking**: Check that the VM has the correct network adapter (VMXNET3 is preferred) and that its IP configuration and network connectivity is correct.

4. **Application testing**: For every migrated VM, verify application functionality before decommissioning the XenServer source.

## The Timeline Reality

Plan for migration to take longer than the conversion tool suggests. For 20-server environments, allow 4-6 weeks:

- Week 1: Environment prep (new ESXi cluster, vCenter, shared storage configured)
- Week 2-3: Pilot migration (5 non-critical VMs, full testing cycle)
- Week 4-5: Production migration in batches
- Week 6: Validation period, XenServer decommission

Running XenServer and ESXi in parallel during migration means you have a rollback path — if a converted VM has issues, the XenServer source is still available. Don't decommission XenServer until you're confident all migrations are successful.
