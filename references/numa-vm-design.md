# NUMA-Aware VM Design

This guide ties together concepts from [cpu-scheduler.md](cpu-scheduler.md) and [memory-sizing.md](memory-sizing.md) into practical VM sizing decisions based on NUMA topology.

## Why NUMA Matters

Modern servers are NUMA (Non-Uniform Memory Access) architectures. Each physical CPU socket has its own local memory. Accessing local memory is fast. Accessing memory attached to another socket is slower.

```
Socket 0                              Socket 1
+------------------+                  +------------------+
| 28 cores         |                  | 28 cores         |
| (56 threads w/HT)|                  | (56 threads w/HT)|
+--------+---------+                  +--------+---------+
         |                                     |
+--------+---------+                  +--------+---------+
| 256 GB local RAM |                  | 256 GB local RAM |
| ~80ns access     |                  | ~80ns access     |
+--------+---------+                  +--------+---------+
         |                                     |
         +---------- QPI / UPI link -----------+
                    ~130ns cross-socket
```

**The performance penalty for remote memory access is 1.5-2x local latency.** For latency-sensitive workloads (databases, in-memory caches), this matters. For general workloads, it's often negligible -- but poor NUMA alignment at scale wastes aggregate capacity.

## Know Your NUMA Topology

Before sizing VMs, know the physical topology of your hosts.

### Discovery Commands

```bash
# NUMA node count and core distribution
esxcli hardware cpu global get

# Memory per NUMA node
esxcli hardware memory get

# Detailed NUMA properties
vsish -e cat /hardware/numa/node0/properties
vsish -e cat /hardware/numa/node1/properties

# In esxtop, press 'm' for memory view, then 'f' to add NUMA fields
```

### Common Server Topologies

| Server      | Sockets | Cores/Socket | Threads/Socket (HT) | RAM/Socket | NUMA Nodes |
| ----------- | ------- | ------------ | ------------------- | ---------- | ---------- |
| 2S typical  | 2       | 16-32        | 32-64               | 128-512GB  | 2          |
| 2S high-end | 2       | 32-56        | 64-112              | 512GB-1TB  | 2          |
| 4S          | 4       | 24-32        | 48-64               | 256-768GB  | 4          |

**Sub-NUMA Clustering (SNC):** Some Intel processors (Sapphire Rapids, Emerald Rapids) can split each socket into 2 NUMA nodes. A 2-socket server with SNC enabled presents 4 NUMA nodes, each with half the cores and half the memory of a full socket. Check your BIOS settings.

## The Core Rule: Fit VMs Within a NUMA Node

The single most important NUMA design principle:

**Size VMs so their vCPUs and memory fit within a single NUMA node.**

A VM that fits in one NUMA node gets:

- All memory accesses at local latency
- No cross-socket scheduling overhead
- Simpler scheduler decisions (less co-stop risk)

A VM that spans NUMA nodes gets:

- Some memory accesses at remote latency (1.5-2x slower)
- vCPU scheduling across sockets
- vNUMA exposure to help the guest optimize (but the guest has to cooperate)

### Calculating NUMA Node Capacity

```
Physical cores per NUMA node = Total cores / NUMA nodes
Logical processors per NUMA node = Physical cores × 2 (with HT)
Memory per NUMA node = Total host RAM / NUMA nodes
```

**Example: Dual-socket, 28 cores/socket, 512GB total, HT enabled**

```
NUMA nodes: 2
Physical cores per node: 28
Logical processors per node: 56
Memory per node: 256GB
```

**VM sizing target**: Up to 28 vCPUs and 256GB RAM to stay within one NUMA node.

### Practical Sizing Table

For a 2-socket host with 28 cores/socket, 256GB/socket:

| VM Size        | Fits in 1 NUMA Node?   | NUMA Impact                                             |
| -------------- | ---------------------- | ------------------------------------------------------- |
| 4 vCPU, 16GB   | Yes                    | Optimal. Fully local.                                   |
| 8 vCPU, 64GB   | Yes                    | Optimal. Fully local.                                   |
| 16 vCPU, 128GB | Yes                    | Optimal. Fully local.                                   |
| 28 vCPU, 256GB | Yes (exactly fills it) | Optimal, but leaves no room for other VMs on that node. |
| 32 vCPU, 384GB | No (spans 2 nodes)     | Wide VM. vNUMA exposed. Some remote memory access.      |
| 56 vCPU, 512GB | No (fills both nodes)  | Monster VM. Consumes entire host.                       |

## Wide VMs (Spanning NUMA Nodes)

When a VM must be larger than a single NUMA node, ESXi handles it by creating a **NUMA client per node**:

```
VM: 32 vCPU, 384GB (on 2-node host, 28 cores/node)
  |
  +-- NUMA Client 0 → Socket 0: 16 vCPUs, 192GB
  +-- NUMA Client 1 → Socket 1: 16 vCPUs, 192GB
```

The scheduler splits the VM across nodes as evenly as possible. Each NUMA client gets local memory on its node. Cross-client memory access (vCPU on node 0 accessing memory allocated by node 1) goes remote.

