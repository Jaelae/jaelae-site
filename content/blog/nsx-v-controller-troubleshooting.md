---
title: "Troubleshooting NSX-V: Controller Issues and What They Actually Mean"
date: 2019-01-22
tags: ["VMware", "NSX", "NSX-V", "Troubleshooting", "Controllers"]
---

NSX-V's control plane runs on a cluster of controller VMs that manage logical network state — VXLAN mappings, logical router tables, and distributed firewall rule distribution. When controllers have problems, the symptoms are usually indirect and confusing: VMs can't reach each other, logical networks stop working, or new VM deployments fail network connectivity. Here's how to diagnose controller issues systematically.

## The Controller Cluster Architecture

NSX-V requires a cluster of three controller VMs for production operation. Three controllers provide quorum — loss of one controller is tolerated, but the cluster continues operating. With only two healthy controllers, the cluster is degraded. With one or zero, network state distribution stops.

Controllers don't run the data plane — traffic between VMs doesn't physically pass through controllers. But controllers maintain the distributed database that tells each ESXi host which VXLAN tunnel endpoint (VTEP) holds which VM MAC address. Without this information, VMs on different hosts can't communicate on logical networks.

## Checking Controller Health

The first step in any NSX-V network troubleshooting: check controller health via the NSX Manager UI or REST API:

In the vSphere Web Client under Networking & Security → Installation → Management → NSX Controller Nodes, all three controllers should show Status: Connected.

From the NSX Manager API:

```
GET /api/2.0/vdn/controller
```

Each controller returns its connection state and the master election status. One controller will be the master (coordinating state across the cluster); the others are replicas.

## Split-Brain and Controller Failures

The dangerous scenario: one controller becomes isolated from the others (network partition, host failure) while still being reachable from some ESXi hosts. This can cause inconsistent network state — some hosts get updates from the isolated controller, others from the healthy majority. The symptom is typically intermittent VM connectivity failures that don't reproduce consistently.

NSX handles this through majority quorum — only the majority partition (two controllers) continues distributing state. The minority (one isolated controller) stops accepting writes. But hosts that were communicating with the isolated controller may have cached state that diverges from the majority until they reconnect.

If you suspect split-brain, the resolution is restoring controller connectivity and allowing state resynchronization. NSX controllers resync automatically once network connectivity is restored.

## Deploying Replacement Controllers

If a controller VM is permanently lost (host failure, disk corruption), you'll need to deploy a replacement. The procedure in NSX Manager: delete the failed controller from the cluster, deploy a new controller (specify the same credentials, data network, and storage), and allow the new controller to join the cluster and resync state.

During the replacement process, you're temporarily running two controllers. Network data plane continues functioning (data plane state is cached on ESXi hosts), but new logical network operations (new VM deployments, configuration changes) may queue until the cluster returns to three members.

## Controller to Host Communication

Controllers communicate with ESXi hosts via SSL over the management network. If your management network has routing issues or firewall rules blocking controller-to-host traffic (TCP 1234 is the primary port), hosts won't receive updated network state. Symptom: VMs that were running before continue communicating (cached state), but newly deployed VMs on logical networks can't reach others.

Test connectivity from an ESXi host to a controller:

```bash
nc -z <controller-ip> 1234
```

A successful connection confirms the communication path is working. A timeout or connection refused points to a routing or firewall issue between host and controller networks.

## When to Involve VMware Support

NSX controller recovery operations — particularly in failure scenarios involving database inconsistency or multi-controller failures — should involve VMware support. Incorrect controller remediation can compound the problem. When you're past basic health checks and straightforward controller replacement, open a support case before proceeding with advanced remediation steps.
