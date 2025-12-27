---
name: hyper-v-expert
description: Microsoft Hyper-V virtualization expertise for Windows Server environments. Use when Claude needs to assist with (1) Installing and configuring Hyper-V hosts, (2) Creating and managing virtual machines, (3) Virtual networking - switches, VLANs, NAT, (4) Storage - VHD/VHDX, checkpoints, shared disks, (5) Live migration and high availability, (6) Hyper-V Replica for disaster recovery, (7) GPU acceleration - DDA and GPU partitioning, (8) Security - shielded VMs, Secure Boot, encryption, (9) PowerShell automation for Hyper-V, or any other Hyper-V administration and implementation tasks.
---

# Hyper-V Expert

Comprehensive Microsoft Hyper-V administration and implementation expertise for Windows Server virtualization.

## Quick Reference

| Task | Reference File |
|------|----------------|
| Install Hyper-V, create VMs | [deployment.md](references/deployment.md) |
| Live migration, integration services | [management.md](references/management.md) |
| Virtual switches, VLANs, NAT | [networking.md](references/networking.md) |
| VHD/VHDX, checkpoints, Fibre Channel | [storage.md](references/storage.md) |
| Shielded VMs, Secure Boot, encryption | [security.md](references/security.md) |
| DDA, GPU partitioning | [gpu-acceleration.md](references/gpu-acceleration.md) |
| Hyper-V Replica, failover | [replication.md](references/replication.md) |
| Cmdlets, automation patterns | [powershell.md](references/powershell.md) |

## Core Workflows

### Install Hyper-V

```powershell
# Windows Server
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart

# Windows 10/11
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

### Create Virtual Machine

```powershell
# Create Gen 2 VM with new disk
New-VM -Name "VMName" -Generation 2 -MemoryStartupBytes 4GB `
    -NewVHDPath "C:\VMs\disk.vhdx" -NewVHDSizeBytes 60GB `
    -SwitchName "vSwitch"

# Mount ISO and start
Add-VMDvdDrive -VMName "VMName" -Path "C:\ISOs\windows.iso"
Start-VM -VMName "VMName"
```

### Create Virtual Switch

```powershell
# External (internet access)
New-VMSwitch -Name "External" -NetAdapterName "Ethernet" -AllowManagementOS $true

# Internal (host + VMs only)
New-VMSwitch -Name "Internal" -SwitchType Internal

# Private (VMs only)
New-VMSwitch -Name "Private" -SwitchType Private
```

### Enable Live Migration

```powershell
Enable-VMMigration
Set-VMMigrationNetwork 192.168.1.0/24
Set-VMHost -VirtualMachineMigrationAuthenticationType Kerberos

# Move VM
Move-VM -Name "VMName" -DestinationHost "DestHost"
```

### Create Checkpoint

```powershell
# Production checkpoint (recommended)
Set-VM -Name "VMName" -CheckpointType Production
Checkpoint-VM -VMName "VMName" -SnapshotName "Before Update"

# Restore
Restore-VMCheckpoint -VMName "VMName" -Name "Before Update" -Confirm:$false
```

### Enable Hyper-V Replica

```powershell
# On replica server
Set-VMReplicationServer -ReplicationEnabled $true `
    -AllowedAuthenticationType Kerberos `
    -DefaultStorageLocation "D:\Replicas"

# On primary - enable replication
Enable-VMReplication -VMName "VMName" `
    -ReplicaServerName "ReplicaHost.domain.com" `
    -ReplicaServerPort 80 `
    -AuthenticationType Kerberos

# Start initial replication
Start-VMInitialReplication -VMName "VMName"
```

## Decision Trees

### VM Generation Selection

```
Is guest OS Windows XP/Server 2003 or older?
├─ Yes → Use Generation 1
└─ No → Is it a 32-bit OS?
         ├─ Yes → Use Generation 1
         └─ No → Use Generation 2 (recommended)
```

### Storage Format Selection

```
Is this for a guest cluster (shared storage)?
├─ Yes → Use VHD Set (.vhds)
└─ No → Do you need legacy compatibility?
         ├─ Yes → Use VHD (max 2TB)
         └─ No → Use VHDX (max 64TB, recommended)
```

### GPU Acceleration Selection

```
Need dedicated GPU per VM with max performance?
├─ Yes → Use Discrete Device Assignment (DDA)
└─ No → Need to share GPU across multiple VMs?
         ├─ Yes (Server 2025+) → Use GPU Partitioning (GPU-P)
         └─ No → Use software rendering
```

### Scheduler Type Selection

```
Are you running untrusted workloads?
├─ Yes → Use Core scheduler (security isolation)
└─ No → Is SMT/hyperthreading enabled?
         ├─ Yes → Use Core scheduler (recommended)
         └─ No → Classic scheduler is acceptable
```

## Best Practices

### Host Configuration

- Use Server Core for reduced attack surface
- Dedicate separate NICs for management and VM traffic
- Enable BitLocker on storage volumes
- Apply Windows Security Baselines
- Configure antivirus exclusions for Hyper-V files

### VM Configuration

- Use Generation 2 for modern OSes
- Enable Secure Boot
- Enable vTPM for BitLocker support
- Use Production checkpoints for production VMs
- Configure integration services appropriately

### Networking

- Use SET (Switch Embedded Teaming) over NIC teaming
- Isolate management traffic from VM traffic
- Use VLANs for network segmentation
- Enable bandwidth management for critical VMs

### Storage

- Use VHDX format (64TB max, resilient)
- Place VHDs on separate physical volumes from OS
- Use fixed-size VHDs for performance-critical VMs
- Enable Storage QoS for shared storage

## Troubleshooting

### VM Won't Start

```powershell
# Check VM status and notes
Get-VM -Name "VMName" | Format-List *

# Check event log
Get-WinEvent -FilterHashTable @{LogName="Microsoft-Windows-Hyper-V-Worker-Admin"} -MaxEvents 20
```

### Network Connectivity Issues

```powershell
# Verify switch connection
Get-VMNetworkAdapter -VMName "VMName"

# Check VLAN settings
Get-VMNetworkAdapterVlan -VMName "VMName"

# Test from host
Test-NetConnection -ComputerName "VM-IP"
```

### Replication Errors

```powershell
# Check replication health
Get-VMReplication -VMName "VMName"

# Measure replication statistics
Measure-VMReplication -VMName "VMName"

# Force resync
Resume-VMReplication -VMName "VMName" -Resynchronize
```

### Performance Issues

```powershell
# Enable resource metering
Enable-VMResourceMetering -VMName "VMName"

# Check resource usage
Measure-VM -VMName "VMName"

# Check scheduler type
Get-WinEvent -FilterHashTable @{ProviderName="Microsoft-Windows-Hyper-V-Hypervisor"; ID=2} -MaxEvents 1
```

## External Resources

- [Official Hyper-V Documentation](https://github.com/MicrosoftDocs/windowsserverdocs/tree/main/WindowsServerDocs/virtualization/hyper-v)
- [Hyper-V PowerShell Module](https://learn.microsoft.com/en-us/powershell/module/hyper-v/)
- [Windows Server Documentation](https://learn.microsoft.com/en-us/windows-server/)