### How ESXi Splits Wide VMs

The default behavior (numa.autosize = TRUE):

```
vCPUs per NUMA client = ceiling(total vCPUs / NUMA nodes used)
Memory per NUMA client = proportional to vCPU split
```

**The split doesn't have to be even.** ESXi can create asymmetric NUMA clients if the vCPU count doesn't divide evenly. For example, a 24-vCPU VM on a 2-node host with 28 cores/node:

```
Option A: 1 NUMA client, 24 vCPUs → fits on one node (24 < 28) ← ESXi prefers this
Option B: 2 NUMA clients, 12+12 → spans nodes unnecessarily
```

ESXi will keep the VM on one node if it fits. It only splits when necessary.

### vNUMA: Exposing Topology to the Guest

When a VM spans NUMA nodes, ESXi exposes the physical NUMA topology to the guest OS as **vNUMA (Virtual NUMA)**. The guest OS sees multiple NUMA nodes and can optimize:

- Memory allocators prefer local node
- Thread schedulers try to keep threads near their data
- Database engines (SQL Server, Oracle) can bind instances to NUMA nodes

**vNUMA is exposed when:**

- VM has more vCPUs than `numa.vcpu.min` (default: 8)
- VM memory exceeds a single NUMA node's capacity
- `numa.autosize` is TRUE (default)

**vNUMA is hidden when:**

- VM fits within a single NUMA node (no need to expose it)
- `numa.vcpu.min` is set higher than the VM's vCPU count
- Manually overridden in VM advanced settings

### vNUMA and Hot-Add

**CPU/memory hot-add disables vNUMA.** If hot-add is enabled on the VM, ESXi does not expose virtual NUMA topology to the guest. The guest sees a flat UMA architecture and can't optimize for NUMA locality.

This is a common hidden performance penalty:

| Setting          | vNUMA   | Guest NUMA-Aware | Impact                                                   |
| ---------------- | ------- | ---------------- | -------------------------------------------------------- |
| Hot-add disabled | Exposed | Yes              | Guest can optimize memory placement                      |
| Hot-add enabled  | Hidden  | No               | Guest treats all memory as uniform, more remote accesses |

**Recommendation**: Disable CPU and memory hot-add on wide VMs and performance-sensitive workloads. Size the VM correctly upfront instead.

## Sizing Guidelines by Workload

### Databases (SQL Server, Oracle, PostgreSQL)

Databases are the most NUMA-sensitive common workload. Buffer pools, hash tables, and sort areas benefit enormously from local memory access.

**Rules:**

- Size to fit within one NUMA node if at all possible
- If the VM must span nodes, ensure vNUMA is exposed (disable hot-add)
- SQL Server and Oracle are NUMA-aware -- they'll bind worker threads and buffer pool regions to NUMA nodes if vNUMA is visible
- Set memory reservation to 80-90% to prevent ballooning (see [vm-reservations.md](vm-reservations.md))

**Example sizing for 28-core/256GB NUMA node:**

```
Small database:   8 vCPU,  64GB  → 1 NUMA node, optimal
Medium database: 16 vCPU, 128GB  → 1 NUMA node, optimal
Large database:  28 vCPU, 256GB  → 1 NUMA node, fills it
XL database:     48 vCPU, 384GB  → 2 NUMA nodes, ensure vNUMA exposed
```

### Application Servers (Java, .NET, Node.js)

Less NUMA-sensitive than databases. JVM and CLR garbage collectors do cross-thread memory access patterns that are hard to NUMA-optimize.

**Rules:**

- Keep within one NUMA node when practical
- Don't over-allocate vCPUs (see co-stop discussion in [cpu-scheduler.md](cpu-scheduler.md))
- JVM `-XX:+UseNUMA` flag helps the JVM allocate objects on the NUMA node where the allocating thread runs (if vNUMA is visible)

### VDI Desktops

Small, numerous VMs. NUMA is rarely a concern per-VM since desktops are 2-4 vCPU, 4-8GB -- far below any NUMA node boundary.

**The NUMA concern for VDI is density**: How many VMs can you pack per NUMA node? If you have 28 cores per node and each desktop is 2 vCPU, that's 14 desktops per node at 1:1 overcommit, or 28+ with overcommit.

### In-Memory Analytics (Spark, SAP HANA)

These workloads scan large memory regions sequentially. Remote NUMA access on every scan iteration adds up.

**Rules:**

- SAP HANA requires NUMA-aligned VM sizing (SAP publishes specific vCPU-to-memory ratios per NUMA node)
- Size the VM as a multiple of NUMA node capacity
- Avoid odd vCPU counts that create asymmetric NUMA clients

## Advanced vNUMA Configuration

### Manual vNUMA Topology

In rare cases, you may need to override ESXi's automatic vNUMA calculation:

```
# Force specific vCPUs per NUMA client
numa.vcpu.maxPerVirtualNode = 16

# Force specific NUMA node count
numa.nodeAffinity = 0,1
```

**When to override:**

