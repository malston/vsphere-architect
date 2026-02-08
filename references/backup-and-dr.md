# Backup and Disaster Recovery Patterns

## Foundational Concepts

### RPO vs RTO

| Metric                             | Definition                                           | Question It Answers                    |
| ---------------------------------- | ---------------------------------------------------- | -------------------------------------- |
| **RPO** (Recovery Point Objective) | Maximum acceptable data loss measured in time        | "How much data can we afford to lose?" |
| **RTO** (Recovery Time Objective)  | Maximum acceptable downtime from failure to recovery | "How long can we be down?"             |

```
         RPO                          RTO
   <------------>                <------------>
   |             |               |             |
Last backup    Failure        Failure       Service
               occurs         occurs        restored
```

These two numbers drive every DR architecture decision. Everything else is implementation detail.

### RPO/RTO Tiers

| Tier     | RPO                 | RTO         | Typical Workloads                          | Cost      |
| -------- | ------------------- | ----------- | ------------------------------------------ | --------- |
| Platinum | Near-zero (seconds) | Minutes     | Financial transactions, healthcare records | Very high |
| Gold     | Minutes to 1 hour   | 1-4 hours   | ERP, CRM, production databases             | High      |
| Silver   | 4-24 hours          | 4-12 hours  | Internal apps, file services               | Moderate  |
| Bronze   | 24-72 hours         | 24-48 hours | Dev/test, archival                         | Low       |

**Assign tiers per application, not per cluster.** A single cluster can host platinum databases and bronze dev VMs -- they just get different protection policies.

## Backup Patterns

### Image-Level VM Backup

Snapshot the entire VM (disks, config, memory state) via the vSphere Storage APIs for Data Protection (VADP).

```
Backup Software → vCenter API → Create Snapshot → Read VMDK via NBD/HotAdd → Backup Target
                                                                               |
                                                     (optional) → Dedup/Compress → Object Storage
```

**How VADP works:**

1. Backup software triggers a VM snapshot via vCenter
2. Snapshot freezes the disk state (and optionally quiesces the guest via VMware Tools)
3. Backup reads the snapshot delta via Changed Block Tracking (CBT)
4. Snapshot is removed after backup completes

**Changed Block Tracking (CBT):** Tracks which disk blocks changed since last backup. Enables incremental backups that transfer only changed blocks instead of full VMDKs. Dramatically reduces backup windows and storage.

| Backup Type  | What It Captures                       | Duration          | Storage  |
| ------------ | -------------------------------------- | ----------------- | -------- |
| Full         | Entire VMDK                            | Hours (large VMs) | High     |
| Incremental  | Changed blocks since last backup (CBT) | Minutes           | Low      |
| Differential | Changed blocks since last full         | Moderate          | Moderate |

**Common tools:** Veeam, Commvault, Dell Avamar/Networker, Cohesity, NAKIVO

### Application-Consistent Backup

Image-level backup captures disk state, but the application may have uncommitted transactions in memory. Application consistency requires:

- **VMware Tools quiescing**: Triggers VSS (Windows) or filesystem sync (Linux) inside the guest before snapshot
- **Pre/post scripts**: Run application-specific commands (e.g., `ALTER DATABASE SET RECOVERY SIMPLE`, Oracle `BEGIN BACKUP`) before and after snapshot
- **Application-aware backup agents**: In-guest agents that coordinate with the application (SQL Server VSS writer, Oracle RMAN)

| Method                    | Consistency Level                       | Guest Agent Required                  |
| ------------------------- | --------------------------------------- | ------------------------------------- |
| Crash-consistent snapshot | Disk only (like pulling the power plug) | No                                    |
| Quiesced snapshot         | Filesystem consistent                   | VMware Tools                          |
| Application-consistent    | Transaction consistent                  | Application-specific agent or scripts |

**Rule of thumb**: Databases always need application-consistent backups. File servers and web servers are usually fine with quiesced snapshots.

### Backup Target Architecture

```
                    Primary Site
                         |
              +----------+----------+
              |                     |
         Local Backup          Backup Copy
         (fast restore)        (offsite protection)
              |                     |
         Dedup Appliance       Object Storage / Tape
         or Backup Repo        (S3, Azure Blob, AWS S3)
```

**3-2-1 Rule:**

- **3** copies of data (production + 2 backups)
- **2** different media types (disk + object storage or tape)
- **1** offsite copy (different location from production)

