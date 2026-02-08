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

| Command              | Description                                                |
| -------------------- | ---------------------------------------------------------- |
| `/vcpu-ratio`        | vCPU:pCPU ratios, CPU oversubscription, scheduler behavior |
| `/memory-mgmt`       | Memory reclamation hierarchy, ballooning, TPS, compression |
| `/storage-design`    | VMFS vs NFS vs vSAN, VMDK types, multipathing              |
| `/drs-rules`         | DRS configuration, affinity rules, resource pools          |
| `/ha-design`         | HA admission control, FT, isolation response               |
| `/capacity-plan`     | Sizing methodology, ratios, formulas                       |
| `/perf-troubleshoot` | esxtop metrics, thresholds, diagnostic workflows           |

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

### Compute

| File                            | Contents                                                                   |
| ------------------------------- | -------------------------------------------------------------------------- |
| `references/cpu-scheduler.md`   | VMkernel scheduler internals, co-scheduling, HT, latency-sensitive tuning  |
| `references/memory-sizing.md`   | TPS mechanics, balloon driver, compression, swap, NUMA memory, large pages |
| `references/vm-reservations.md` | CPU/memory reservations, shares, limits, HA slot size interaction          |
| `references/numa-vm-design.md`  | NUMA-aware VM sizing, vNUMA, wide VMs, hot-add impact, CPS alignment       |

### Availability

| File                                 | Contents                                                            |
| ------------------------------------ | ------------------------------------------------------------------- |
| `references/ha-patterns.md`          | Slot calculation, isolation detection, FT requirements              |
| `references/ha-admission-control.md` | Host Failures vs Percentage policy, slot inflation, troubleshooting |
| `references/drs-tuning.md`           | Migration thresholds, share calculations, affinity patterns         |

### Storage

| File                              | Contents                                                                     |
| --------------------------------- | ---------------------------------------------------------------------------- |
| `references/storage-patterns.md`  | Datastore selection, vSAN policies, multipathing PSPs                        |
| `references/vsan-architecture.md` | OSA vs ESA, fault domains, storage policies, object model, rebuild mechanics |

### Networking

| File                             | Contents                                                                          |
| -------------------------------- | --------------------------------------------------------------------------------- |
| `references/nsx-segmentation.md` | Overlay/underlay, tier-0/tier-1 topology, DFW, microsegmentation, security groups |

### Security

| File                              | Contents                                                                         |
| --------------------------------- | -------------------------------------------------------------------------------- |
| `references/vsphere-hardening.md` | Lockdown mode, RBAC, certificates, VM encryption, STIG/CIS baselines, zero trust |

### Disaster Recovery

| File                          | Contents                                                                        |
| ----------------------------- | ------------------------------------------------------------------------------- |
| `references/backup-and-dr.md` | RPO/RTO tiering, VADP backups, replication patterns, SRM, ransomware resilience |

### Containers

| File                                  | Contents                                                                              |
| ------------------------------------- | ------------------------------------------------------------------------------------- |
| `references/kubernetes-on-vsphere.md` | Tanzu Supervisor, TKG, vSphere Pods, CSI storage, Antrea CNI, workload cluster design |

### Design

| File                                    | Contents                                                                     |
| --------------------------------------- | ---------------------------------------------------------------------------- |
| `references/cluster-design-patterns.md` | Management/compute/edge clusters, sizing, resource pools, stretched clusters |

### Cloud and Modern Workloads

| File                                        | Contents                                                                  |
| ------------------------------------------- | ------------------------------------------------------------------------- |
| `references/on-prem-vs-cloud.md`            | Decision framework for on-prem vs cloud-native workload placement         |
| `references/cloud-platform-architecture.md` | VCF and AWS architectural comparison, layered model, conceptual parallels |
| `references/ai-workload-patterns.md`        | GPU infrastructure, model serving, API gateway patterns for AI workloads  |
| `references/hybrid-cloud-patterns.md`       | VMC on AWS, AVS, GCVE, HCX migration, connectivity, cost considerations   |

### Operations

| File                                         | Contents                                                                       |
| -------------------------------------------- | ------------------------------------------------------------------------------ |
| `references/troubleshooting.md`              | Complete esxtop interpretation, diagnostic workflows                           |
| `references/lifecycle-management.md`         | vLCM images/baselines, firmware management, upgrade sequencing, patch strategy |
| `references/capacity-planning.md`            | Usable capacity formulas, right-sizing, growth modeling, procurement triggers  |
| `references/monitoring-and-observability.md` | esxtop field reference, Aria Operations dashboards, alerting strategy, syslog  |

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

- Explain _why_ settings work the way they do, not just _what_ to configure
- Include practical thresholds and validation metrics
- Reference enterprise best practices
- Connect concepts to capacity planning implications

## License

MIT
