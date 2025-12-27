# Hyper-V Deployment Reference

## Table of Contents
- [Installation](#installation)
- [Virtual Machine Creation](#virtual-machine-creation)
- [VM Export and Import](#vm-export-and-import)
- [VM Version Upgrades](#vm-version-upgrades)
- [Generation 1 vs Generation 2](#generation-1-vs-generation-2)

---

## Installation

### Requirements

**Windows Server:**
- Compatible hardware meeting Hyper-V specifications
- No conflicting virtualization software (VMware Workstation, VirtualBox)

**Windows 10/11:**
- Pro or Enterprise edition (Home unsupported)
- 64-bit processor with SLAT (Second Level Address Translation)
- CPU with VM Monitor Mode support (Intel VT-c)
- Minimum 4GB RAM

### Installation Commands

**Windows Server (PowerShell):**
```powershell
# Local installation
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart

# Remote installation
Install-WindowsFeature -Name Hyper-V -ComputerName <computer_name> -IncludeManagementTools -Restart
```

**Windows Client (PowerShell):**
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

**Windows Client (DISM):**
```powershell
DISM /Online /Enable-Feature /All /FeatureName:Microsoft-Hyper-V
```

---

## Virtual Machine Creation

### Hyper-V Manager Method

1. Open Hyper-V Manager
2. Select server → **New** → **Virtual Machine**
3. Configure wizard screens:
   - **Name and Location**: VM name and storage path
   - **Generation**: Select Gen 1 or Gen 2 (Gen 2 recommended)
   - **Memory**: 32 MB minimum, enable Dynamic Memory if needed
   - **Networking**: Select virtual switch or skip
   - **Virtual Hard Disk**: Create new, use existing, or attach later
   - **Installation Options**: ISO, floppy, network, or defer

### PowerShell Method

**List available switches:**
```powershell
Get-VMSwitch * | Format-Table Name
```

**Create VM with existing VHD:**
```powershell
New-VM -Name "VMName" `
    -MemoryStartupBytes 4GB `
    -BootDevice VHD `
    -VHDPath "C:\VMs\disk.vhdx" `
    -Path "C:\VMs" `
    -Generation 2 `
    -Switch "vSwitch"
```

**Create VM with new VHD:**
```powershell
New-VM -Name "VMName" `
    -MemoryStartupBytes 4GB `
    -BootDevice VHD `
    -NewVHDPath "C:\VMs\disk.vhdx" `
    -NewVHDSizeBytes 60GB `
    -Generation 2 `
    -Switch "vSwitch"
```

**Start and connect:**
```powershell
Start-VM -Name "VMName"
VMConnect.exe localhost "VMName"
```

---

## VM Export and Import

### Export

**Hyper-V Manager:** Right-click VM → Export → Choose location

**PowerShell:**
```powershell
Export-VM -Name "VMName" -Path "D:\Exports"
```

VMs can be exported while running or stopped.

### Import

**Import Types:**

| Type | Files | VM ID | Multiple Imports |
|------|-------|-------|------------------|
| Register in-place | Original location | Same | No |
| Restore | Custom/default | Same | No |
| Copy | Custom location | New | Yes |

**PowerShell Examples:**

```powershell
# Register in place (use existing files)
Import-VM -Path 'C:\Export\VM\Virtual Machines\*.vmcx'

# Restore to custom location
Import-VM -Path 'C:\Export\VM\Virtual Machines\*.vmcx' `
    -Copy `
    -VhdDestinationPath 'D:\VMs\Disks' `
    -VirtualMachinePath 'D:\VMs'

# Copy with new ID (allows multiple imports)
Import-VM -Path 'C:\Export\VM\Virtual Machines\*.vmcx' `
    -Copy `
    -GenerateNewId
```

---

## VM Version Upgrades

### Check Versions

```powershell
# List all VMs with versions
Get-VM * | Format-Table Name, Version

# Check host supported versions
Get-VMHostSupportedVersion
```

### Upgrade Procedure

**Prerequisites:**
- VM must be shut down
- Cluster functional level upgraded (if clustered)
- Cannot downgrade after upgrade

**Hyper-V Manager:** Shut down VM → Action → Upgrade Configuration Version

**PowerShell:**
```powershell
Update-VMVersion -VMName "VMName"
```

### Version Compatibility

| Host Version | Supported Config Versions |
|--------------|---------------------------|
| Windows Server 2025 | 8.1 - 12.0 |
| Windows Server 2022 | 8.0 - 10.0 |
| Windows Server 2019 | 5.0 - 9.0 |
| Windows Server 2016 | 5.0 - 8.0 |

---

## Generation 1 vs Generation 2

### Generation 2 (Recommended)

- UEFI firmware with Secure Boot
- Boot from VHDX, virtual SCSI, or virtual network adapter
- Resize boot VHDX while running
- Supports shielded VMs
- Supports nested virtualization
- Required for Linux VMs with Secure Boot

### Generation 1 (Legacy)

- BIOS firmware
- Boot from virtual IDE
- Required for older OSes (Windows XP, Server 2003)
- Required for 32-bit guests
- Supports VHD format

### Selection PowerShell

```powershell
# Create Gen 2 VM
New-VM -Name "ModernVM" -Generation 2 -MemoryStartupBytes 4GB

# Create Gen 1 VM (legacy compatibility)
New-VM -Name "LegacyVM" -Generation 1 -MemoryStartupBytes 2GB
```
