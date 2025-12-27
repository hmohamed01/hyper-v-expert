# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This repository is the source for the **hyper-v-expert** skill for Claude Code, providing comprehensive Microsoft Hyper-V virtualization expertise.

## Structure

```
├── SKILL.md              # Main skill definition (triggers, workflows, decision trees)
├── references/           # Detailed reference documentation
│   ├── deployment.md     # Installation, VM creation, upgrades
│   ├── management.md     # Live migration, integration services, CPU groups
│   ├── networking.md     # Virtual switches, VLANs, NAT, QoS
│   ├── storage.md        # VHD/VHDX, checkpoints, Fibre Channel
│   ├── security.md       # Shielded VMs, Secure Boot, encryption
│   ├── gpu-acceleration.md   # DDA, GPU partitioning
│   ├── replication.md    # Hyper-V Replica, failover
│   └── powershell.md     # Cmdlets and automation patterns
├── hyper-v-expert.skill  # Packaged skill for distribution
└── README.md             # Installation instructions for users
```

## Packaging

After modifying the skill, repackage for distribution:

```bash
zip -r hyper-v-expert.skill SKILL.md references -x "*.DS_Store"
```

## Installation (for testing)

To install/update in Claude Code's skills directory:

```bash
cp -r SKILL.md references ~/.claude/skills/hyper-v-expert/
```

## Documentation Source

Content is based on official Microsoft documentation:
- https://github.com/MicrosoftDocs/windowsserverdocs/tree/main/WindowsServerDocs/virtualization/hyper-v
