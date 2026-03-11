---
title: "vSphere Update Manager: First Deployment Lessons Learned"
date: 2016-09-06
tags: ["VMware", "vSphere", "Update Manager", "VUM", "Patching"]
---

vSphere Update Manager (VUM) is the centralized patch management component for ESXi hosts. When it works, patching 20 ESXi hosts is barely more effort than patching one. When it's misconfigured or fighting with your environment, it creates a new category of problems. Here's what to know before your first VUM deployment.

## VUM Architecture

In vSphere 6.0 and earlier, VUM was a separate Windows application that installed alongside Windows-based vCenter. In vSphere 6.5+, VUM is integrated into the VCSA as the vSphere Lifecycle Manager service — no separate installation required. If you're on VCSA 6.5+, VUM is already there; you just need to configure it.

VUM requires internet access (or a local patch depot) to download ESXi patches and upgrades from VMware's patch repository. If your vCenter appliance sits in a network segment without direct internet access (common in security-conscious environments), configure a proxy in the VUM settings or set up an offline patch depot on a server that can reach the internet.

## Creating Baselines

VUM manages patching through baselines — collections of patches or upgrades that you want to apply to hosts. The three baseline types:

**Patch baseline**: A collection of specific patches. Dynamic baselines automatically include new patches as they're released; static baselines fix a specific set.

**Extension baseline**: For VIBs (third-party drivers, tools) that aren't part of the standard ESXi patch track.

**Upgrade baseline**: For major version upgrades (5.5 to 6.5, etc.).

For ongoing patch management, create a dynamic patch baseline that includes all critical and important security patches. Attach it to your host clusters and scan regularly.

## The Remediation Process

Patching with VUM follows three steps: scan, review, remediate. Scan checks which hosts are non-compliant (missing patches from the attached baseline). Review shows you what patches will be applied and any pre-check warnings. Remediate applies the patches.

Remediation puts each host in maintenance mode (migrating VMs off via DRS), applies patches, reboots if necessary, exits maintenance mode, and moves to the next host. For a four-host cluster, expect 30-60 minutes per host depending on patch size and VM evacuation time.

**The sequencing matter**: Remediate one host at a time, not all simultaneously. Remediating all hosts simultaneously puts your entire cluster into maintenance mode, taking down all VMs. VUM's default behavior is sequential, but verify this in your remediation settings before clicking apply.

## Common VUM Problems

**Scan failures**: The most common — VUM can't connect to a host to assess compliance. Usually a network connectivity issue (management network, firewall between VUM and the host) or an authentication problem. Check the VUM log at `/var/log/vmware/updatemgr/` on VCSA.

**Proxy/repo connectivity**: If VUM reports it can't download patches, verify the proxy configuration or internet connectivity from the vCenter appliance. A simple curl from the VCSA shell to vmware.com will confirm connectivity.

**VIB conflicts at remediation time**: Pre-remediation checks should catch these, but sometimes they surface during remediation itself. VUM will abort the remediation on the affected host and roll back — the host comes back to its previous state. Investigate the conflicting VIB and resolve before retrying.

Document your patch baseline configuration, remediation schedule (I recommend monthly for security patches, quarterly for everything else), and the list of any VIBs excluded from baselines. This documentation makes every future patching event faster and prevents "why isn't X host being updated" questions.