**Immutable backups**: Protection against ransomware. Write-once storage (S3 Object Lock, hardened Linux repos, air-gapped tape) where backups can't be encrypted or deleted by an attacker who compromises the backup infrastructure.

## Replication Patterns

### vSphere Replication

Built into vSphere (no additional license for basic use). Asynchronous replication at the VM level.

- **RPO**: Minimum 5 minutes (configured per VM)
- **Mechanism**: Tracks changed blocks in the ESXi kernel, replicates to target host/datastore
- **Does not require**: Shared storage, storage array replication, or SRM (but integrates with it)
- **Limitation**: Asynchronous only. Not suitable for near-zero RPO

```
Source ESXi Host                    Target ESXi Host
  |                                    |
  VM-A → [Changed blocks] ----WAN---→ Replica of VM-A
              (async, RPO ≥ 5min)
```

### Array-Based Replication

Storage array handles replication at the datastore/LUN level. The hypervisor isn't involved in replication -- it's transparent.

| Type         | RPO              | Mechanism                                            | Use Case                                   |
| ------------ | ---------------- | ---------------------------------------------------- | ------------------------------------------ |
| Synchronous  | Near-zero        | Write acknowledged only after both sites confirm     | Metro distance (<100km), latency sensitive |
| Asynchronous | Minutes to hours | Write acknowledged at source, replicated on schedule | Long distance, WAN bandwidth constrained   |

**Sync replication trade-off**: Every write incurs round-trip latency to the remote array. Works at metro distances (~5ms RTT). Unusable across continents.

**Vendors**: Dell SRDF/RecoverPoint, NetApp SnapMirror, Pure ActiveCluster, HPE Peer Persistence

### vSAN Stretched Clusters

vSAN can stretch a single cluster across two sites with a witness node at a third.

```
Site A (active)          Site B (active)          Witness Site
  ESXi hosts               ESXi hosts              Witness VM
  vSAN data                vSAN data               Metadata only
       |                       |                       |
       +---------- L2 stretched or L3 with BGP --------+
```

- **RPO**: Zero (synchronous writes to both sites)
- **RTO**: Minutes (HA restarts VMs at surviving site)
- **Requirements**: <5ms RTT between sites, 10Gbps+ bandwidth, witness at third site
- **Failure handling**: Site failure → VMs restart at surviving site using local data copies

**When to use**: Active-active workloads at metro distance where near-zero RPO/RTO justifies the infrastructure cost.

## Disaster Recovery Orchestration

### VMware Site Recovery Manager (SRM)

SRM automates the failover process. Without orchestration, DR is a runbook that someone executes manually at 3 AM during an outage -- error-prone and slow.

**What SRM does:**

1. **Recovery plans**: Ordered sequence of VM power-on, IP re-mapping, script execution
2. **Non-disruptive testing**: Test failover without impacting production (isolated bubble network)
3. **Re-protect**: After failover, reverse replication direction for failback
4. **Automation**: Network re-IP, DNS updates, custom scripts at each step

```
Recovery Plan: "Failover Production"
  Step 1: Power on DNS/AD servers           (Priority 1, wait for heartbeat)
  Step 2: Power on database servers         (Priority 2, run pre-script: check replication)
  Step 3: Power on application servers      (Priority 3, wait 60s for DB connection)
  Step 4: Power on web servers              (Priority 4, run post-script: health check)
  Step 5: Update load balancer VIP          (Custom step)
```

**SRM works with:**

- vSphere Replication (built-in)
- Array-based replication (via Storage Replication Adapters / SRAs)

### Non-SRM Orchestration

Alternatives when SRM isn't in the stack:

| Tool                             | Approach                                                                                                        |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Zerto                            | Continuous replication with journal-based recovery (any point in time). Hypervisor-level, not storage dependent |
| Veeam Replication + Orchestrator | VM replication with orchestrated failover plans                                                                 |
| Cloud-based DR                   | Replicate to VMware Cloud on AWS, Azure VMware Solution, or native cloud VMs                                    |

## DR Architecture Patterns

### Pattern 1: Active-Passive (Most Common)

```
Primary Site (active)              DR Site (passive)
  Production VMs                     Replica VMs (powered off)
  Full compute capacity              Minimal compute (scales up on failover)
       |                                  |
       +--- Async replication (RPO: hours) ---+
```

- **Cost**: Low (DR site can be smaller, replicas consume only storage)
- **RTO**: Hours (power on VMs, verify, re-IP)
- **When to use**: Silver/Bronze tier workloads, budget-constrained

### Pattern 2: Active-Active (Hot Standby)

