---
title: "ESXi Networking Basics: vSwitches, Port Groups, and Where People Go Wrong"
date: 2016-04-20
tags: ["VMware", "ESXi", "Networking", "vSwitch"]
---

VMware's virtual networking model is one of the most powerful things about the platform and one of the most common sources of misconfiguration in smaller deployments. The concepts are learnable in an afternoon, but the consequences of getting them wrong can be subtle and persistent.

## Standard vSwitch vs. Distributed vSwitch

ESXi standalone hosts use Standard vSwitches (vSS). When you add vCenter, you gain access to Distributed vSwitches (vDS), which are managed centrally and provide more consistent networking across hosts.

For SMB deployments starting with just ESXi (no vCenter yet), you're working with standard vSwitches. Each host has its own vSwitch configuration — if you have four hosts and want consistent VM network configurations, you need to configure those four vSwitches identically. Manually. This is error-prone and is one of the primary reasons adding vCenter matters beyond just having a unified management interface.

## Port Groups

Port groups define the network that VMs connect to. At minimum, you'll create port groups for each VLAN your VMs need access to. A standard port group configuration:

- **Management Network**: VMkernel port for ESXi host management traffic. IP assigned to the host itself. Do not put VM traffic on this port group.
- **VM Network**: Default port group for virtual machine traffic. Trunk this for multiple VLANs if needed.
- **vMotion Network**: Dedicated VMkernel port for live migration traffic. Should be on a separate physical NIC path from management.

VLAN tagging in VMware: port groups can be assigned a VLAN ID (guest tagging), set to 4095 (VGT — virtual guest tagging, where the VM handles VLAN tagging), or set to 0 (no VLAN). For most VM workloads, configure the VLAN ID on the port group and let the hypervisor handle tagging. Your upstream switch port should be a trunk carrying the relevant VLANs.

## NIC Teaming and Load Balancing Policies

Standard vSwitches support NIC teaming for redundancy and optionally for load balancing. The default load balancing policy is "Route based on originating virtual port" — this means each VM's NIC is assigned to a physical uplink and stays there. It provides failover but not true load balancing across uplinks.

The "Route based on IP hash" policy actually distributes traffic across uplinks, but it requires your physical switch to have LACP (802.3ad) configured on the corresponding ports. Without LACP on the switch side, IP-hash teaming will appear to work but won't distribute load correctly.

Common mistake: enabling IP-hash teaming on the VMware side without configuring LACP on the physical switch. The result is asymmetric traffic that may work acceptably at low load but breaks under heavy I/O.

## The Promiscuous Mode Problem

Promiscuous mode on a port group allows VMs on that port group to see all traffic passing through the vSwitch, not just traffic addressed to them. There are legitimate uses (network monitoring, certain appliances), but promiscuous mode enabled globally on a production vSwitch is a security problem.

Check your port group configurations periodically: in vCenter or the ESXi host client, look at each port group's security settings. Promiscuous mode, MAC address changes, and forged transmits should be disabled on most port groups. Document any exceptions and the reason for them.

## What to Separate at the Physical Level

For a four-NIC host (minimum recommended for production):

- NIC1/NIC2 bonded: VM traffic + management (separated by VLAN)
- NIC3/NIC4 bonded: vMotion + storage (iSCSI/NFS if applicable)

For six or eight NICs, you can separate management from VM traffic and give vMotion and storage their own dedicated paths. More isolation means that any single component failure or performance issue is contained to one traffic type.

The temptation to simplify ("two NICs is fine for now") is always there in SMB deployments. Resist it at the design phase — retrofitting network separation onto a running cluster is painful.