- Application has specific NUMA affinity requirements (e.g., SAP HANA sizing notes)
- ESXi auto-calculation produces suboptimal split for a specific workload
- Testing NUMA performance characteristics

**When NOT to override:**

- General workloads -- ESXi's auto-sizing is correct in the vast majority of cases
- You're guessing -- incorrect NUMA overrides hurt more than the default

### Cores Per Socket (CPS) Setting

The VM's "Cores per Socket" setting affects vNUMA topology:

```
VM: 16 vCPU
  CPS = 16 → 1 socket × 16 cores → 1 vNUMA node
  CPS = 8  → 2 sockets × 8 cores → 2 vNUMA nodes
  CPS = 4  → 4 sockets × 4 cores → 4 vNUMA nodes
```

**Best practice**: Set CPS to match the physical cores per NUMA node. This aligns the VM's virtual topology with the physical topology.

**Example**: Host has 28 cores per socket. A 28-vCPU VM should be configured as 1 socket × 28 cores (CPS=28), not 28 sockets × 1 core or 2 sockets × 14 cores.

**For VMs that fit in one NUMA node**: CPS doesn't matter much -- ESXi places everything on one node regardless.

**For wide VMs**: CPS matters because it determines how many vNUMA nodes the guest sees, which affects guest-level NUMA optimization.

### Preferred NUMA Node

ESXi assigns a "home" NUMA node to each VM at power-on and periodically rebalances. You can influence (but shouldn't usually force) this:

```
# Suggest preferred node (scheduler may override)
numa.nodeAffinity = 0

# Hard pin (breaks rebalancing -- use only for extreme cases)
numa.nodeAffinity = 0
numa.autosize = FALSE
```

**Don't hard-pin unless you have a specific, measurable reason.** It prevents the scheduler from rebalancing across nodes, which can leave one node overloaded while the other is idle.

## Monitoring NUMA Alignment

### esxtop NUMA Metrics

In esxtop memory view (`m`), add NUMA fields (`f`):

| Metric | What It Shows                      | Target                                       |
| ------ | ---------------------------------- | -------------------------------------------- |
| NHN    | NUMA home node                     | Should be stable (not constantly changing)   |
| NRMEM  | Remote memory (MB)                 | Should be close to 0 for VMs within one node |
| NLMEM  | Local memory (MB)                  | Should be close to total VM memory           |
| N%L    | Percentage of memory that is local | > 95% for NUMA-sensitive workloads           |

**Key diagnostic**: If N%L is below 80%, the VM has significant remote memory access. Either the VM spans nodes unnecessarily or NUMA rebalancing placed memory poorly.

### Checking NUMA Placement via CLI

```bash
# See NUMA node assignments for all VMs
vscsiStats -l  # (for storage stats, not NUMA -- see below)

# NUMA client info via vsish
vsish -e cat /vm/VMID/numa/topology

# Per-VM NUMA stats in esxtop
# Press 'm', then 'f', enable NUMA fields
```

## Decision Flowchart

```
VM needs X vCPUs and Y GB RAM
    |
    v
Does it fit within 1 NUMA node?
(X ≤ cores/node AND Y ≤ RAM/node)
    |
    +-- Yes → Set CPS = X (or cores/node). Disable hot-add if perf-sensitive.
    |         Done. Optimal placement.
    |
    +-- No → Can the VM be made smaller?
              |
              +-- Yes → Right-size first. Fewer vCPUs often means
              |         better performance (less co-stop, NUMA-local).
              |
              +-- No → Wide VM. Accept multi-node placement.
                       |
                       +-- Disable CPU/memory hot-add (enable vNUMA)
                       +-- Set CPS = physical cores per socket
                       +-- Verify vNUMA is exposed (guest sees NUMA nodes)
                       +-- Configure guest app for NUMA awareness
                       +-- Monitor N%L in esxtop (target > 80%)
```

## Common Mistakes

| Mistake                                                                     | Consequence                                                                                     | Fix                                                                  |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| VM vCPU count slightly over NUMA node size (e.g., 30 vCPUs on 28-core node) | Forces 2-node split for 2 extra vCPUs. Major penalty for marginal gain.                         | Reduce to 28 vCPUs or accept full 2-node split with proper alignment |
| Hot-add enabled on wide VMs                                                 | vNUMA hidden from guest, guest can't optimize                                                   | Disable hot-add, size VM correctly upfront                           |
| CPS set to 1 (1 core per socket)                                            | Guest sees many sockets, Windows licensing may require more licenses, vNUMA topology misaligned | Set CPS to match physical cores per NUMA node                        |
| Hard-pinning NUMA affinity                                                  | Prevents scheduler rebalancing, creates hotspots                                                | Let ESXi manage placement except in extreme cases                    |
| Ignoring NUMA for "small" VMs                                               | Usually fine, but 50 small VMs on one node while the other is empty                             | Trust ESXi rebalancing, but monitor node balance                     |
| Sizing VMs as multiples of 2 by habit                                       | 32 vCPUs on a 28-core node forces unnecessary NUMA split                                        | Size to NUMA node boundary, not powers of 2                          |
