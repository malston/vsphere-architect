# Cluster Design Patterns

## Why Cluster Design Matters

A vSphere cluster is the boundary for HA, DRS, resource pools, and admission control. How you draw those boundaries determines your failure domains, resource contention model, and operational complexity.

## Cluster Types

### Management Cluster

Runs the infrastructure control plane: vCenter, NSX Manager, SDDC Manager, Aria suite, backup servers, monitoring.

**Why separate it:**

- Control plane availability during compute failures. If vCenter is in the same cluster as production VMs and DRS misbehaves, you can't fix it via vCenter because vCenter is affected.
- Different lifecycle -- management components upgrade on a different cadence than workloads
- Smaller blast radius -- management cluster is typically 3-4 hosts

**Sizing:**

| Component        | Typical Resources     |
| ---------------- | --------------------- |
| vCenter          | 4 vCPU, 24GB RAM      |
| NSX Manager (x3) | 6 vCPU, 24GB RAM each |
| SDDC Manager     | 4 vCPU, 16GB RAM      |
| Aria Operations  | 4-8 vCPU, 32GB RAM    |
| Aria Automation  | 4-8 vCPU, 16GB RAM    |

**Rule of thumb**: 3-4 hosts with 256-512GB RAM each is sufficient for most management clusters. N+1 HA means one host can fail and everything still runs.

### Compute Cluster

Runs application workloads. This is where most VMs live.

**Design variables:**

- How many hosts per cluster
- Whether to separate workload types (see patterns below)
- DRS automation level
- HA admission control policy

### Edge / DMZ Cluster

Runs workloads with external network exposure: load balancers, reverse proxies, NSX Edge nodes, WAFs.

**Why separate it:**

- Security boundary -- different network access than internal compute
- NSX Edge nodes need dedicated hosts for predictable performance (uplink bandwidth, NAT throughput)
- Compliance -- DMZ hosts may have different hardening requirements

**Sizing**: Typically 2-4 hosts. NSX Edge VMs are the primary tenants. Edge nodes should have CPU and memory reservations for consistent data plane performance.

## Cluster Sizing

### Hosts Per Cluster

| Cluster Size | Pros                                                 | Cons                                                          | When to Use                                  |
| ------------ | ---------------------------------------------------- | ------------------------------------------------------------- | -------------------------------------------- |
| 3-4 hosts    | Simple, low overhead, easy to reason about           | Limited capacity, N+1 means losing 25-33% to HA               | Management clusters, small environments      |
| 6-8 hosts    | Sweet spot for most environments. N+1 cost is 12-17% | More DRS decisions, moderate complexity                       | Production compute                           |
| 12-16 hosts  | High density, low HA overhead percentage             | DRS complexity, larger failure domain                         | Large-scale compute, VDI                     |
| 32+ hosts    | Maximum density                                      | vCenter performance considerations, very large failure domain | Rarely justified unless workloads require it |

**Practical ceiling**: vSphere supports up to 96 hosts per cluster, but most environments stay under 16. Larger clusters increase the blast radius of misconfigurations and make DRS behavior harder to predict.

### Right-Sizing the N+1 Question

HA reserves capacity for host failures. The cost of that reservation depends on cluster size:

```
HA overhead = Hosts to Tolerate / Total Hosts

3-host cluster, tolerate 1:  33% reserved
4-host cluster, tolerate 1:  25% reserved
6-host cluster, tolerate 1:  17% reserved
8-host cluster, tolerate 1:  13% reserved
12-host cluster, tolerate 1:  8% reserved
```

Larger clusters are more efficient for HA, but that efficiency comes with trade-offs in failure domain size and operational complexity.

### Host Homogeneity

**Strongly recommended: identical hosts within a cluster.**

