---
title: "NSX-V and Physical Firewalls: Finding the Right Boundary"
date: 2019-02-14
tags: ["VMware", "NSX", "NSX-V", "Networking", "Security", "Firewall"]
---

Organizations deploying NSX-V often already have physical perimeter firewalls — Palo Alto, Fortinet, Cisco ASA, or Check Point. A common design question: what does NSX do, and what does the physical firewall continue to do? Getting this boundary wrong creates operational complexity without commensurate security benefit.

## The Principle: NSX for East-West, Physical for North-South

The clearest mental model for NSX/physical firewall responsibilities:

**Physical perimeter firewall**: North-south traffic — traffic entering or leaving your environment from the internet or WAN. External users accessing internal applications, outbound internet access, VPN termination, cross-site connectivity. Physical firewalls excel at this: stateful inspection, IPS/IDS integration, SSL decryption, zone-based policies.

**NSX Distributed Firewall**: East-west traffic — VM-to-VM communication within your data center. Application tier segmentation, preventing lateral movement within the environment. The DFW's strength is micro-segmentation at the vNIC level, which physical firewalls can't do without hairpinning all traffic through the physical device.

**NSX Edge Services Gateway**: The handoff point between logical networks and the physical network. The ESG handles routing between your logical networks and the physical firewall, SNAT for outbound traffic, and load balancing for internally-accessed applications.

## What Doesn't Work Well

**Hairpinning all VM traffic through a physical firewall**: Some organizations want every VM-to-VM communication to traverse the physical firewall for logging and inspection. This creates a massive I/O bottleneck — all inter-VM traffic has to leave the virtual environment, hit the physical firewall, and return. The physical firewall becomes the performance ceiling for your entire virtual environment. NSX DFW provides equivalent east-west control without this hairpin penalty.

**Duplicating rules in both places**: Attempting to maintain matching security policies in both the DFW and the physical firewall doubles administrative work and creates inconsistency risk. Pick the appropriate tool for each traffic type and enforce policy in one place.

**Using NSX Edge as a full internet-facing perimeter**: NSX ESG is a powerful routing and perimeter device, but for internet-facing perimeter duty (external-user-accessible applications, direct internet connectivity), a purpose-built physical firewall with IPS, URL filtering, and SSL inspection is more appropriate.

## Practical Integration Patterns

**Pattern 1: Simple physical perimeter + NSX east-west**. The physical firewall handles all external connectivity and perimeter inspection. NSX DFW handles all internal east-west segmentation. NSX ESG routes between logical networks and the physical firewall. Clean separation, straightforward to operate.

**Pattern 2: NSX Edge for load balancing, physical for perimeter**. NSX ESG runs the load balancer for internally-accessed web applications. The physical firewall still handles the internet perimeter. The ESG VIP accepts traffic from the physical firewall's destination NAT rules and distributes to backend VMs. This offloads load balancing from the physical device without changing perimeter security posture.

**Pattern 3: NSX for everything in VDI/dev environments, physical for production perimeter**. Different environments with different requirements can use different models. VDI environments and development clusters where east-west isolation matters but perimeter security is less critical can be NSX-native. Production environments with external exposure keep physical perimeter security while adding NSX for east-west.

## Operational Alignment

Whatever the design, ensure the security and network teams operating the physical firewalls and the team managing NSX are in close coordination. NSX security policies that complement (rather than duplicate or conflict with) physical firewall policies require both teams to understand the boundary and be involved in security policy changes that cross it.

The NSX administrator and the physical firewall administrator should have a documented policy on who owns which boundary, how changes are coordinated, and how troubleshooting works for traffic that crosses between NSX-controlled and physically-controlled segments.
