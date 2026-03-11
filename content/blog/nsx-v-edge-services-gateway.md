---
title: "NSX-V Edge Services Gateways: The Part That Trips Up First-Timers"
date: 2019-03-07
tags: ["VMware", "NSX", "NSX-V", "Networking", "Edge", "Routing"]
---

If the Distributed Firewall is NSX-V's most-used feature, the Edge Services Gateway (ESG) is its most confusing. ESGs are software-defined appliances that provide routing, NAT, load balancing, VPN, and perimeter firewall functions. They're powerful, but the design decisions around ESG sizing and topology have long-term consequences that aren't obvious when you're setting one up for the first time.

## What an Edge Does

An NSX Edge can play multiple roles depending on deployment. In most configurations:

**Distributed Logical Router (DLR)**: Handles east-west routing between logical networks at the hypervisor kernel level. Extremely efficient — routing happens in the kernel, not through a VM. DLRs provide the default gateway for VMs on logical networks and connect logical segments to each other.

**Edge Services Gateway (ESG)**: A VM-based edge that handles north-south traffic — the boundary between your logical networking space and the physical network. ESGs handle BGP/OSPF peering with physical routers, SNAT/DNAT for internet-bound traffic, VPN termination, and load balancing.

The typical NSX topology: VMs connect to logical switches, DLRs route between logical switches and connect to an ESG, the ESG connects to the physical network and handles everything external. This is the hub-spoke model and it works well for most deployments.

## ESG Sizing: Get This Right at Deployment

ESGs come in four form factors: compact, large, quad-large, and x-large. They differ in vCPU, memory, and throughput capabilities. The temptation is to start with compact and upgrade if needed. Here's the problem: resizing an ESG requires redeployment, and ESG redeployment in a production environment means a brief traffic interruption for everything routed through that edge.

For production deployments, start with Large. The resources are minimal (4 vCPU, 1GB RAM), the cost is negligible relative to your VMware infrastructure, and you avoid a disruptive resize operation later.

For environments with high-throughput load balancing (hundreds of VPS/connections) or heavy NAT processing, consider Quad-Large or X-Large from the start. CPU is the typical bottleneck for ESG throughput, not memory.

## HA for ESGs

ESGs support an HA mode where a standby ESG instance sits in hot-standby alongside the active instance. If the active ESG fails, the standby takes over within seconds using a keepalive mechanism. Enable HA for any ESG in a production path.

HA ESG pairs require two hosts in the same cluster with anti-affinity rules to keep them separated. Don't put the active and standby ESG on the same host — the host failure scenario you're protecting against would take both out simultaneously.

## The Routing Integration Problem

The trickiest part of NSX-V ESG deployments in SMB environments: integrating logical network routing with the physical network. Your ESG needs to advertise the IP prefixes used by your logical networks to the physical routers, and physical routers need to route traffic destined for logical network IPs to the ESG.

For environments with simple physical routing (a single firewall or router, static routes acceptable), static routing between the ESG and the physical router is straightforward. For environments with multiple routing hops or dynamic routing requirements, NSX supports OSPF and BGP on the ESG for dynamic route advertisement.

The common mistake: deploying NSX with logical networks using IP ranges that overlap with existing physical network ranges. When the ESG tries to route traffic for those prefixes, the physical network can't route it correctly because the same range is visible both physically and logically. Plan IP address space carefully — use non-overlapping ranges for logical networks and document the separation clearly.

## Load Balancer: One-Armed vs. Inline

The NSX ESG load balancer supports two topologies. In-line (transparent) mode routes client traffic through the ESG before and after the backend servers — the ESG is in the traffic path and can see full connection state. One-armed mode deploys the load balancer with a single interface, using SNAT to direct traffic to backends — simpler to deploy but changes the source IP that backend servers see.

For most SMB use cases, one-armed with SNAT is simpler and more manageable. In-line mode is appropriate when source IP preservation to backends is required for logging or access control purposes.
