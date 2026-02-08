# vSAN Architecture and Failure Domains

## What vSAN Is

vSAN is VMware's software-defined storage layer. It pools local disks (SSDs, NVMe, HDDs) across ESXi hosts in a cluster into a single shared datastore. No external SAN or NAS required -- the storage is the hosts.

```
+--Host 1--+  +--Host 2--+  +--Host 3--+  +--Host 4--+
| NVMe SSD |  | NVMe SSD |  | NVMe SSD |  | NVMe SSD |  ← Cache tier
| SSD  SSD |  | SSD  SSD |  | SSD  SSD |  | SSD  SSD |  ← Capacity tier
+----|-----+  +----|-----+  +----|-----+  +----|-----+
     |             |             |             |
     +------vSAN Distributed Datastore---------+
                         |
                    Single namespace
                    visible to all hosts
```

## Architecture Models

### vSAN Original Storage Architecture (OSA)

The original disk group model. Each host contributes one or more disk groups.

**Disk Group:**

- Exactly 1 cache disk (SSD or NVMe) per group
- 1-7 capacity disks per group
- Maximum 5 disk groups per host

```
Host
  +-- Disk Group 1
  |     +-- Cache: 400GB NVMe (write buffer + read cache)
  |     +-- Capacity: 1.8TB SSD
  |     +-- Capacity: 1.8TB SSD
  |     +-- Capacity: 1.8TB SSD
  |
  +-- Disk Group 2
        +-- Cache: 400GB NVMe
        +-- Capacity: 1.8TB SSD
        +-- Capacity: 1.8TB SSD
        +-- Capacity: 1.8TB SSD
```

**Cache behavior (OSA):**

- 70% of cache disk used for write buffer (destaged to capacity tier)
- 30% used for read cache (frequently accessed blocks)
- Cache disk failure = entire disk group is lost (all capacity disks in that group go offline)

**This is the critical OSA risk**: a single cache disk failure takes out the entire disk group, not just the cache. Size and plan accordingly.

### vSAN Express Storage Architecture (ESA)

Introduced in vSAN 8. Eliminates the disk group model entirely. Designed for all-NVMe configurations.

```
Host (ESA)
  +-- NVMe disk 1 (participates in both cache and capacity)
  +-- NVMe disk 2 (participates in both cache and capacity)
  +-- NVMe disk 3 (participates in both cache and capacity)
  +-- NVMe disk 4 (participates in both cache and capacity)
```

**Key differences from OSA:**

| Aspect               | OSA                                              | ESA                                        |
| -------------------- | ------------------------------------------------ | ------------------------------------------ |
| Disk organization    | Disk groups (cache + capacity)                   | Flat storage pool (all disks equal)        |
| Cache failure impact | Entire disk group lost                           | Single disk lost, no cascade               |
| Supported media      | HDD + SSD + NVMe                                 | NVMe only                                  |
| Compression          | Available                                        | Always on, more efficient (log-structured) |
| RAID-5/6 efficiency  | Requires 4+ fault domains for RAID-5             | Better space efficiency at same protection |
| Snapshots            | Redirect-on-write (performance penalty at depth) | Native, efficient snapshots                |
| Minimum hosts        | 3                                                | 3                                          |

**When to choose ESA:** New deployments with all-NVMe hardware on vSAN 8+. ESA is the direction VMware is investing in.

**When OSA still makes sense:** Existing clusters, mixed media (HDD + SSD), or hardware that doesn't meet ESA compatibility requirements.

## Storage Policies

vSAN uses policy-based storage management. Every VM object (VMDK, swap, snapshot) has a storage policy that defines its protection level and performance characteristics.

### Key Policy Settings

| Policy                         | What It Controls                                     | Default       |
| ------------------------------ | ---------------------------------------------------- | ------------- |
| **Failures to Tolerate (FTT)** | How many host/disk failures the object survives      | 1             |
| **Failure Tolerance Method**   | RAID-1 (mirroring) or RAID-5/6 (erasure coding)      | RAID-1        |
| **Stripe width**               | Number of capacity disks an object is striped across | 1             |
| **Force provisioning**         | Allow provisioning even if policy can't be satisfied | No            |
| **Object space reservation**   | Thick vs thin provisioning (% of VMDK pre-allocated) | 0% (thin)     |
| **IOPS limit**                 | Per-object IOPS cap                                  | 0 (unlimited) |

### FTT and Capacity Impact

FTT determines how many data copies or erasure coding fragments exist:

**RAID-1 (Mirroring):**

| FTT | Copies            | Hosts Required | Capacity Overhead |
| --- | ----------------- | -------------- | ----------------- |
| 0   | 1 (no protection) | 1              | 0%                |
| 1   | 2 mirrors         | 3              | 100% (2x raw)     |
| 2   | 3 mirrors         | 5              | 200% (3x raw)     |
| 3   | 4 mirrors         | 7              | 300% (4x raw)     |

**RAID-5/6 (Erasure Coding):**

