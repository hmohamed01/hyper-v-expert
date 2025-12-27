# Hyper-V PowerShell Reference

## Table of Contents
- [Module and Remoting](#module-and-remoting)
- [VM Lifecycle](#vm-lifecycle)
- [VM Configuration](#vm-configuration)
- [Networking](#networking)
- [Storage](#storage)
- [Snapshots and Checkpoints](#snapshots-and-checkpoints)
- [Migration](#migration)
- [Monitoring](#monitoring)
- [Common Patterns](#common-patterns)

---

## Module and Remoting

### Import Module

```powershell
# Import Hyper-V module
Import-Module Hyper-V

# List available commands
Get-Command -Module Hyper-V

# Get help
Get-Help New-VM -Detailed
```

### Remote Management

```powershell
# Connect to remote host
Enter-PSSession -ComputerName "HyperVHost" -Credential (Get-Credential)

# Run command on remote host
Invoke-Command -ComputerName "HyperVHost" -ScriptBlock {
    Get-VM | Select Name, State
}

# Use -ComputerName parameter
Get-VM -ComputerName "HyperVHost"
```

### PowerShell Direct (Guest Access)

```powershell
# Enter guest session
Enter-PSSession -VMName "VMName" -Credential (Get-Credential)

# Run script in guest
Invoke-Command -VMName "VMName" -Credential $cred -ScriptBlock {
    Get-Service
}

# Copy file to guest
Copy-VMFile -Name "VMName" -SourcePath "C:\file.txt" `
    -DestinationPath "C:\file.txt" -CreateFullPath -FileSource Host
```

---

## VM Lifecycle

### Create VM

```powershell
# Basic VM
New-VM -Name "VMName" -MemoryStartupBytes 4GB -Generation 2

# Full configuration
New-VM -Name "VMName" `
    -MemoryStartupBytes 4GB `
    -Generation 2 `
    -NewVHDPath "C:\VMs\VMName.vhdx" `
    -NewVHDSizeBytes 60GB `
    -SwitchName "vSwitch" `
    -Path "C:\VMs"
```

### Start/Stop/Restart

```powershell
# Start
Start-VM -Name "VMName"

# Stop (graceful)
Stop-VM -Name "VMName"

# Stop (force)
Stop-VM -Name "VMName" -Force

# Turn off (power off)
Stop-VM -Name "VMName" -TurnOff

# Restart
Restart-VM -Name "VMName"

# Suspend/Resume
Suspend-VM -Name "VMName"
Resume-VM -Name "VMName"
```

### Delete VM

```powershell
# Remove VM only (keeps VHDs)
Remove-VM -Name "VMName" -Force

# Remove VM and VHDs
$vm = Get-VM -Name "VMName"
$vhds = $vm | Get-VMHardDiskDrive | Select-Object -ExpandProperty Path
Remove-VM -Name "VMName" -Force
$vhds | ForEach-Object { Remove-Item $_ }
```

---

## VM Configuration

### Memory

```powershell
# Static memory
Set-VMMemory -VMName "VMName" -StartupBytes 4GB

# Dynamic memory
Set-VMMemory -VMName "VMName" `
    -DynamicMemoryEnabled $true `
    -MinimumBytes 1GB `
    -StartupBytes 2GB `
    -MaximumBytes 8GB `
    -Buffer 20
```

### Processor

```powershell
# Set vCPU count
Set-VMProcessor -VMName "VMName" -Count 4

# CPU reservation and limit
Set-VMProcessor -VMName "VMName" `
    -Reserve 10 `
    -Maximum 80 `
    -RelativeWeight 100

# Enable nested virtualization
Set-VMProcessor -VMName "VMName" -ExposeVirtualizationExtensions $true

# NUMA configuration
Set-VMProcessor -VMName "VMName" `
    -MaximumCountPerNumaNode 4 `
    -MaximumCountPerNumaSocket 8
```

### Firmware and Boot

```powershell
# Get firmware settings
Get-VMFirmware -VMName "VMName"

# Set boot order
$dvd = Get-VMDvdDrive -VMName "VMName"
$hdd = Get-VMHardDiskDrive -VMName "VMName"
Set-VMFirmware -VMName "VMName" -BootOrder $dvd, $hdd

# Enable Secure Boot
Set-VMFirmware -VMName "VMName" -EnableSecureBoot On
```

### Integration Services

```powershell
# List services
Get-VMIntegrationService -VMName "VMName"

# Enable/Disable
Enable-VMIntegrationService -VMName "VMName" -Name "Guest Service Interface"
Disable-VMIntegrationService -VMName "VMName" -Name "Time Synchronization"
```

---

## Networking

### Virtual Switch

```powershell
# Create switches
New-VMSwitch -Name "External" -NetAdapterName "Ethernet"
New-VMSwitch -Name "Internal" -SwitchType Internal
New-VMSwitch -Name "Private" -SwitchType Private

# List switches
Get-VMSwitch

# Remove switch
Remove-VMSwitch -Name "SwitchName"
```

### Network Adapters

```powershell
# Add adapter
Add-VMNetworkAdapter -VMName "VMName" -SwitchName "vSwitch"

# Connect/Disconnect
Connect-VMNetworkAdapter -VMName "VMName" -SwitchName "vSwitch"
Disconnect-VMNetworkAdapter -VMName "VMName"

# Set MAC address
Set-VMNetworkAdapter -VMName "VMName" -StaticMacAddress "00-15-5D-00-00-01"

# Configure VLAN
Set-VMNetworkAdapterVlan -VMName "VMName" -Access -VlanId 100
```

---

## Storage

### VHD Operations

```powershell
# Create VHD
New-VHD -Path "C:\disk.vhdx" -SizeBytes 100GB -Dynamic
New-VHD -Path "C:\disk.vhdx" -SizeBytes 100GB -Fixed

# Resize VHD
Resize-VHD -Path "C:\disk.vhdx" -SizeBytes 200GB

# Get VHD info
Get-VHD -Path "C:\disk.vhdx"

# Compact VHD
Optimize-VHD -Path "C:\disk.vhdx" -Mode Full
```

### Attach/Detach Disks

```powershell
# Add hard disk
Add-VMHardDiskDrive -VMName "VMName" -Path "C:\disk.vhdx"

# Add to specific controller
Add-VMHardDiskDrive -VMName "VMName" `
    -ControllerType SCSI `
    -ControllerNumber 0 `
    -Path "C:\disk.vhdx"

# Remove disk
Remove-VMHardDiskDrive -VMName "VMName" `
    -ControllerType SCSI `
    -ControllerNumber 0 `
    -ControllerLocation 1
```

### DVD Drive

```powershell
# Add DVD drive
Add-VMDvdDrive -VMName "VMName"

# Mount ISO
Set-VMDvdDrive -VMName "VMName" -Path "C:\iso\windows.iso"

# Eject
Set-VMDvdDrive -VMName "VMName" -Path $null
```

---

## Snapshots and Checkpoints

```powershell
# Create checkpoint
Checkpoint-VM -VMName "VMName" -SnapshotName "Before Update"

# List checkpoints
Get-VMCheckpoint -VMName "VMName"

# Restore checkpoint
Restore-VMCheckpoint -VMName "VMName" -Name "Before Update" -Confirm:$false

# Delete checkpoint
Remove-VMCheckpoint -VMName "VMName" -Name "Before Update"

# Set checkpoint type
Set-VM -Name "VMName" -CheckpointType Production
```

---

## Migration

### Live Migration

```powershell
# Enable migration
Enable-VMMigration

# Move VM to another host
Move-VM -Name "VMName" -DestinationHost "DestHost"

# Move with storage
Move-VM -Name "VMName" -DestinationHost "DestHost" `
    -DestinationStoragePath "D:\VMs"
```

### Storage Migration

```powershell
# Move storage only
Move-VMStorage -VMName "VMName" -DestinationStoragePath "E:\NewPath"

# Move specific VHD
Move-VMStorage -VMName "VMName" `
    -VHDs @(@{SourceFilePath="C:\old.vhdx"; DestinationFilePath="D:\new.vhdx"})
```

### Export/Import

```powershell
# Export VM
Export-VM -Name "VMName" -Path "D:\Exports"

# Import VM
Import-VM -Path "D:\Exports\VMName\*.vmcx"

# Import with copy
Import-VM -Path "D:\Exports\VMName\*.vmcx" -Copy -GenerateNewId
```

---

## Monitoring

### VM Status

```powershell
# List all VMs
Get-VM

# Detailed info
Get-VM -Name "VMName" | Format-List *

# Specific properties
Get-VM | Select Name, State, CPUUsage, MemoryAssigned, Uptime
```

### Resource Metrics

```powershell
# Enable resource metering
Enable-VMResourceMetering -VMName "VMName"

# Get metrics
Measure-VM -VMName "VMName"

# Reset metrics
Reset-VMResourceMetering -VMName "VMName"

# Disable metering
Disable-VMResourceMetering -VMName "VMName"
```

---

## Common Patterns

### Bulk Operations

```powershell
# Stop all running VMs
Get-VM | Where-Object {$_.State -eq "Running"} | Stop-VM

# Start all VMs
Get-VM | Start-VM

# Checkpoint all VMs
Get-VM | Checkpoint-VM -SnapshotName "Bulk Checkpoint"
```

### VM Creation Script

```powershell
$vmConfig = @{
    Name = "NewVM"
    MemoryStartupBytes = 4GB
    Generation = 2
    NewVHDPath = "C:\VMs\NewVM.vhdx"
    NewVHDSizeBytes = 60GB
    SwitchName = "vSwitch"
    Path = "C:\VMs"
}

New-VM @vmConfig
Set-VMProcessor -VMName "NewVM" -Count 2
Set-VMMemory -VMName "NewVM" -DynamicMemoryEnabled $true
Add-VMDvdDrive -VMName "NewVM" -Path "C:\ISOs\Windows.iso"
```

### Health Check Script

```powershell
Get-VM | ForEach-Object {
    [PSCustomObject]@{
        Name = $_.Name
        State = $_.State
        CPUUsage = $_.CPUUsage
        MemoryMB = [math]::Round($_.MemoryAssigned / 1MB)
        Uptime = $_.Uptime
        ReplicationHealth = (Get-VMReplication -VMName $_.Name -ErrorAction SilentlyContinue).Health
    }
} | Format-Table -AutoSize
```
