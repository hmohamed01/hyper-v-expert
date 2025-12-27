# Hyper-V GPU Acceleration Reference

## Table of Contents
- [Overview](#overview)
- [Discrete Device Assignment (DDA)](#discrete-device-assignment-dda)
- [GPU Partitioning (GPU-P)](#gpu-partitioning-gpu-p)
- [RemoteFX vGPU (Deprecated)](#remotefx-vgpu-deprecated)

---

## Overview

### When to Use GPU Acceleration

| Workload | Description |
|----------|-------------|
| VDI/Remoting | CAD, simulation, gaming, 3D rendering |
| Rendering/Encoding | Video encoding, graphics rendering |
| HPC/ML | Machine learning, data-parallel computation |

### Technology Comparison

| Feature | DDA | GPU-P |
|---------|-----|-------|
| Isolation | Full device | Hardware SR-IOV |
| VMs per GPU | 1 | Multiple |
| Driver | Native | Native |
| Performance | Full | Near-native |
| Availability | Server 2016+ | Server 2025+ |

---

## Discrete Device Assignment (DDA)

DDA passes entire PCIe devices (GPUs, NVMe) directly to a VM.

### Requirements

- Windows Server 2016+
- Gen 2 VM
- Processors supporting IOMMU/VT-d
- Compatible GPU with IOMMU support
- 64-bit guest OS

### Step 1: Configure VM

```powershell
# Set automatic stop action (required)
Set-VM -Name "VMName" -AutomaticStopAction TurnOff

# Enable write-combining
Set-VM -Name "VMName" -GuestControlledCacheTypes $true

# Configure 32-bit MMIO space (adjust based on GPU)
Set-VM -Name "VMName" -LowMemoryMappedIoSpace 3Gb

# Configure high MMIO space (for >4GB VRAM)
Set-VM -Name "VMName" -HighMemoryMappedIoSpace 33280Mb
```

### Step 2: Find GPU Location Path

```powershell
# List PCIe devices
Get-PnpDevice -PresentOnly | Where-Object {$_.Class -eq "Display"}

# Get location path for GPU
$locationPath = (Get-PnpDeviceProperty -InstanceId "PCI\VEN_10DE*" -KeyName DEVPKEY_Device_LocationPaths).Data[0]
```

### Step 3: Dismount GPU from Host

```powershell
# Disable device in Device Manager first, then:
Dismount-VMHostAssignableDevice -LocationPath $locationPath -Force
```

### Step 4: Assign to VM

```powershell
# Assign GPU to VM
Add-VMAssignableDevice -VMName "VMName" -LocationPath $locationPath
```

### Step 5: Install Drivers

Start VM and install GPU vendor drivers inside the guest.

### Remove GPU from VM

```powershell
# Stop VM first
Stop-VM -Name "VMName"

# Remove from VM
Remove-VMAssignableDevice -VMName "VMName" -LocationPath $locationPath

# Remount to host
Mount-VMHostAssignableDevice -LocationPath $locationPath
```

### Verify Assignment

```powershell
# List assigned devices
Get-VMAssignableDevice -VMName "VMName"
```

---

## GPU Partitioning (GPU-P)

GPU-P shares a single GPU across multiple VMs using SR-IOV. Available in Windows Server 2025+.

### Requirements

- Windows Server 2025+
- Compatible GPU with SR-IOV support
- GPU driver supporting partitioning
- Gen 2 VM

### Check GPU-P Support

```powershell
# List partitionable GPUs
Get-VMHostPartitionableGpu

# Get partition count
(Get-VMHostPartitionableGpu).PartitionCount
```

### Assign GPU Partition

```powershell
# Add GPU partition to VM
Add-VMGpuPartitionAdapter -VMName "VMName"

# Configure partition resources (optional)
Set-VMGpuPartitionAdapter -VMName "VMName" `
    -MinPartitionVRAM 1GB `
    -MaxPartitionVRAM 4GB `
    -OptimalPartitionVRAM 2GB
```

### Copy GPU Driver to VM

GPU-P requires the host driver files to be copied to the guest:

```powershell
# Mount VM's VHD
$vhd = Mount-VHD -Path "C:\VMs\VMName.vhdx" -Passthru
$driveLetter = ($vhd | Get-Disk | Get-Partition | Where-Object {$_.Type -eq "Basic"}).DriveLetter

# Copy driver files
$hostDriverPath = "C:\Windows\System32\DriverStore\FileRepository\nvlddmkm*"
$guestDriverPath = "${driveLetter}:\Windows\System32\DriverStore\FileRepository\"
Copy-Item -Path $hostDriverPath -Destination $guestDriverPath -Recurse

# Dismount
Dismount-VHD -Path "C:\VMs\VMName.vhdx"
```

### Remove GPU Partition

```powershell
Remove-VMGpuPartitionAdapter -VMName "VMName"
```

### View GPU Partition Status

```powershell
# List VM GPU adapters
Get-VMGpuPartitionAdapter -VMName "VMName"

# Host GPU status
Get-VMHostPartitionableGpu | Format-List *
```

---

## RemoteFX vGPU (Deprecated)

> **Warning:** RemoteFX vGPU is deprecated and removed in Windows Server 2025. Use DDA or GPU-P instead.

RemoteFX was used for VDI scenarios but has security vulnerabilities.

### Migration Path

- For dedicated GPU per VM: Use DDA
- For shared GPU: Use GPU-P (Server 2025+)
- For older servers: Consider upgrading or using software rendering