| FTT | Method       | Hosts Required | Capacity Overhead |
| --- | ------------ | -------------- | ----------------- |
| 1   | RAID-5 (3+1) | 4              | 33%               |
| 2   | RAID-6 (4+2) | 6              | 50%               |

**Trade-off**: RAID-5/6 is far more space-efficient than mirroring but has higher CPU overhead for parity calculations and higher write latency. Use RAID-1 for write-intensive workloads (databases). Use RAID-5/6 for read-heavy or capacity-sensitive workloads.

### Policy Examples

```
Policy: mission-critical
  FTT: 2, RAID-1 (triple mirror)
  Stripe width: 2
  → 3 copies of every block, striped across 2 disks per copy
  → Survives 2 simultaneous host failures
  → Uses 3x raw capacity

Policy: general-production
  FTT: 1, RAID-1
  → Standard protection, 2x capacity overhead

Policy: space-efficient
  FTT: 1, RAID-5
  → Same protection as above, only 33% overhead
  → Higher write latency

Policy: dev-test
  FTT: 0
  → No protection. Single disk failure = data loss.
  → Only for disposable workloads
```

## Failure Domains

### Default Behavior (No Fault Domains)

Without explicit fault domains, vSAN treats each host as a fault domain. An FTT=1 policy places data copies on two different hosts. If both copies happen to be in the same rack and that rack loses power, both copies are lost.

### Explicit Fault Domains

Fault domains group hosts by physical failure boundary -- typically rack, but could be room, building, or power feed.

```
Fault Domain: Rack-A          Fault Domain: Rack-B          Fault Domain: Rack-C
  +-- Host 1                    +-- Host 3                    +-- Host 5
  +-- Host 2                    +-- Host 4                    +-- Host 6
```

With fault domains configured, vSAN places data copies in **different fault domains**, not just different hosts. FTT=1 means the object survives the loss of an entire fault domain (entire rack).

**Requirements:**

- FTT=1 requires at least 3 fault domains (just like it requires 3 hosts without fault domains)
- FTT=2 requires at least 5 fault domains
- Each fault domain should have roughly equal capacity (imbalanced domains waste space)

### Fault Domain Sizing

| Fault Domain Count | FTT Supported  | Typical Mapping |
| ------------------ | -------------- | --------------- |
| 3                  | FTT=1 (RAID-1) | 3 racks         |
| 4                  | FTT=1 (RAID-5) | 4 racks         |
| 5                  | FTT=2 (RAID-1) | 5 racks         |
| 6                  | FTT=2 (RAID-6) | 6 racks         |

### Stretched Clusters as Fault Domains

A vSAN stretched cluster is essentially two fault domains at two sites with a witness at a third:

```
Site A (Fault Domain)       Site B (Fault Domain)       Witness (Site C)
  Host 1, 2, 3               Host 4, 5, 6               Witness appliance
  Data copies                 Data copies                 Metadata only
```

- Every write goes to both sites synchronously (RPO = 0)
- Witness provides quorum -- breaks the tie if a site fails
- Witness stores only object metadata, not data (small appliance)
- Requires < 5ms RTT between data sites, < 200ms to witness

See [backup-and-dr.md](backup-and-dr.md) for broader DR patterns.

## Object Model

Everything on vSAN is an object. Understanding the object model explains vSAN behavior.

### Object Types

| Object         | What It Is                                   | Size                    |
| -------------- | -------------------------------------------- | ----------------------- |
| VMDK           | Virtual disk                                 | Variable (primary data) |
| VM Home        | VMX config, logs, snapshots                  | Small                   |
| VM Swap        | vswp file (configured RAM minus reservation) | Up to VM memory size    |
| Snapshot delta | Changed blocks since snapshot                | Grows over time         |

### Object Components

Each object is split into components distributed across hosts. For a 100GB VMDK with FTT=1, RAID-1:

```
Object: vm-prod-01.vmdk (100GB, FTT=1, RAID-1)
  |
  +-- Component: Mirror copy 1 (100GB) → Host 1, Disk Group 1
  +-- Component: Mirror copy 2 (100GB) → Host 3, Disk Group 2
  +-- Witness: (metadata only)         → Host 2
```

The witness is a tiny component that acts as a tiebreaker. With 2 mirror copies, a witness on a third host ensures quorum (2 of 3 components) survives a single host failure.

### Component Limits

| Limit                            | Value                                                  |
| -------------------------------- | ------------------------------------------------------ |
| Max components per host          | 9,000                                                  |
| Max components per vSAN cluster  | 90,000 (OSA), higher for ESA                           |
| Max object size before splitting | 255GB (objects > 255GB split into multiple components) |

Large VMs with multiple large VMDKs consume many components. A 2TB VMDK with FTT=1 RAID-1: `ceiling(2048/255) * 2 mirrors + witnesses = ~18 components`. Monitor component counts in large environments.

## Resync and Rebuild

### When Resync Happens

- Host failure (permanent) -- vSAN rebuilds data copies on surviving hosts
- Host maintenance mode (ensure accessibility vs full data migration)
- Disk failure -- affected components rebuilt on remaining disks
- Policy change -- objects restructured to match new policy

