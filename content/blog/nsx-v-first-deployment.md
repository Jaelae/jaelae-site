---
title: "NSX-V First Deployment: What to Actually Prepare For"
date: 2018-09-20
tags: ["VMware", "NSX", "NSX-V", "Networking", "SDN"]
---

NSX-V (for vSphere) is VMware's software-defined networking platform for vSphere environments. It decouples network functions — switching, routing, firewalling, load balancing — from physical infrastructure and manages them in software. The pitch is compelling: micro-segmentation, zero-trust networking, rapid deployment of complex network topologies without touching physical switches. The reality of deploying it for the first time in a mid-size organization has some significant learning curves attached.

## The Prerequisites Are Not Optional

NSX-V has specific prerequisites that, if not met, will cause intermittent failures that are very hard to diagnose:

**MTU**: All physical network infrastructure between ESXi hosts must support an MTU of at least 1600 bytes (1550 is the NSX overlay header overhead over standard 1500 frames; 1600 gives some buffer). NSX uses VXLAN encapsulation for logical networking, which adds a header to every frame. On a physical infrastructure configured for standard 1500 MTU, VXLAN frames get fragmented or dropped, causing seemingly random VM connectivity failures. Verify jumbo frames (MTU 9000 is cleaner than trying to hit 1600 exactly) are configured end-to-end before you install NSX.

**vCenter and ESXi version compatibility**: NSX-V is tightly coupled to specific vSphere versions. Check the VMware Product Interoperability Matrix before starting. Deploying NSX-V on a vSphere version it doesn't fully support creates compatibility gaps in control plane communication.

**Dedicated vCenter for NSX**: For production environments, NSX Manager registers with one vCenter instance. Don't deploy NSX-V in an environment where vCenter is inconsistently available or where SSO has unresolved issues — NSX Manager's dependency on vCenter creates cascading failures when vCenter has problems.

## The NSX Manager Deployment

NSX Manager deploys as an OVF from the NSX installation package. It registers with vCenter and becomes the management plane for your NSX environment. After registration, NSX components appear in the vSphere Web Client interface (note: NSX-V is not compatible with the newer HTML5 vSphere Client in earlier versions — you're using the Flash-based client for NSX management).

After deployment and registration, the next step is preparing ESXi hosts for NSX — this installs the NSX VIBs (kernel modules) on each host that enable VXLAN transport, distributed switching, and distributed firewall. Host preparation can be done rolling (host by host, staying in cluster) or all at once during a maintenance window. For production, rolling preparation is safer — it installs VIBs on each host without requiring downtime for VMs.

## VXLAN Transport Configuration

VXLAN is the overlay network protocol NSX uses to create logical networks that are independent of physical VLANs. To configure it, you create a VTEP (VXLAN Tunnel Endpoint) on each ESXi host — a VMkernel interface on a specific VLAN that will carry VXLAN traffic.

Get the VTEP network design right before you deploy: dedicated VLAN for VXLAN transport, proper MTU, and IP addressing on a separate subnet. Each ESXi host needs a VTEP IP — either assigned statically or via DHCP from a pool you control. I prefer static assignment for production; DHCP for VTEP addresses creates a dependency on DHCP availability for your logical network infrastructure.

## Distributed Firewall: Start Small

The NSX Distributed Firewall (DFW) is what most organizations deploy NSX-V for. It provides micro-segmentation — per-VM firewall rules enforced at the hypervisor kernel level, invisible to the guest OS and independent of physical firewall topology.

The first mistake new NSX deployments make: trying to implement a full zero-trust ruleset from day one. Start with a small pilot: pick a set of non-critical VMs, implement specific application-tier separation rules, and validate that the intended traffic flows work before restricting them. The DFW default rule allows all traffic — it's a permissive baseline that you progressively tighten, not a deny-all you punch holes through.

Document your intended security policy before touching the firewall configuration. Drawing it out — which VMs need to talk to which other VMs on which ports — makes the rule authoring process substantially faster and reduces the chance of cutting off something important.