```
Site A (active)                    Site B (active)
  50% of workloads                   50% of workloads
  Replicas of Site B                 Replicas of Site A
       |                                  |
       +--- Sync replication (RPO: zero) ---+
```

- **Cost**: High (both sites fully provisioned)
- **RTO**: Minutes (VMs restart at surviving site)
- **When to use**: Platinum/Gold tier, metro distance, can't tolerate data loss

### Pattern 3: Pilot Light

```
Primary Site (active)              DR Site (pilot light)
  Full production                    Core infrastructure only:
                                       - AD/DNS (running)
                                       - DB replicas (running)
                                       - App/Web (powered off, replicated)
       |                                  |
       +--- Replication ---+
```

- **Cost**: Moderate (only critical VMs running at DR)
- **RTO**: 1-4 hours (power on remaining VMs, scale compute)
- **When to use**: Gold tier -- keeps critical services warm, reduces recovery time vs full active-passive

### Pattern 4: Cloud DR

Replicate on-prem VMs to cloud infrastructure:

- **VMware Cloud on AWS / Azure VMware Solution**: vSphere-native DR -- SRM works the same way
- **Native cloud**: Convert VMDKs to AMIs/managed disks. Bigger architectural gap but no VMware licensing at DR site
- **Backup-based DR**: Restore from cloud-stored backups (Veeam, Commvault). Highest RTO but lowest cost

## Ransomware Resilience

Modern DR must account for an attacker who has been in the environment for weeks/months before detonation.

### Key Principles

1. **Immutable backups**: Backups that can't be deleted or encrypted (S3 Object Lock, hardened repos, air-gapped tape)
2. **Isolated recovery environment**: Test restores in a network-isolated environment before connecting to production
3. **Long retention**: Keep backups beyond typical dwell time (30-90+ days). If your oldest backup is 7 days and the attacker was in for 30, all your backups are compromised
4. **Backup infrastructure separation**: Backup admin credentials should not be in the same AD domain as production. Compromising AD shouldn't give access to backups

### Recovery Sequence for Ransomware

```
1. Isolate: Disconnect affected systems from network
2. Assess: Determine scope and dwell time (how long were they in?)
3. Identify clean restore point: Find backup predating compromise
4. Build clean environment: Rebuild AD/DNS in isolated network
5. Restore to isolation: Bring VMs up in quarantined environment
6. Validate: Scan for persistence mechanisms, verify data integrity
7. Reconnect: Gradually restore network connectivity
```

This is fundamentally different from a hardware failure DR plan. Hardware failure assumes clean data at the replica. Ransomware assumes the data itself may be compromised.

## Testing

### Test Types

| Test                      | Disruption  | What It Validates                                          |
| ------------------------- | ----------- | ---------------------------------------------------------- |
| Tabletop exercise         | None        | Runbook completeness, team readiness                       |
| Non-disruptive test (SRM) | None        | Automated failover in isolated network                     |
| Partial failover          | Minimal     | Specific application recovery                              |
| Full failover             | Significant | Complete site recovery (usually during maintenance window) |

### Testing Cadence

| Tier     | Minimum Frequency                         |
| -------- | ----------------------------------------- |
| Platinum | Monthly non-disruptive, quarterly partial |
| Gold     | Quarterly non-disruptive, annual full     |
| Silver   | Semi-annual non-disruptive                |
| Bronze   | Annual tabletop                           |

**If you haven't tested it, you don't have DR.** Untested recovery plans fail at the worst possible time.

## Common Mistakes

| Mistake                           | Consequence                                           | Fix                                            |
| --------------------------------- | ----------------------------------------------------- | ---------------------------------------------- |
| Same RPO/RTO for everything       | Over-spending on low-value workloads                  | Tier applications, protect accordingly         |
| Backup but no DR orchestration    | 12-hour recovery that should take 1 hour              | Automate with SRM, Zerto, or equivalent        |
| Testing backups but not restores  | Backups succeed, restores fail                        | Regularly test restore to isolated environment |
| Backup infra on same AD domain    | Ransomware encrypts backups too                       | Separate credential domain for backup admin    |
| RPO = backup frequency            | Misunderstanding -- RPO is max _acceptable_ loss      | RPO drives backup frequency, not the reverse   |
| No immutable copies               | Single ransomware event destroys all recovery points  | Implement at least one immutable backup tier   |
| Replication without orchestration | Replicas exist but nobody knows the failover sequence | Document and automate recovery plans           |
