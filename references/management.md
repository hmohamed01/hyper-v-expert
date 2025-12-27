# Hyper-V Management Reference

## Table of Contents
- [Live Migration](#live-migration)
- [Integration Services](#integration-services)
- [CPU Groups](#cpu-groups)
- [Scheduler Types](#scheduler-types)
- [Remote Management](#remote-management)

---

## Live Migration

Live migration moves running VMs between hosts without downtime. Available without Failover Clustering since Windows Server 2016.

### Prerequisites

- Both hosts running same or compatible Windows Server versions
- Shared storage or SMB storage accessible to both hosts
- Same virtual switch names on both hosts
- Adequate network bandwidth

### Configure Hosts for Live Migration

**Step 1: Enable Live Migration (PowerShell)**
```powershell
# Enable on both source and destination hosts
Enable-VMMigration

# Configure migration network
Set-VMMigrationNetwork 192.168.1.0/24

# Set authentication (Kerberos recommended)
Set-VMHost -VirtualMachineMigrationAuthenticationType Kerberos
```

**Hyper-V Manager Method:**
1. Open Hyper-V Settings → Live Migrations
2. Check "Enable incoming and outgoing live migrations"
3. Set simultaneous migrations (default: 2)
4. Configure Advanced Features: authentication and performance options

### Constrained Delegation (Kerberos)

For Kerberos authentication, configure delegation in Active Directory:

1. Open Active Directory Users and Computers
2. Locate source server → Properties → Delegation tab
3. Select "Trust this computer for delegation to specified services only"
4. Select "Use any authentication protocol"
5. Add destination server with services:
   - `cifs` (storage migration)
   - `Microsoft Virtual System Migration Service` (VM migration)
6. Repeat for destination server pointing to source

### Perform Live Migration

**PowerShell:**
```powershell
# Basic live migration
Move-VM -Name "VMName" -DestinationHost "DestHost"

# With storage migration
Move-VM -Name "VMName" -DestinationHost "DestHost" `
    -DestinationStoragePath "D:\VMs"

# Move storage only (no host change)
Move-VMStorage -VMName "VMName" -DestinationStoragePath "E:\NewLocation"
```

**Hyper-V Manager:** Right-click VM → Move → select destination

---

## Integration Services

Integration services enhance VM-host communication and performance.

### Available Services

| Service | Description | Default |
|---------|-------------|---------|
| Operating System Shutdown | Graceful shutdown from host | Enabled |
| Time Synchronization | Sync guest time with host | Enabled |
| Data Exchange (KVP) | Registry key exchange | Enabled |
| Heartbeat | Monitor guest health | Enabled |
| Backup (VSS) | Application-consistent backups | Enabled |
| Guest Service Interface | File copy to guest | Disabled |

### Management Commands

```powershell
# List integration services status
Get-VMIntegrationService -VMName "VMName"

# Enable a service
Enable-VMIntegrationService -VMName "VMName" -Name "Guest Service Interface"

# Disable a service
Disable-VMIntegrationService -VMName "VMName" -Name "Time Synchronization"
```

### Linux Integration Services

Linux guests use kernel modules:
- `hv_vmbus` - VMBus driver
- `hv_storvsc` - Storage driver
- `hv_netvsc` - Network driver
- `hv_utils` - Utility driver

Daemons: `hv_vss_daemon`, `hv_kvp_daemon`, `hv_fcopy_daemon`

---

## CPU Groups

CPU groups allocate and restrict host CPU resources across VMs. Introduced in Windows Server 2016.

### Concepts

- **Group Cap**: Maximum CPU allocation for all VMs in a group
- **VM Cap**: Maximum CPU per individual VM within a group
- **LP Affinity**: Bind group to specific logical processors

### Formula

`G = n × C`

Where:
- G = Host logical processors allocated
- n = Total LPs in the group
- C = Maximum allocation percentage

Example: 4 LPs with 50% cap = 2 LP equivalents of CPU time

### Management

CPU groups are managed via **CpuGroups.exe** utility (available from Microsoft Download Center).

> **Note:** PowerShell, WMI, and Hyper-V Manager do not support CPU group management. Only the Host Compute Service (HCS) interface via CpuGroups.exe is supported.

---

## Scheduler Types

The hypervisor scheduler controls how virtual processors are scheduled on physical CPUs.

### Types

| Scheduler | Description | Default Since |
|-----------|-------------|---------------|
| Classic | Fair-share round-robin, VMs can share cores | Pre-2019 |
| Core | VMs isolated to physical cores, SMT-aware | Server 2019 |
| Root | Delegates to root partition OS | Client only |

### Security Considerations

- **Classic**: VMs from different security contexts may share physical cores (SMT threads), enabling side-channel attacks
- **Core**: Strong isolation - VMs never share physical cores with other VMs

### Configuration

```powershell
# Set scheduler type (requires restart)
bcdedit /set hypervisorschedulertype Core

# Check current scheduler
Get-WinEvent -FilterHashTable @{
    ProviderName="Microsoft-Windows-Hyper-V-Hypervisor"
    ID=2
} -MaxEvents 1
```

Event ID 2 values: 1 (Classic/no SMT), 2 (Classic), 3 (Core), 4 (Root)

### VM SMT Configuration

```powershell
# Set VM SMT threads per core
Set-VMProcessor -VMName "VMName" -HwThreadCountPerCore 2

# Inherit host topology (Server 2019+)
Set-VMProcessor -VMName "VMName" -HwThreadCountPerCore 0

# Disable SMT
Set-VMProcessor -VMName "VMName" -HwThreadCountPerCore 1

# Check current setting
(Get-VMProcessor -VMName "VMName").HwThreadCountPerCore
```

### Recommendations

1. Use Core scheduler on Windows Server 2016+
2. Configure VMs to match host SMT topology
3. Monitor Event ID 3498 for suboptimal configurations

---

## Remote Management

### Enable Remote Management

```powershell
# Enable on remote host
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "ManagementHost"
```

### Connect Remotely

**PowerShell:**
```powershell
# Enter remote session
Enter-PSSession -ComputerName "HyperVHost" -Credential (Get-Credential)

# Run commands remotely
Invoke-Command -ComputerName "HyperVHost" -ScriptBlock {
    Get-VM | Select Name, State
}
```

**Hyper-V Manager:**
1. Right-click Hyper-V Manager → Connect to Server
2. Enter server name and credentials

### Firewall Rules

```powershell
# Enable Hyper-V management firewall rules
Enable-NetFirewallRule -DisplayGroup "Hyper-V Management Clients"
Enable-NetFirewallRule -DisplayGroup "Remote Volume Management"
```
