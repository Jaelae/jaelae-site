---
title: "PowerCLI Basics for VMware Administrators"
date: 2019-10-22
tags: ["VMware", "PowerCLI", "Automation", "PowerShell"]
---

If you manage VMware infrastructure and you're not using PowerCLI, you're doing the same tasks manually that a few lines of PowerShell would automate. PowerCLI is VMware's PowerShell module for vSphere management, and once you get comfortable with the basics, it becomes an essential tool for everything from inventory reporting to bulk configuration changes.

## Getting Started

Install PowerCLI from the PowerShell Gallery — the clean, current way:

```powershell
Install-Module -Name VMware.PowerCLI -Scope CurrentUser
```

Connect to your vCenter:

```powershell
Connect-VIServer -Server vcenter.company.com -Credential (Get-Credential)
```

For scripted connections without interactive credential prompts, store credentials securely (PowerShell credential export or a secrets manager) and pass them programmatically.

## Essential Commands

A few PowerCLI commands that I use constantly:

**Get-VM**: Lists VMs. Combine with `-Name` for specific VMs or leave without arguments for all. Pipe to `Select` for specific properties:

```powershell
Get-VM | Select Name, PowerState, NumCpu, MemoryGB | Sort Name
```

**Get-VMHost**: Lists ESXi hosts. Useful for inventory and checking host status:

```powershell
Get-VMHost | Select Name, ConnectionState, Version, MemoryTotalGB, MemoryUsageGB
```

**Get-Datastore**: Lists datastores and their usage:

```powershell
Get-Datastore | Select Name, CapacityGB, FreeSpaceGB, @{N='UsedGB';E={[math]::Round($_.CapacityGB - $_.FreeSpaceGB,1)}} | Sort FreeSpaceGB
```

This is the command I run first when someone reports storage issues — it shows every datastore's free space instantly across the entire cluster.

## Practical Scripts I Use Regularly

**Find all VMs with snapshots**:

```powershell
Get-VM | Get-Snapshot | Select VM, Name, Created, SizeGB | Sort SizeGB -Descending
```

Snapshot accumulation is a quiet killer in VMware environments. This reveals forgotten snapshots that are consuming datastore space.

**Check all VMs without VMware Tools installed**:

```powershell
Get-VM | Where-Object {$_.ExtensionData.Guest.ToolsStatus -ne 'toolsOk'} | Select Name, @{N='ToolsStatus';E={$_.ExtensionData.Guest.ToolsStatus}}
```

**Get VMs with CPU or memory reservations set** (reservations affect DRS balancing):

```powershell
Get-VM | Where-Object {$_.ExtensionData.ResourceConfig.CpuAllocation.Reservation -gt 0 -or $_.ExtensionData.ResourceConfig.MemoryAllocation.Reservation -gt 0} | Select Name
```

## Bulk Operations

PowerCLI shines for bulk operations. Examples:

**Set VM startup policy for all VMs in a folder**:

```powershell
Get-Folder "Production" | Get-VM | Get-VMStartPolicy | Set-VMStartPolicy -StartAction PowerOn -StartOrder 1
```

**Move all VMs from one datastore to another** (requires vMotion/Storage vMotion):

```powershell
Get-Datastore "OldDatastore" | Get-VM | Move-VM -Datastore "NewDatastore"
```

**Apply a tag to all VMs matching a naming pattern**:

```powershell
Get-VM | Where-Object {$_.Name -like "WEB-*"} | New-TagAssignment -Tag (Get-Tag "tier-web")
```

## Building Scheduled Reports

One of the most valuable uses of PowerCLI in SMB environments: scheduled health reports. A script that emails you a summary of datastore space, snapshot status, host health, and powered-off VMs every morning takes an hour to write and saves that time every week indefinitely.

```powershell
$report = Get-Datastore | Select Name, 
    @{N='FreePct';E={[math]::Round(($_.FreeSpaceGB/$_.CapacityGB)*100,1)}} |
    Where-Object {$_.FreePct -lt 30}

if ($report) {
    Send-MailMessage -To "admin@company.com" -Subject "Datastore Alert" -Body ($report | Out-String)
}
```

That's a basic datastore space alert script. The output is readable, the logic is simple, and it's been running unmodified in multiple client environments for years.
