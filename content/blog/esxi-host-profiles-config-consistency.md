---
title: "ESXi Host Profiles: Configuration Consistency Across a Growing Cluster"
date: 2016-12-14
tags: ["VMware", "ESXi", "Host Profiles", "Configuration Management", "vSphere"]
---

In a two-host cluster, keeping ESXi hosts configured identically is manageable manually. By the time you have six or eight hosts, configuration drift is inevitable unless you have a systematic approach. Host Profiles are VMware's mechanism for capturing and enforcing ESXi host configuration state.

## What Host Profiles Cover

A Host Profile captures the configuration of a reference ESXi host and can apply that configuration to other hosts in the cluster. Configuration captured includes:

- Network configuration (vSwitches, port groups, VMkernel interfaces, NIC teaming policies)
- Storage configuration (iSCSI initiator settings, multipath policies, NFS datastores)
- Security settings (SSH enabled/disabled, lockdown mode, firewall rules)
- Advanced kernel parameters
- NTP server configuration
- Authentication (Active Directory join settings)

Host Profiles don't capture VM configurations or running state — they're focused on the hypervisor layer configuration.

## Creating a Profile

Designate one correctly configured host as your reference host — the "gold standard" configuration for your cluster. In vCenter: Home → Policies and Profiles → Host Profiles → Extract Profile from a Host. Select the reference host and extract the profile.

After extraction, review and edit the profile. Some settings from the reference host shouldn't be in the profile because they're host-specific (static IP addresses, for example). Edit the profile to mark IP address fields as "User must explicitly choose the policy option" rather than inheriting the reference host's specific IP. This creates a template where host-specific settings prompt for input during application.

## Compliance Checking

Attach the profile to a host cluster: in vCenter cluster settings, associate the Host Profile. Once attached, vCenter can check each host's compliance against the profile. Non-compliant hosts are flagged — settings that have drifted from the profile are identified specifically.

Running compliance checks after every host change (patch, configuration update) catches drift before it accumulates. I run compliance checks as part of the monthly operational review for client environments.

## The Remediation Process

When a host is non-compliant, you can remediate it: apply the profile's settings to the host. Remediation typically requires the host to be in maintenance mode (VMs evacuated). Some settings apply live without disruption; network reconfigurations usually require the host to be quiesced.

The important workflow: test profile remediation on one non-production host before applying to the entire cluster. Profile remediation errors (unexpected setting conflicts, missing required inputs) surface clearly in the remediation preview — review them before clicking apply.

## Where Host Profiles Help Most

The scenario where Host Profiles pay off most clearly: adding a new host to an existing cluster. Without profiles, configuring a new host to match the cluster standard requires manually replicating every setting — a process that's error-prone and that I've seen produce subtle misconfigurations that persist for months. With profiles, applying the cluster standard to a new host is a structured, verifiable process.

Host Profiles also help when a host needs to be rebuilt (hardware replacement, OS corruption). Apply the profile to the rebuilt host, provide the host-specific inputs (IP addresses, hostname), and the configuration is restored to standard quickly and completely.

Not a replacement for configuration management tools in complex environments, but for most SMB VMware deployments, Host Profiles provide adequate consistency management without additional tooling.
