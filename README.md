# vSphere Architect Skill

A Claude skill for VMware vSphere architecture, design, and troubleshooting. Provides expert-level guidance on capacity planning, resource management, performance tuning, and infrastructure design for enterprise virtualization environments.

## Installation

### Claude.ai / Claude Desktop

1. Download `vsphere-architect.skill`
2. Go to Settings → Skills
3. Upload the skill file

### Claude Code

Copy the skill folder to your skills directory:

```bash
cp -r vsphere-architect ~/.claude/skills/
```

Or add to a project's `.claude/skills/` directory.

### Slash Commands (Claude Code)

For tab-completable slash commands, copy the command files:

```bash
cp commands/*.md ~/.claude/commands/
```

## Commands

| Command | Description |
|---------|-------------|
| `/vcpu-ratio` | vCPU:pCPU ratios, CPU oversubscription, scheduler behavior |
| `/memory-mgmt` | Memory reclamation hierarchy, ballooning, TPS, compression |
| `/storage-design` | VMFS vs NFS vs vSAN, VMDK types, multipathing |
| `/drs-rules` | DRS configuration, affinity rules, resource pools |
| `/ha-design` | HA admission control, FT, isolation response |
| `/capacity-plan` | Sizing methodology, ratios, formulas |
| `/perf-troubleshoot` | esxtop metrics, thresholds, diagnostic workflows |

## Usage Examples

```
/vcpu-ratio explain how 4:1 affects scheduling for database workloads

/memory-mgmt when does ballooning kick in vs compression?

/storage-design compare vSAN vs NFS for VDI with 500 desktops

/drs-rules configure anti-affinity for a 3-node SQL Always On cluster

/ha-design admission control for N+1 with 4 hosts and mixed VM sizes

/capacity-plan size a cluster for 200 VMs: 60% general, 30% database, 10% VDI

/perf-troubleshoot VM showing 8% CPU ready but host only at 60% utilization
```

## Reference Files

The skill includes detailed reference documentation:

| File | Contents |
|------|----------|
| `references/cpu-scheduler.md` | VMkernel scheduler internals, NUMA, HT, latency-sensitive tuning |
| `references/memory-sizing.md` | TPS mechanics, balloon driver, NUMA memory, large pages |
| `references/storage-patterns.md` | Datastore selection, vSAN policies, multipathing PSPs |
| `references/drs-tuning.md` | Migration thresholds, share calculations, affinity patterns |
| `references/ha-patterns.md` | Slot calculation, isolation detection, FT requirements |
| `references/troubleshooting.md` | Complete esxtop interpretation, diagnostic workflows |

## Topics Covered

### CPU
- vCPU:pCPU ratio calculation and recommendations
- CPU Ready vs Co-Stop interpretation
- NUMA scheduling and vNUMA
- Hyperthreading considerations
- Latency-sensitive workload tuning
- Resource pool shares and limits

### Memory
- Reclamation hierarchy (TPS → Balloon → Compress → Swap)
- Memory state thresholds
- Overcommitment guidelines
- NUMA memory locality
- Large page support

### Storage
- VMFS 6 vs NFS v3/v4.1 vs vSAN vs vVols
- VMDK provisioning types
- vSAN FTT policies and stripe width
- Multipathing (Round Robin, Fixed, MRU)
- Storage I/O Control (SIOC)
- Queue depth tuning

### DRS
- Automation levels and migration thresholds
- Affinity/anti-affinity rules (VM-VM, VM-Host)
- Required vs Preferred rules
- Resource pool hierarchy
- DRS groups and VM-Host rules

### High Availability
- Admission control policies
- Slot calculation
- Network isolation handling
- APD/PDL response
- Fault Tolerance requirements
- N+1 and N+2 design patterns

### Performance
- esxtop views and key metrics
- Critical thresholds
- Troubleshooting workflows
- Log analysis
- vm-support bundle collection

## Response Style

The skill provides architectural explanations that:
- Explain *why* settings work the way they do, not just *what* to configure
- Include practical thresholds and validation metrics
- Reference enterprise best practices
- Connect concepts to capacity planning implications

## License

MIT