- DRS works best when all hosts have the same capacity (no "small host" that can't accept large VMs)
- HA slot sizing is simpler (no host-specific capacity differences)
- vMotion compatibility requires matching CPU families (EVC mode helps but adds constraints)
- Spare parts and firmware management are simpler

**When heterogeneous is acceptable:**

- Temporary during hardware refresh (migrating from old to new generation)
- Adding GPU hosts to an existing cluster (if passthrough, those hosts have different workload profiles anyway)

Use EVC (Enhanced vMotion Compatibility) to set a CPU baseline across the cluster. This allows mixed CPU generations to vMotion freely at the cost of disabling newer CPU features.

## Design Patterns

### Pattern 1: Collapsed (Everything in One Cluster)

```
+------------------------------------------+
|            Single Cluster                 |
|  Management VMs + Production + Dev/Test   |
|  (3-4 hosts)                              |
+------------------------------------------+
```

- **When**: Small environments (<50 VMs), limited hardware budget
- **Risk**: vCenter failure during cluster-wide issue means no management plane
- **Mitigation**: At minimum, set VM-VM anti-affinity rules to keep vCenter and NSX Manager on separate hosts

### Pattern 2: Management + Compute (Most Common)

```
+-------------------+    +-------------------+
| Management Cluster|    | Compute Cluster   |
| (3-4 hosts)       |    | (6-12 hosts)      |
| vCenter, NSX,     |    | Production VMs    |
| monitoring, backup|    | Dev/Test VMs      |
+-------------------+    +-------------------+
```

- **When**: Most production environments
- **Benefit**: Management plane survives compute cluster issues
- **Consideration**: If running dev and production in the same compute cluster, use resource pools and shares to prioritize production during contention

### Pattern 3: Management + Prod + Dev (Environment Separation)

```
+-------------------+    +-------------------+    +-------------------+
| Management Cluster|    | Production Cluster|    | Dev/Test Cluster  |
| (3-4 hosts)       |    | (6-12 hosts)      |    | (3-4 hosts)       |
| vCenter, NSX,     |    | Production VMs    |    | Dev/Test VMs      |
| monitoring, backup|    | HA: strict        |    | HA: relaxed       |
|                   |    | Overcommit: low   |    | Overcommit: high  |
+-------------------+    +-------------------+    +-------------------+
```

- **When**: Compliance requires environment isolation, or dev workloads are unpredictable enough to impact production
- **Benefit**: Different overcommit ratios, HA policies, and patching schedules per environment
- **Cost**: More hosts, more management overhead

### Pattern 4: Management + Compute + Edge

```
+-------------------+    +-------------------+    +-------------------+
| Management Cluster|    | Compute Cluster   |    | Edge Cluster      |
| (3-4 hosts)       |    | (6-12 hosts)      |    | (2-4 hosts)       |
| vCenter, NSX Mgr  |    | Application VMs   |    | NSX Edge VMs      |
| monitoring, backup|    |                   |    | Load balancers    |
|                   |    |                   |    | DMZ workloads     |
+-------------------+    +-------------------+    +-------------------+
```

- **When**: NSX deployed, need predictable edge performance, security zone separation
- **Benefit**: Edge nodes don't compete with application VMs for resources. Clear security boundary.
- **Consideration**: Edge cluster hosts need adequate NIC bandwidth for north-south traffic

### Pattern 5: Workload-Specific Clusters

```
+----------+  +----------+  +----------+  +----------+  +----------+
| Mgmt     |  | General  |  | Database |  | VDI      |  | GPU/AI   |
| Cluster  |  | Compute  |  | Cluster  |  | Cluster  |  | Cluster  |
| (3-4)    |  | (8-12)   |  | (4-6)    |  | (8-16)   |  | (2-4)    |
+----------+  +----------+  +----------+  +----------+  +----------+
```

- **When**: Large enterprise with distinct workload profiles
- **Database cluster**: Hosts with high memory, fast storage, memory reservations without inflating other clusters' HA slots
- **VDI cluster**: High density, specific GPU profiles (vGPU), burst capacity
- **GPU/AI cluster**: Hosts with NVIDIA GPUs, passthrough or MIG, different DRS constraints
- **Benefit**: Each cluster optimized for its workload profile. No cross-workload resource contention. HA and DRS tuned per workload type.
- **Cost**: More clusters = more hosts (HA overhead per cluster), more operational complexity

## Resource Pools

### When They Help

Resource pools partition a cluster's resources among groups of VMs. They're useful within a shared compute cluster:

```
Compute Cluster
  |
  +-- Resource Pool: Production (High shares)
  |     +-- VM: prod-web-01
  |     +-- VM: prod-app-01
  |     +-- VM: prod-db-01
  |
  +-- Resource Pool: Dev/Test (Low shares)
        +-- VM: dev-web-01
        +-- VM: dev-app-01
```

During contention, production gets priority proportional to its share allocation. During normal operation, dev/test can use idle capacity.

### When They Don't Help

- **Resource pools are not folders.** Don't use them for organizational hierarchy. That's what VM folders are for.
- **Nested resource pools** create complex, hard-to-predict share inheritance. Keep the tree shallow (one level preferred, two maximum).
- **Reservations on resource pools** are hard to get right and create rigid capacity partitions. Prefer shares for priority and let DRS handle placement.

### Resource Pool Anti-Patterns

| Anti-Pattern              | Problem                                 | Fix                                                                      |
| ------------------------- | --------------------------------------- | ------------------------------------------------------------------------ |
| Deep nesting (3+ levels)  | Share calculations become unpredictable | Flatten to 1-2 levels                                                    |
| One resource pool per VM  | Defeats the purpose, adds overhead      | Group by workload class                                                  |
| Reservation on every pool | Rigid partitioning, defeats overcommit  | Use shares for priority, reserve only at VM level for critical workloads |
| Using pools as folders    | Cluttered pool hierarchy                | Use VM folders for organization, pools for resource allocation           |

## DRS and Cluster Design

DRS behavior varies with cluster design decisions:

| Design Choice          | DRS Impact                                                                    |
| ---------------------- | ----------------------------------------------------------------------------- |
| Homogeneous hosts      | DRS can place VMs anywhere equally. Simple, predictable.                      |
| Heterogeneous hosts    | DRS must account for capacity differences. May leave large hosts underloaded. |
| VM-Host affinity rules | Constrains DRS placement. Too many rules and DRS can't balance effectively.   |
| Large VMs (wide NUMA)  | Fewer valid placement options. DRS has less flexibility.                      |
| Memory reservations    | DRS must respect reservation capacity. Limits migration options.              |

**DRS automation levels by cluster type:**

| Cluster         | Recommended Level             | Rationale                                  |
| --------------- | ----------------------------- | ------------------------------------------ |
| Management      | Partially automated           | Control when management VMs move           |
| General compute | Fully automated               | Let DRS handle balancing                   |
| Database        | Manual or partially automated | DB vMotion can cause performance blips     |
| VDI             | Fully automated               | Rapid balancing for login storms           |
| Edge            | Partially automated           | Edge VM placement matters for traffic flow |

See [drs-tuning.md](drs-tuning.md) for migration threshold and aggressiveness settings.

## Stretched Clusters

A single vSphere cluster spanning two physical sites.

```
Site A                                Site B
  ESXi hosts (rack 1-3)                ESXi hosts (rack 4-6)
       |                                    |
       +-------- L2 stretched / L3 ---------+
       |                                    |
  vSAN fault domain A               vSAN fault domain B
                    |
              Witness (Site C)
```

### Requirements

- < 5ms RTT between sites
- 10Gbps+ dedicated bandwidth
- Witness host or VM at a third site (for quorum)
- vSAN or shared storage accessible from both sites

### When Stretched Makes Sense

- Metro-distance active-active with zero RPO
- Workloads that must survive a full site failure without data loss
- Sites close enough that latency doesn't impact application performance

### When Stretched Doesn't Make Sense

- Sites > 100km apart (latency kills write performance)
- Applications sensitive to storage latency (synchronous replication adds round-trip to every write)
- Budget doesn't justify the bandwidth and infrastructure cost
- You only need active-passive DR (use replication + SRM instead)

### Stretched Cluster + HA

When a site fails, HA restarts VMs at the surviving site. Key considerations:

- VMs using vSAN have data copies at both sites. Surviving site has local copies -- no performance cliff.
- Admission control must account for losing an entire site worth of hosts (typically 50% of cluster)
- Set HA to tolerate host failures equal to the number of hosts at one site

## Putting It Together: Decision Framework

```
How many VMs?
├── < 50 VMs, limited budget
│   └── Pattern 1: Collapsed (single cluster)
│
├── 50-500 VMs
│   ├── Single environment?
│   │   └── Pattern 2: Management + Compute
│   ├── Need env separation?
│   │   └── Pattern 3: Management + Prod + Dev
│   └── Running NSX with edge requirements?
│       └── Pattern 4: Management + Compute + Edge
│
└── 500+ VMs or specialized workloads
    └── Pattern 5: Workload-specific clusters
        (add clusters as workload profiles diverge)
```

Start simple. Add clusters when you have a concrete reason -- workload isolation, compliance, performance characteristics -- not because the diagram looks more impressive.

## Common Mistakes

| Mistake                                  | Consequence                                                     | Fix                                                         |
| ---------------------------------------- | --------------------------------------------------------------- | ----------------------------------------------------------- |
| vCenter in the compute cluster           | Management plane failure during compute issues                  | Separate management cluster                                 |
| Too many small clusters                  | High HA overhead, underutilized hosts                           | Consolidate where isolation isn't required                  |
| One massive cluster                      | Large failure domain, complex DRS behavior                      | Split when cluster exceeds 12-16 hosts without clear reason |
| Deep resource pool nesting               | Unpredictable share behavior                                    | Flatten to 1-2 levels, use folders for organization         |
| Heterogeneous hosts without EVC          | vMotion failures between CPU generations                        | Enable EVC at lowest common CPU level                       |
| No anti-affinity for critical VMs        | HA, vCenter, NSX Manager on same host = single point of failure | VM-VM anti-affinity for infrastructure components           |
| Stretched cluster over high-latency link | Write performance tanks, application timeouts                   | Verify < 5ms RTT before committing to stretched design      |
