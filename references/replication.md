# Hyper-V Replica Reference

## Table of Contents
- [Overview](#overview)
- [Planning](#planning)
- [Configuration](#configuration)
- [Enable Replication](#enable-replication)
- [Failover Operations](#failover-operations)
- [Extended Replication](#extended-replication)

---

## Overview

Hyper-V Replica provides asynchronous VM replication for disaster recovery. Available on Windows Server 2012+ (not available on Windows client).

### Key Features

- Replication intervals: 30 seconds, 5 minutes, or 15 minutes
- Up to 24 recovery points (hourly)
- Application-consistent snapshots via VSS
- Works with or without Failover Clustering
- Extended replication to third site

---

## Planning

### Decision Points

| Factor | Consideration |
|--------|---------------|
| VHDs to replicate | Exclude page files, temp data |
| Replication interval | Based on RPO requirements |
| Recovery points | Up to 24 hourly points |
| Data churn | High churn = more bandwidth |
| Initial seeding | Network, media, or existing VM |

### Authentication Methods

| Method | Use Case | Requirements |
|--------|----------|--------------|
| Kerberos | Domain-joined hosts | AD, no encryption |
| Certificate | Non-domain, encryption needed | PKI infrastructure |

### Bandwidth Calculation

Estimate required bandwidth:
```
Bandwidth = (Data churn per interval) / (Replication interval)
```

Example: 10 GB daily churn, 15-minute interval
```
10 GB / 96 intervals = ~107 MB per interval
107 MB / 15 min = ~7 MB/min = ~1 Mbps
```

---

## Configuration

### Enable Replica Server Role

**Hyper-V Manager:**
1. Open Hyper-V Settings → Replication Configuration
2. Enable "Enable this computer as a Replica server"
3. Select authentication method
4. Configure allowed servers and storage location

**PowerShell:**
```powershell
# Enable as replica server (Kerberos)
Set-VMReplicationServer -ReplicationEnabled $true `
    -AllowedAuthenticationType Kerberos `
    -DefaultStorageLocation "D:\Replicas"

# Enable as replica server (Certificate)
Set-VMReplicationServer -ReplicationEnabled $true `
    -AllowedAuthenticationType Certificate `
    -CertificateThumbprint "THUMBPRINT" `
    -DefaultStorageLocation "D:\Replicas"
```

### Configure Firewall

```powershell
# HTTP (Kerberos) - port 80
Enable-NetFirewallRule -DisplayName "Hyper-V Replica HTTP Listener (TCP-In)"

# HTTPS (Certificate) - port 443
Enable-NetFirewallRule -DisplayName "Hyper-V Replica HTTPS Listener (TCP-In)"
```

### Trust Configuration

```powershell
# Allow specific primary server
Set-VMReplicationServer -ReplicationAllowedFromAnyServer $false
New-VMReplicationAuthorizationEntry -AllowedPrimaryServer "PrimaryHost.domain.com" `
    -ReplicaStorageLocation "D:\Replicas\PrimaryHost" `
    -TrustGroup "Default"

# Allow all servers in domain
Set-VMReplicationServer -ReplicationAllowedFromAnyServer $true
```

---

## Enable Replication

### Enable on Primary Server

**PowerShell:**
```powershell
# Basic replication (Kerberos)
Enable-VMReplication -VMName "VMName" `
    -ReplicaServerName "ReplicaHost.domain.com" `
    -ReplicaServerPort 80 `
    -AuthenticationType Kerberos

# With certificate authentication
Enable-VMReplication -VMName "VMName" `
    -ReplicaServerName "ReplicaHost.domain.com" `
    -ReplicaServerPort 443 `
    -AuthenticationType Certificate `
    -CertificateThumbprint "THUMBPRINT"

# With custom settings
Enable-VMReplication -VMName "VMName" `
    -ReplicaServerName "ReplicaHost.domain.com" `
    -ReplicaServerPort 443 `
    -AuthenticationType Certificate `
    -CertificateThumbprint "THUMBPRINT" `
    -ReplicationFrequencySec 300 `
    -RecoveryHistory 12 `
    -VSSSnapshotFrequencyHour 4 `
    -CompressionEnabled $true
```

### Enable on Replica Server (As Replica)

```powershell
# Configure VM as replica target
Enable-VMReplication -VMName "VMName" -AsReplica `
    -AllowedPrimaryServer "PrimaryHost.domain.com"
```

### Start Initial Replication

```powershell
# Immediate network transfer
Start-VMInitialReplication -VMName "VMName"

# Scheduled transfer
Start-VMInitialReplication -VMName "VMName" `
    -ScheduledStartTime "2024-01-15 22:00:00"

# From exported VM
Start-VMInitialReplication -VMName "VMName" `
    -UseExternalVMAsInitialCopy `
    -ExternalVMPath "\\FileServer\Exports\VMName"
```

---

## Failover Operations

### Test Failover

Creates test VM without disrupting replication:

```powershell
# Start test failover
Start-VMFailover -VMName "VMName" -AsTest

# Connect and verify...

# Stop test failover
Stop-VMFailover -VMName "VMName"
```

### Planned Failover

Graceful failover with no data loss:

```powershell
# On primary: prepare for failover
Start-VMFailover -VMName "VMName" -Prepare

# On replica: complete failover
Start-VMFailover -VMName "VMName"

# Start the VM on replica
Start-VM -VMName "VMName"

# Reverse replication direction
Set-VMReplication -VMName "VMName" -Reverse
```

### Unplanned Failover

When primary is unavailable:

```powershell
# On replica: force failover
Start-VMFailover -VMName "VMName"

# Start the VM
Start-VM -VMName "VMName"
```

### Complete/Cancel Failover

```powershell
# Complete failover (commit)
Complete-VMFailover -VMName "VMName"

# Cancel failover (revert)
Stop-VMFailover -VMName "VMName"
```

---

## Extended Replication

Replicate to a third site for additional protection.

### Enable Extended Replication

```powershell
# On replica server, extend to third site
Enable-VMReplication -VMName "VMName" `
    -ReplicaServerName "ExtendedHost.domain.com" `
    -ReplicaServerPort 443 `
    -AuthenticationType Certificate `
    -CertificateThumbprint "THUMBPRINT" `
    -ExtendedReplication
```

### Extended Replication Limitations

- Only 5-minute or 15-minute intervals
- No additional recovery points
- No application-consistent snapshots

---

## Monitoring and Management

### Check Replication Status

```powershell
# Get replication status
Get-VMReplication -VMName "VMName"

# Detailed health
Measure-VMReplication -VMName "VMName"

# All replicated VMs
Get-VMReplication | Format-Table VMName, State, Health, Mode
```

### Resynchronization

```powershell
# Manual resync (after errors)
Resume-VMReplication -VMName "VMName" -Resynchronize

# Schedule resync window
Resume-VMReplication -VMName "VMName" -Resynchronize `
    -ResynchronizeStartTime "22:00" `
    -ResynchronizeEndTime "06:00"
```

### Suspend/Resume Replication

```powershell
# Suspend
Suspend-VMReplication -VMName "VMName"

# Resume
Resume-VMReplication -VMName "VMName"
```

### Remove Replication

```powershell
# Remove replication relationship
Remove-VMReplication -VMName "VMName"
```
