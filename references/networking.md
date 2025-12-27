# Hyper-V Networking Reference

## Table of Contents
- [Virtual Switches](#virtual-switches)
- [VLANs](#vlans)
- [NAT Networking](#nat-networking)
- [Network QoS](#network-qos)
- [NIC Teaming](#nic-teaming)
- [Extended Port ACLs](#extended-port-acls)

---

## Virtual Switches

### Switch Types

| Type | VM Access | Host Access | External Network |
|------|-----------|-------------|------------------|
| External | Yes | Optional | Yes |
| Internal | Yes | Yes | No |
| Private | Yes | No | No |

### Create Virtual Switches

**External Switch (with host sharing):**
```powershell
# List available adapters
Get-NetAdapter | Where Status -eq "Up"

# Create external switch
New-VMSwitch -Name "ExternalSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true
```

**Internal Switch:**
```powershell
New-VMSwitch -Name "InternalSwitch" -SwitchType Internal
```

**Private Switch:**
```powershell
New-VMSwitch -Name "PrivateSwitch" -SwitchType Private
```

### Manage Switches

```powershell
# List all switches
Get-VMSwitch

# Get switch details
Get-VMSwitch -Name "SwitchName" | Format-List *

# Remove switch
Remove-VMSwitch -Name "SwitchName"

# Modify management OS access
Set-VMSwitch -Name "ExternalSwitch" -AllowManagementOS $false
```

### Connect VMs to Switches

```powershell
# Connect VM adapter to switch
Connect-VMNetworkAdapter -VMName "VMName" -SwitchName "SwitchName"

# Add new adapter and connect
Add-VMNetworkAdapter -VMName "VMName" -SwitchName "SwitchName" -Name "Adapter2"

# Disconnect adapter
Disconnect-VMNetworkAdapter -VMName "VMName"
```

---

## VLANs

### Prerequisites

- Physical NIC and driver supporting 802.1q VLAN tagging
- Physical switch with VLAN support

### Configure VLAN on Management OS

```powershell
# Set VLAN for management OS adapter
Set-VMNetworkAdapterVlan -ManagementOS -VMNetworkAdapterName "ExternalSwitch" `
    -Access -VlanId 100
```

### Configure VLAN on VM Adapter

```powershell
# Access mode (single VLAN)
Set-VMNetworkAdapterVlan -VMName "VMName" -Access -VlanId 100

# Trunk mode (multiple VLANs)
Set-VMNetworkAdapterVlan -VMName "VMName" -Trunk `
    -NativeVlanId 1 `
    -AllowedVlanIdList "100,200,300"

# Disable VLAN tagging
Set-VMNetworkAdapterVlan -VMName "VMName" -Untagged
```

### View VLAN Configuration

```powershell
# Check VM VLAN settings
Get-VMNetworkAdapterVlan -VMName "VMName"

# Check management OS VLAN
Get-VMNetworkAdapterVlan -ManagementOS
```

---

## NAT Networking

NAT allows VMs on internal networks to access external networks through the host.

### Create NAT Network

```powershell
# 1. Create internal switch
New-VMSwitch -Name "NATSwitch" -SwitchType Internal

# 2. Get interface index
$ifIndex = (Get-NetAdapter -Name "vEthernet (NATSwitch)").ifIndex

# 3. Configure host IP
New-NetIPAddress -IPAddress 192.168.100.1 -PrefixLength 24 -InterfaceIndex $ifIndex

# 4. Create NAT
New-NetNat -Name "VMNat" -InternalIPInterfaceAddressPrefix "192.168.100.0/24"
```

### Configure VM Networking

Inside VM, configure static IP:
- IP: 192.168.100.x (e.g., 192.168.100.10)
- Subnet: 255.255.255.0
- Gateway: 192.168.100.1
- DNS: External DNS server

### Manage NAT

```powershell
# List NAT configurations
Get-NetNat

# Remove NAT
Remove-NetNat -Name "VMNat"

# Port forwarding
Add-NetNatStaticMapping -NatName "VMNat" `
    -Protocol TCP `
    -ExternalIPAddress 0.0.0.0 `
    -ExternalPort 8080 `
    -InternalIPAddress 192.168.100.10 `
    -InternalPort 80
```

---

## Network QoS

### Bandwidth Management

```powershell
# Set minimum bandwidth (absolute)
Set-VMNetworkAdapter -VMName "VMName" -MinimumBandwidthAbsolute 100Mbps

# Set minimum bandwidth (weight-based, 1-100)
Set-VMNetworkAdapter -VMName "VMName" -MinimumBandwidthWeight 50

# Set maximum bandwidth
Set-VMNetworkAdapter -VMName "VMName" -MaximumBandwidth 1Gbps
```

### Enable QoS on Switch

```powershell
# Enable bandwidth management mode
Set-VMSwitch -Name "SwitchName" -DefaultFlowMinimumBandwidthAbsolute 100Mbps

# Or weight-based
Set-VMSwitch -Name "SwitchName" -DefaultFlowMinimumBandwidthWeight 10
```

---

## NIC Teaming

### Create Team for Hyper-V

```powershell
# Create NIC team
New-NetLbfoTeam -Name "HVTeam" -TeamMembers "NIC1","NIC2" -TeamingMode SwitchIndependent -LoadBalancingAlgorithm Dynamic

# Create switch on team
New-VMSwitch -Name "TeamSwitch" -NetAdapterName "HVTeam" -AllowManagementOS $true
```

### SET (Switch Embedded Teaming)

Preferred for Windows Server 2016+:

```powershell
# Create switch with embedded teaming
New-VMSwitch -Name "SETSwitch" -NetAdapterName "NIC1","NIC2" -EnableEmbeddedTeaming $true

# Add adapter to SET
Add-VMSwitchTeamMember -VMSwitchName "SETSwitch" -NetAdapterName "NIC3"

# View team members
Get-VMSwitchTeam -Name "SETSwitch"
```

---

## Extended Port ACLs

Control network traffic at the virtual switch port level.

### Create ACLs

```powershell
# Block outbound traffic to specific IP
Add-VMNetworkAdapterExtendedAcl -VMName "VMName" `
    -Action Deny `
    -Direction Outbound `
    -RemoteIPAddress 10.0.0.100 `
    -Weight 10

# Allow specific port
Add-VMNetworkAdapterExtendedAcl -VMName "VMName" `
    -Action Allow `
    -Direction Inbound `
    -LocalPort 443 `
    -Protocol TCP `
    -Weight 20

# Block all other inbound (lower weight = lower priority)
Add-VMNetworkAdapterExtendedAcl -VMName "VMName" `
    -Action Deny `
    -Direction Inbound `
    -Weight 1
```

### Manage ACLs

```powershell
# View ACLs
Get-VMNetworkAdapterExtendedAcl -VMName "VMName"

# Remove specific ACL
Remove-VMNetworkAdapterExtendedAcl -VMName "VMName" -Direction Inbound -Weight 10

# Remove all ACLs
Get-VMNetworkAdapterExtendedAcl -VMName "VMName" | Remove-VMNetworkAdapterExtendedAcl
```

### MAC Address Spoofing

```powershell
# Enable (required for nested virtualization, NLB)
Set-VMNetworkAdapter -VMName "VMName" -MacAddressSpoofing On

# Disable
Set-VMNetworkAdapter -VMName "VMName" -MacAddressSpoofing Off
```

### DHCP Guard and Router Guard

```powershell
# Prevent VM from acting as DHCP server
Set-VMNetworkAdapter -VMName "VMName" -DhcpGuard On

# Prevent VM from acting as router
Set-VMNetworkAdapter -VMName "VMName" -RouterGuard On
```
