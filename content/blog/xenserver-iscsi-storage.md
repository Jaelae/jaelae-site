---
title: "XenServer iSCSI Storage Repositories: What the Documentation Misses"
date: 2015-08-24
tags: ["XenServer", "Citrix", "Storage", "iSCSI"]
---

Connecting XenServer to iSCSI shared storage is more nuanced than the documentation implies. Most guides walk you through the happy path — discover the target, create the SR, done. They skip the operational details that determine whether your shared storage actually behaves reliably over time.

## Use a Dedicated Storage Network

This one is non-negotiable. iSCSI storage traffic should be on a dedicated NIC (or bonded pair) and a dedicated VLAN, completely separate from management traffic and VM network traffic.

Running iSCSI over your management network introduces two problems: first, iSCSI I/O competes with pool management traffic, causing intermittent timeouts in XenCenter and slow xe CLI responses under I/O load; second, any network disruption that affects management also affects storage, making troubleshooting events cascade.

Set up a dedicated iSCSI NIC on each XenServer host during initial configuration. Assign static IPs on the storage network (not DHCP — storage should never rely on a DHCP server being available). Configure your storage target to present on the same subnet.

## Multipath: Configure It, Test It, Monitor It

For any production XenServer iSCSI deployment, multipath is essential. Connect at least two paths from each host to the storage array — two ports on the host's storage NIC (or two separate NICs) to two separate paths on the storage target.

XenServer uses the Linux device mapper multipath daemon under the hood. After connecting your iSCSI storage, verify that multipath is seeing all expected paths:

```bash
multipath -ll
```

You should see each LUN presented with multiple paths in `active` state. A single path seen per LUN means multipath isn't functioning or wasn't configured correctly — you have no failover protection.

Test your multipath configuration by physically disconnecting one network path and verifying that storage I/O continues without interrupting running VMs. Do this during a maintenance window, not in response to an incident.

## The LUN ID Limitation

XenServer has a historical limitation worth knowing: LUN IDs greater than 255 are not supported for iSCSI SRs. This sounds obscure until you're working with a storage array that auto-assigns LUN IDs and has handed out a high-numbered LUN to your XenServer hosts. The SR creation will fail in a non-obvious way.

Check the LUN ID assigned by your storage admin before spending time debugging SR creation. It's a quick `xe sr-probe` check and can save an hour of confusion.

## iSCSI CHAP Authentication

Use CHAP. It's not optional in any environment where you care about storage security. XenServer supports CHAP authentication for iSCSI — configure the CHAP secret in both the initiator configuration and the storage target. Do it at SR creation time; retrofitting CHAP onto an existing SR is possible but requires detaching and reattaching the storage.

One gotcha: XenServer hosts use a single iSCSI initiator IQN per host, automatically generated during installation. When registering your XenServer hosts as initiators in your storage array, use this IQN. Find it with:

```bash
cat /etc/iscsi/initiatorname.iscsi
```

Each host has a unique IQN. For a pool of four hosts, you'll be registering four separate initiators at the storage array and then mapping your LUN to all four.

## SR Forget vs. SR Destroy

Know the difference before you need it in an emergency: `sr-forget` removes the SR from XenServer's database but leaves the data intact on the physical LUN. Use this when moving an SR from one pool to another, or when you need to re-attach a LUN that XenServer lost track of. `sr-destroy` wipes the contents of the LUN. Running the wrong command under pressure is a bad day. These are permanently destructive operations — verify twice.