### Maintenance Mode Options

| Option               | What Happens                                                               | When to Use                                                    |
| -------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Ensure accessibility | No data migration. Objects may be under-protected during maintenance.      | Quick reboot, firmware update. Host returning soon.            |
| Full data migration  | All data moved off host. Full protection maintained.                       | Host leaving cluster permanently or for extended period.       |
| No data migration    | No movement at all. Objects may become inaccessible if host was sole copy. | Only with FTT > actual failures tolerated. Rarely appropriate. |

**Ensure accessibility** is the common choice for routine maintenance. It's fast (no data movement) and protection is restored when the host returns. Full data migration can take hours and generates significant I/O.

### Rebuild Mechanics

When a host fails permanently and vSAN needs to rebuild:

1. **60-minute delay** (default): vSAN waits before rebuilding, assuming the host might return (reboot, not failure)
2. **Component identification**: Identifies all components that were on the failed host
3. **Rebuild**: Creates new copies on hosts with available capacity
4. **Resync bandwidth**: Controlled by `VSAN.ResyncThrottleThreshold` -- balances rebuild speed vs impact on running workloads

**The 60-minute delay is deliberate.** Rebuilding is expensive (network, disk, CPU). If a host reboots and returns in 10 minutes, a premature rebuild wastes resources and creates unnecessary I/O pressure. Adjust the timer only if your environment has clear patterns (e.g., hosts never return within an hour means you can shorten it).

## Capacity Planning

### Usable Capacity Formula

```
Usable Capacity = (Total Raw - vSAN Overhead) / Protection Multiplier - Slack

Where:
  vSAN Overhead ≈ 1-2% of raw (metadata, checksums)
  Protection Multiplier = 2x for RAID-1 FTT=1, 1.33x for RAID-5 FTT=1
  Slack = 25-30% free for rebuilds and headroom
```

**Example: 6 hosts, each with 8TB raw capacity**

```
Total Raw:        48 TB
vSAN Overhead:    ~1 TB
Available:        47 TB
RAID-1 FTT=1:    47 / 2 = 23.5 TB protected
30% slack:        23.5 * 0.70 = 16.5 TB usable

RAID-5 FTT=1:    47 / 1.33 = 35.3 TB protected
30% slack:        35.3 * 0.70 = 24.7 TB usable
```

### Why 25-30% Slack

- **Rebuild capacity**: When a host fails, surviving hosts need free space to absorb rebuilt components. If you're at 95% full, there's nowhere to rebuild.
- **Performance cliff**: vSAN performance degrades as capacity fills. Write amplification increases, garbage collection competes with workload I/O.
- **vSAN health warnings**: vSAN alerts at 70% (warning) and 80% (critical) by default.

## Monitoring

### Key Metrics

| Metric            | Where to Find                             | What It Means                                                  |
| ----------------- | ----------------------------------------- | -------------------------------------------------------------- |
| Capacity used %   | vCenter > vSAN > Capacity                 | Overall fill level. Stay under 70-75%.                         |
| Resync operations | vCenter > vSAN > Resyncing Objects        | Active rebuilds. High count = recovery in progress.            |
| Backend latency   | vCenter > vSAN > Performance > Backend    | Disk-level latency. Spikes indicate disk or contention issues. |
| Congestion        | vCenter > vSAN > Performance > Congestion | vSAN backpressure. > 0 means vSAN is throttling I/O.           |
| Component health  | vCenter > vSAN > Health                   | Degraded/absent components. Should be zero in steady state.    |

### esxtop Disk Metrics

```
DAVG: Device average latency (disk-level, should be < 10ms for SSD, < 1ms for NVMe)
KAVG: Kernel average latency (ESXi overhead, should be < 2ms)
GAVG: Guest-observed latency (DAVG + KAVG, what the VM experiences)
```

If GAVG is high but DAVG is low, the bottleneck is in the ESXi stack, not the disks. If DAVG is high, the disks are the bottleneck.

## Common Mistakes

| Mistake                               | Consequence                                              | Fix                                                        |
| ------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| Running at > 80% capacity             | Rebuild failures, performance degradation                | Maintain 25-30% free space                                 |
| FTT=0 on anything you care about      | Single disk or host failure = data loss                  | FTT=1 minimum for all non-disposable workloads             |
| Ignoring the 60-minute rebuild timer  | Panic during normal host reboot                          | Understand the delay is intentional                        |
| Unequal fault domains                 | Capacity imbalance, wasted space on larger domains       | Size fault domains roughly equally                         |
| OSA with undersized cache disk        | Write buffer saturation, destaging bottleneck            | Cache disk should be >= 10% of disk group capacity         |
| Mixing disk types within a disk group | Performance inconsistency                                | All capacity disks in a group should be same type and size |
| No storage policy differentiation     | Everything gets same protection                          | Create policies matching workload tiers                    |
| Snapshots left running for days       | Delta VMDKs grow, component count increases, I/O penalty | Monitor and consolidate snapshots promptly                 |
