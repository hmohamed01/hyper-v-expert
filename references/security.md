# Hyper-V Security Reference

## Table of Contents
- [Host Security](#host-security)
- [Virtual Machine Security](#virtual-machine-security)
- [Shielded VMs](#shielded-vms)
- [Secure Boot](#secure-boot)
- [Virtualization-Based Security](#virtualization-based-security)
- [Encryption](#encryption)

---

## Host Security

### Operating System Hardening

- Use Server Core or minimal installation
- Keep firmware, drivers, and patches current
- Avoid using host as general workstation
- Apply Windows Security Baselines

### Network Security

```powershell
# Dedicate NICs for management traffic
New-VMSwitch -Name "Management" -NetAdapterName "ManagementNIC" -AllowManagementOS $true

# Separate switch for VM traffic
New-VMSwitch -Name "VMTraffic" -NetAdapterName "DataNIC" -AllowManagementOS $false
```

### Access Control

```powershell
# View Hyper-V administrators
Get-LocalGroupMember -Group "Hyper-V Administrators"

# Add user to Hyper-V administrators (limited access)
Add-LocalGroupMember -Group "Hyper-V Administrators" -Member "DOMAIN\User"
```

### Storage Security

```powershell
# Enable BitLocker on VHD storage
Enable-BitLocker -MountPoint "D:" -EncryptionMethod XtsAes256 -UsedSpaceOnly

# Restrict VHD folder permissions
$acl = Get-Acl "D:\VMs"
$acl.SetAccessRuleProtection($true, $false)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule("Administrators","FullControl","ContainerInherit,ObjectInherit","None","Allow")
$acl.AddAccessRule($rule)
Set-Acl "D:\VMs" $acl
```

### Antivirus Exclusions

Configure these exclusions for antivirus:
- VM configuration files: `*.xml`, `*.vmcx`, `*.vmrs`
- Virtual hard disks: `*.vhd`, `*.vhdx`, `*.avhd`, `*.avhdx`
- Checkpoint files: `*.vsv`, `*.bin`
- Directories: Default VM paths, Hyper-V directories

---

## Virtual Machine Security

### Generation 2 Security Features

Gen 2 VMs support:
- UEFI Secure Boot
- Virtual TPM (vTPM)
- Shielded VMs
- Nested virtualization

### Configure VM Security Settings

```powershell
# Enable Secure Boot
Set-VMFirmware -VMName "VMName" -EnableSecureBoot On

# Set Secure Boot template
Set-VMFirmware -VMName "VMName" -SecureBootTemplate MicrosoftWindows

# For Linux VMs
Set-VMFirmware -VMName "LinuxVM" -SecureBootTemplate MicrosoftUEFICertificateAuthority

# Enable TPM
Enable-VMTPM -VMName "VMName"

# Enable Key Protector (for shielding)
Set-VMKeyProtector -VMName "VMName" -NewLocalKeyProtector
```

### Guest Configuration

```powershell
# Enable virtualization extensions (for nested Hyper-V)
Set-VMProcessor -VMName "VMName" -ExposeVirtualizationExtensions $true

# Disable for production (unless needed)
Set-VMProcessor -VMName "VMName" -ExposeVirtualizationExtensions $false
```

---

## Shielded VMs

Shielded VMs protect against compromised fabric administrators using:
- vTPM for BitLocker encryption
- VMGS (VM Guest State) encryption
- Attestation through Host Guardian Service

### Requirements

- Windows Server 2016+ Datacenter
- Host Guardian Service (HGS)
- Gen 2 VMs
- TPM 2.0 on hosts (for TPM attestation)

### Enable Shielding

```powershell
# Enable VM shielding (requires HGS)
Set-VMSecurityPolicy -VMName "VMName" -Shielded $true

# Check shielding status
Get-VMSecurityPolicy -VMName "VMName"
```

### Create Shielding Data

```powershell
# Export guardian metadata
Get-HgsGuardian -Name "MyGuardian" | Export-HgsGuardian -Path "C:\guardian.xml"

# Create shielding data file for VM deployment
New-ShieldingDataFile -ShieldingDataFilePath "C:\shieldingdata.pdk" -Owner (Get-HgsGuardian -Name "Owner") -Guardian (Import-HgsGuardian -Path "C:\guardian.xml")
```

---

## Secure Boot

### Enable Secure Boot

```powershell
# Check current status
Get-VMFirmware -VMName "VMName" | Select SecureBoot*

# Enable with Windows template
Set-VMFirmware -VMName "VMName" -EnableSecureBoot On -SecureBootTemplate MicrosoftWindows

# Enable for Linux
Set-VMFirmware -VMName "VMName" -EnableSecureBoot On -SecureBootTemplate MicrosoftUEFICertificateAuthority

# Disable (not recommended)
Set-VMFirmware -VMName "VMName" -EnableSecureBoot Off
```

### Secure Boot Templates

| Template | Use Case |
|----------|----------|
| MicrosoftWindows | Windows VMs |
| MicrosoftUEFICertificateAuthority | Linux, BSD, custom |
| OpenSourceShieldedVM | Shielded Linux VMs |

---

## Virtualization-Based Security

VBS uses hypervisor to create isolated memory regions.

### Enable VBS in Guest

```powershell
# Enable nested virtualization (required for VBS in guest)
Set-VMProcessor -VMName "VMName" -ExposeVirtualizationExtensions $true

# In guest, enable VBS via Group Policy or:
# Computer Configuration > Administrative Templates > System > Device Guard
```

### Enable Credential Guard in Guest

Requires:
- Gen 2 VM
- Nested virtualization enabled
- vTPM enabled
- UEFI with Secure Boot

```powershell
# Configure VM for Credential Guard
Set-VMProcessor -VMName "VMName" -ExposeVirtualizationExtensions $true
Enable-VMTPM -VMName "VMName"
Set-VMFirmware -VMName "VMName" -EnableSecureBoot On
```

---

## Encryption

### Encrypt VM Traffic

**Live Migration:**
```powershell
# Encrypt live migration traffic
Set-VMHost -VirtualMachineMigrationPerformanceOption SMB

# Use SMB encryption
Set-SmbServerConfiguration -EncryptData $true
```

**Storage Migration:**
```powershell
# Force SMB encryption
Set-SmbClientConfiguration -RequireSecuritySignature $true
Set-SmbServerConfiguration -RequireSecuritySignature $true -EncryptData $true
```

### Dump Encryption

Protect memory dumps from compromised VMs:

```powershell
# Enable dump encryption
Set-VMHost -EnableEnhancedSessionMode $true

# Configure per-VM
Set-VM -VMName "VMName" -EncryptStateAndVmMigrationTraffic Enable
```

### VHDX Encryption with BitLocker

```powershell
# Inside VM, enable BitLocker on OS drive
Enable-BitLocker -MountPoint "C:" -EncryptionMethod XtsAes256 -TPMProtector

# With vTPM, BitLocker keys are protected by the vTPM
```
