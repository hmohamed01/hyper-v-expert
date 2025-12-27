# Hyper-V Expert Skill for Claude Code

A comprehensive Microsoft Hyper-V skill that provides Claude Code with deep expertise in Windows Server virtualization.

## Features

- **Deployment** - Install Hyper-V, create VMs, export/import, version upgrades
- **Management** - Live migration, integration services, CPU groups, scheduler types
- **Networking** - Virtual switches, VLANs, NAT, NIC teaming, QoS
- **Storage** - VHD/VHDX management, checkpoints, Fibre Channel, persistent memory
- **Security** - Shielded VMs, Secure Boot, vTPM, encryption
- **GPU Acceleration** - Discrete Device Assignment (DDA), GPU Partitioning
- **Replication** - Hyper-V Replica configuration, failover operations
- **PowerShell** - Comprehensive cmdlet reference and automation patterns

## Installation

```bash
unzip hyper-v-expert.skill -d ~/.claude/skills/hyper-v-expert
```

### Verify Installation

After installation, your skills directory should contain:

```
~/.claude/skills/
└── hyper-v-expert/
    ├── SKILL.md
    └── references/
        ├── deployment.md
        ├── gpu-acceleration.md
        ├── management.md
        ├── networking.md
        ├── powershell.md
        ├── replication.md
        ├── security.md
        └── storage.md
```

## Usage

Once installed, Claude Code will automatically use this skill when you ask about Hyper-V topics. Examples:

- "How do I install Hyper-V on Windows Server?"
- "Create a PowerShell script to set up a new VM with 8GB RAM"
- "Configure live migration between two hosts"
- "Set up Hyper-V Replica for disaster recovery"
- "What's the difference between DDA and GPU partitioning?"

## Requirements

- Claude Code CLI
- The skill provides guidance for:
  - Windows Server 2016, 2019, 2022, 2025
  - Windows 10/11 Pro or Enterprise (for client Hyper-V)

## Documentation Sources

This skill was built using official Microsoft documentation:
- [Windows Server Hyper-V Documentation](https://github.com/MicrosoftDocs/windowsserverdocs/tree/main/WindowsServerDocs/virtualization/hyper-v)
- [Hyper-V PowerShell Module](https://learn.microsoft.com/en-us/powershell/module/hyper-v/)

## License

MIT
