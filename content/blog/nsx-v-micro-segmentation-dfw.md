---
title: "NSX-V Micro-Segmentation: Applying Distributed Firewall in Practice"
date: 2018-12-10
tags: ["VMware", "NSX", "NSX-V", "Security", "Micro-Segmentation", "Firewall"]
---

The distributed firewall is the feature that justifies NSX-V for most mid-size organizations. Traditional network firewalls sit at the perimeter or at layer boundaries — north-south traffic is controlled, but east-west traffic between VMs on the same network or VLAN moves freely. NSX's DFW enforces policy at the vNIC level, so every packet between VMs passes through a firewall regardless of whether they're on the same logical network.

## Security Groups and Dynamic Membership

The power of NSX DFW comes from security groups with dynamic membership rules. Instead of writing rules based on IP addresses (which change), you write rules based on VM attributes: vCenter tags, VM names, folder membership, OS type, or Security Group membership itself.

A practical example: you want to allow web servers to talk to application servers on TCP 8080, but not allow web servers to talk directly to database servers. In IP-based firewall terms, this means maintaining IP lists and updating rules every time a server's IP changes. In NSX security groups, you create `sg-web`, `sg-app`, and `sg-db` groups with dynamic membership rules that match VMs based on vCenter tags you apply. The firewall rule references the groups — IP addresses are irrelevant.

When you deploy a new web server VM and tag it `tier=web` in vCenter, it automatically joins `sg-web` and the firewall rules apply to it immediately, without any firewall configuration change.

## Rule Order and Processing

DFW rules are processed top-to-bottom within a section, and sections are processed in order. Rule order matters — a permit rule above a deny rule means the permit wins. Plan your rule hierarchy before authoring:

1. **Infrastructure rules** (allow VMs to reach DNS, NTP, management tools — exceptions that apply everywhere)
2. **Emergency rules** (break-glass allows that override everything below during incidents)
3. **Application-tier rules** (the specific east-west controls you care about)
4. **Default deny** (or rely on the section-level default)

Never modify the default rule at the bottom to deny-all without extensive testing. It applies to every VM in the environment — a misconfiguration there is a cluster-wide outage.

## The Rule Application Problem

The most operationally significant lesson from NSX DFW deployments: applying rules before you know what traffic the application actually uses creates more problems than it solves.

The correct approach: enable DFW in monitor mode first (or start with a default-allow policy and use the Activity Monitoring feature to observe traffic). NSX Activity Monitoring logs observed connections to/from VMs — you can use this data to understand actual application dependencies before writing any deny rules.

I've seen environments where DFW was enabled with assume-deny policies without an activity monitoring phase, cutting off applications from their dependencies in ways that took days to fully map and fix. The monitoring investment upfront saves enormous troubleshooting time.

## Service Composer and Endpoint Security

NSX Service Composer lets you create security policies that combine DFW rules with third-party security services (network inspection, endpoint security) into a unified policy. If you're integrating NSX with a network security vendor (Palo Alto, Check Point, or others that have NSX integration), Service Composer is how you wire it together.

For SMB deployments without third-party NSX integration partners, Service Composer is optional complexity. Start with basic DFW rules, prove the value, and add Service Composer if you later integrate partner security services.

## Operational Monitoring

Once DFW is running, you need visibility into what it's doing. NSX provides Flow Monitoring — a traffic analysis tool in the NSX UI that shows observed connections between VMs and whether they were permitted or denied. Use Flow Monitoring actively during the first few weeks after DFW deployment to catch unexpected denies.

Unexpected DFW denies often surface as intermittent application failures where nothing obvious changed from the application team's perspective. Flow Monitoring turns "something's wrong with the application" into "DFW denied traffic from app-server-01 to db-server-03 on port 1433 at 14:33 UTC" — a specific, actionable event.
