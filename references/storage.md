# Hyper-V Storage Reference

## Table of Contents
- [Virtual Hard Disks](#virtual-hard-disks)
- [Checkpoints](#checkpoints)
- [Storage QoS](#storage-qos)
- [Virtual Fibre Channel](#virtual-fibre-channel)
- [Persistent Memory](#persistent-memory)
- [Shared VHDX and VHD Sets](#shared-vhdx-and-vhd-sets)

---

## Virtual Hard Disks

### Formats

| Format | Max Size | Features |
|--------|----------|----------|
| VHD | 2 TB | Legacy, Gen 1 compatible |
| VHDX | 64 TB | Resilient, Gen 2 recommended |
| VHDS | 64 TB | Shared disk for guest clusters |

### Create VHDs

```powershell
# Fixed size VHDX (best performance)
New-VHD -Path "C:\VMs\disk.vhdx" -SizeBytes 100GB -Fixed

# Dynamic VHDX (grows as needed)
New-VHD -Path "C:\VMs\disk.vhdx" -SizeBytes 100GB -Dynamic

# Differencing disk (based on parent)
New-VHD -Path "C:\VMs\diff.vhdx" -ParentPath "C:\VMs\base.vhdx" -Differencing
```

### Manage VHDs

```powershell
# Get VHD info
Get-VHD -Path "C:\VMs\disk.vhdx"

# Resize VHD (VM must be off or disk must be unattached)
Resize-VHD -Path "C:\VMs\disk.vhdx" -SizeBytes 200GB

# Compact dynamic VHD
Optimize-VHD -Path "C:\VMs\disk.vhdx" -Mode Full

# Convert VHD to VHDX
Convert-VHD -Path "C:\old.vhd" -DestinationPath "C:\new.vhdx"

# Convert dynamic to fixed
Convert-VHD -Path "C:\dynamic.vhdx" -DestinationPath "C:\fixed.vhdx" -VHDType Fixed
```

### Attach/Detach VHDs

```powershell
# Add disk to VM
Add-VMHardDiskDrive -VMName "VMName" -Path "C:\VMs\disk.vhdx"

# Add to specific controller
Add-VMHardDiskDrive -VMName "VMName" -ControllerType SCSI -ControllerNumber 0 -Path "C:\VMs\disk.vhdx"

# Remove disk from VM
Remove-VMHardDiskDrive -VMName "VMName" -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 1

# Mount VHD on host
Mount-VHD -Path "C:\VMs\disk.vhdx"

# Dismount from host
Dismount-VHD -Path "C:\VMs\disk.vhdx"
```

---

## Checkpoints

### Types

| Type | Description | Use Case |
|------|-------------|----------|
| Standard | Saves VM state and memory | Development, testing |
| Production | Uses VSS/fsfreeze for consistency | Production workloads |

### Configure Checkpoint Type

```powershell
# Set to production checkpoints
Set-VM -Name "VMName" -CheckpointType Production

# Set to standard checkpoints
Set-VM -Name "VMName" -CheckpointType Standard

# Disable checkpoints
Set-VM -Name "VMName" -CheckpointType Disabled

# Production with fallback to standard
Set-VM -Name "VMName" -CheckpointType ProductionOnly
```

### Manage Checkpoints

```powershell
# Create checkpoint
Checkpoint-VM -Name "VMName" -SnapshotName "BeforeUpdate"

# List checkpoints
Get-VMCheckpoint -VMName "VMName"

# Restore to checkpoint
Restore-VMCheckpoint -VMName "VMName" -Name "BeforeUpdate" -Confirm:$false

# Delete checkpoint
Remove-VMCheckpoint -VMName "VMName" -Name "BeforeUpdate"

# Delete all checkpoints
Get-VMCheckpoint -VMName "VMName" | Remove-VMCheckpoint
```

### Checkpoint Location

```powershell
# Set checkpoint location
Set-VM -Name "VMName" -SnapshotFileLocation "D:\Checkpoints"

# View current location
(Get-VM -Name "VMName").SnapshotFileLocation
```

---

## Storage QoS

### Enable Storage QoS

```powershell
# Set minimum IOPS
Set-VMHardDiskDrive -VMName "VMName" -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 0 -MinimumIOPS 100

# Set maximum IOPS
Set-VMHardDiskDrive -VMName "VMName" -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 0 -MaximumIOPS 1000
```

### Storage QoS Policies (Scale-Out File Server)

```powershell
# Create policy
New-StorageQosPolicy -Name "GoldPolicy" -MinimumIops 1000 -MaximumIops 5000 -PolicyType Dedicated

# Apply policy to VHD
Get-VM "VMName" | Get-VMHardDiskDrive | Set-VMHardDiskDrive -QoSPolicyID (Get-StorageQosPolicy -Name "GoldPolicy").PolicyId

# View policies
Get-StorageQosPolicy
```

---

## Virtual Fibre Channel

### Requirements

- Windows Server 2012+ host
- Fibre Channel HBAs with NPIV support
- NPIV-enabled SAN

### Configure Virtual SAN

```powershell
# Create virtual SAN
New-VMSan -Name "VirtualSAN" -WorldWideNodeNameSetA "C003FF0000FFFF00" -WorldWidePortNameSetA "C003FF0000FFFF01" -WorldWideNodeNameSetB "C003FF0000FFFF02" -WorldWidePortNameSetB "C003FF0000FFFF03"

# Add physical HBA to virtual SAN
Add-VMSanHba -SanName "VirtualSAN" -HostBusAdapter (Get-InitiatorPort)
```

### Add Fibre Channel Adapter to VM

```powershell
# Add virtual FC adapter
Add-VMFibreChannelHba -VMName "VMName" -SanName "VirtualSAN"

# View FC adapters
Get-VMFibreChannelHba -VMName "VMName"

# Remove FC adapter
Remove-VMFibreChannelHba -VMName "VMName" -WorldWideNodeNameSetA "C003FF0000FFFF00"
```

### WWN Management

Each virtual FC adapter has two WWN sets (A and B) for live migration support:
- Set A: Active during normal operation
- Set B: Active during migration to destination

---

## Persistent Memory

### Requirements

- Windows Server 2019+ host
- Gen 2 VMs only
- NTFS DAX volume on host
- Not compatible with Live Migration or production checkpoints

### Configure Persistent Memory

```powershell
# Create persistent memory device
New-VHD -Path "D:\DAXVolume\pmem.vhdpmem" -SizeBytes 4GB -Fixed

# Add controller to VM
Add-VMPmemController -VMName "VMName"

# Attach persistent memory device
Add-VMHardDiskDrive -VMName "VMName" -ControllerType PMEM -ControllerNumber 0 -Path "D:\DAXVolume\pmem.vhdpmem"
```

---

## Shared VHDX and VHD Sets

For guest clustering scenarios.

### VHD Sets (.vhds)

Introduced in Windows Server 2016, supports:
- Online resizing
- Hyper-V Replica
- Application-consistent checkpoints

```powershell
# Create VHD Set
New-VHD -Path "C:\ClusterStorage\shared.vhds" -SizeBytes 100GB -Fixed

# Convert VHDX to VHDS
Convert-VHD -Path "C:\existing.vhdx" -DestinationPath "C:\shared.vhds"

# Attach shared disk
Add-VMHardDiskDrive -VMName "ClusterNode1" -Path "C:\ClusterStorage\shared.vhds" -SupportPersistentReservations
Add-VMHardDiskDrive -VMName "ClusterNode2" -Path "C:\ClusterStorage\shared.vhds" -SupportPersistentReservations
```

### Shared VHDX (Legacy)

```powershell
# Attach as shared
Add-VMHardDiskDrive -VMName "VMName" -Path "\\FileServer\Share\shared.vhdx" -SupportPersistentReservations
```

> **Note:** VHD Sets (.vhds) are preferred over shared VHDX for new deployments.
