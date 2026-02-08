# Capacity Planning

## The Goal

Capacity planning answers three questions:

1. **How much capacity do I have right now?** (current state)
2. **When will I run out?** (trend projection)
3. **What do I need to buy and when?** (procurement planning)

Get it wrong in one direction and you waste money on idle hardware. Get it wrong in the other and you run out of capacity during a critical workload ramp.

## Capacity Dimensions

Every cluster has four capacity dimensions. You hit a wall when any one of them is exhausted:

| Dimension   | What Limits It                       | Typical Bottleneck                                      |
| ----------- | ------------------------------------ | ------------------------------------------------------- |
| **CPU**     | Physical cores × frequency           | Rarely the first limit in modern servers                |
| **Memory**  | Physical RAM per host                | Most common limit, especially with overcommit           |
| **Storage** | Datastore capacity and IOPS          | vSAN capacity fills; IOPS contention on shared storage  |
| **Network** | NIC bandwidth, port group throughput | Less common, except for vSAN and vMotion heavy clusters |

**Memory is almost always the first constraint.** CPU overcommit is efficient (scheduler handles contention well). Memory overcommit has real costs (ballooning, compression, swap). Most clusters run out of memory before CPU.

## Measuring Current Capacity

### Usable vs Raw Capacity

Raw capacity is what you bought. Usable capacity is what workloads can actually consume after overhead.

**CPU:**

```
Raw CPU = Hosts × Cores × GHz
ESXi overhead ≈ 5-10% of raw
HA reserve = Raw × (1 / Host count)   [for N+1]
Usable CPU = Raw - ESXi overhead - HA reserve
```

**Memory:**

```
Raw Memory = Hosts × RAM per host
ESXi overhead ≈ 2-4GB per host + ~100MB + (VM_RAM_GB × 8MB) per VM
HA reserve = Raw × (1 / Host count)
Usable Memory = Raw - ESXi overhead - HA reserve
```

**Storage (vSAN):**

```
Raw Storage = Hosts × capacity disks × disk size
vSAN overhead ≈ 1-2%
Protection overhead = depends on FTT/RAID (see vsan-architecture.md)
Slack = 25-30% for rebuilds and headroom
Usable = (Raw - overhead) / protection multiplier × (1 - slack)
```

### Worked Example

**Cluster: 6 hosts, each with 2 × 28-core CPUs, 512GB RAM, 8 × 1.92TB NVMe**

```
CPU:
  Raw: 6 × 56 cores × 2.4 GHz = 806 GHz
  ESXi overhead (~7%): 56 GHz
  HA reserve (N+1, 1/6): 134 GHz
  Usable: 616 GHz

Memory:
  Raw: 6 × 512GB = 3,072 GB
  ESXi overhead (~18GB): 18 GB
  HA reserve (1/6): 512 GB
  Usable: 2,542 GB

Storage (vSAN, RAID-1 FTT=1):
  Raw: 6 × 8 × 1.92TB = 92.2 TB
  vSAN overhead (1.5%): 1.4 TB
  Protection (RAID-1, 2x): 90.8 / 2 = 45.4 TB
  Slack (30%): 45.4 × 0.70 = 31.8 TB usable
```

### Measuring Actual Utilization

Raw capacity minus overhead gives you what's available. Actual utilization tells you what's consumed.

**vCenter metrics to track:**

| Metric                  | Where                           | What It Tells You                               |
| ----------------------- | ------------------------------- | ----------------------------------------------- |
| Cluster CPU usage %     | Cluster > Monitor > Utilization | How much CPU is being consumed                  |
| Cluster memory usage %  | Cluster > Monitor > Utilization | How much memory is consumed (active + overhead) |
| Cluster memory consumed | Cluster > Monitor > Utilization | Total physical RAM backing VMs                  |
| Datastore capacity used | Datastore > Summary             | Storage fill level                              |
| Per-host CPU ready %    | Host > Monitor > Performance    | Whether hosts are saturated                     |

**The gap between "allocated" and "consumed" is your overcommit opportunity** (or your risk, depending on perspective):

```
Allocated memory (sum of all VM configured RAM):  4,000 GB
Consumed memory (physical RAM actually used):     2,200 GB
Overcommit ratio:                                 4,000 / 3,072 = 1.3:1
Actual utilization:                               2,200 / 3,072 = 72%
```

## Right-Sizing VMs

Right-sizing is the highest-leverage capacity activity. Oversized VMs waste resources without improving performance.

### Identifying Oversized VMs

**CPU:**

```
Consistently < 25% average CPU usage over 30 days → candidate for reduction
```

Check actual utilization inside the guest OS, not just vSphere metrics. A 4-vCPU VM at 25% in vSphere is using ~1 vCPU of real work. Reduce to 2 vCPUs -- performance stays the same, scheduling improves (less co-stop).

**Memory:**

```
Active memory consistently < 50% of configured over 30 days → candidate for reduction
```

Active memory (not consumed) is the right metric. Consumed includes guest OS caches that will be released under pressure. Active memory is what the VM is actually using.

**Storage:**

```
Provisioned disk >> actual used space inside guest → thin provision, reclaim space
```

### Right-Sizing Process

```
1. Collect: 30+ days of utilization data (CPU, memory, disk)
2. Analyze: Identify VMs consistently below threshold
3. Recommend: Propose specific reductions with justification
4. Validate: Get application owner sign-off
5. Implement: Reduce in stages (e.g., 8 → 4 vCPU, not 8 → 1)
6. Monitor: Verify no performance degradation for 2 weeks
7. Repeat: Quarterly right-sizing cycle
```

**Never right-size without application owner buy-in.** A VM might look idle for 30 days but run a month-end batch that needs all its resources.

### Right-Sizing Caveats

| Caveat                | Why It Matters                                                                               |
| --------------------- | -------------------------------------------------------------------------------------------- |
| Seasonal workloads    | Holiday retail, year-end processing. 30-day average misses peaks.                            |
| DR VMs                | Powered off or idle until failover. Don't right-size your DR replicas based on idle metrics. |
| Newly deployed VMs    | Need time to reach steady state. Don't right-size in the first 30 days.                      |
| Database buffer pools | Databases allocate all available memory intentionally. High consumption is expected.         |

## Growth Modeling

### Linear Projection

The simplest model. Assumes growth continues at the same rate.

```
Current usage:     2,200 GB memory consumed
6-month trend:     +100 GB/month
Usable capacity:   2,542 GB
Time to exhaustion: (2,542 - 2,200) / 100 = 3.4 months
```

**When linear works:** Steady organic growth, predictable workload additions.

**When linear fails:** Project-based growth (new app deployments), acquisitions, seasonal patterns.

### Step-Function Growth

Many environments don't grow linearly. They grow in steps when new projects deploy:

```
Month 1-3:   +50 GB/month (organic growth)
Month 4:     +400 GB (new ERP deployment)
Month 5-8:   +50 GB/month (organic)
Month 9:     +200 GB (VDI expansion)
```

**Model step-function growth by combining:**

- Baseline organic growth rate (from historical trend)
- Known project deployments (from project pipeline)
- Buffer for unplanned growth (15-20%)

### Capacity Model Inputs

| Input                  | Source                        | Refresh Frequency |
| ---------------------- | ----------------------------- | ----------------- |
| Current utilization    | vCenter / Aria Operations     | Real-time         |
| Historical growth rate | 6-12 months of trend data     | Monthly           |
| Planned projects       | Project management, app teams | Quarterly         |
| Hardware refresh cycle | Procurement, vendor EOL dates | Annual            |
| HA/DR reserves         | Cluster configuration         | Per change        |

## What-If Analysis

### Scenario: Host Failure

```
Current state: 6 hosts, 72% memory utilized
Question: What happens if a host fails?

Remaining capacity: 5/6 × usable = 83% of original
New utilization: 72% × (6/5) = 86%

Result: Operational but tight. Another failure pushes to 104% -- over capacity.
```

### Scenario: New Project Deployment

```
New project: 20 VMs, 8 vCPU / 32GB each
Total new demand: 160 vCPU, 640 GB memory

Current free CPU: 200 GHz (~83 cores at 2.4 GHz) → 160 vCPU fits
Current free memory: 342 GB → 640 GB DOES NOT FIT

Result: Need more memory before deployment. Options:
  a) Add hosts (2 × 512GB = 1,024 GB headroom)
  b) Right-size existing VMs to free 300+ GB
  c) Increase overcommit ratio (risky for production)
```

### Scenario: Hardware Refresh

```
Current: 6 × older hosts (256GB RAM each = 1,536 GB raw)
Proposed: 4 × new hosts (512GB RAM each = 2,048 GB raw)

Current usable (N+1): 1,536 × (5/6) = 1,280 GB
New usable (N+1): 2,048 × (3/4) = 1,536 GB

Result: 4 new hosts provide 20% more usable capacity than 6 old hosts,
        with fewer hosts to manage and better per-core performance.
```

## Procurement Planning

### Lead Time Awareness

Hardware procurement is not instant:

| Phase                           | Typical Duration                                             |
| ------------------------------- | ------------------------------------------------------------ |
| Budget approval                 | 2-8 weeks                                                    |
| Vendor quoting and ordering     | 1-2 weeks                                                    |
| Manufacturing and shipping      | 4-12 weeks (longer for custom configs or supply constraints) |
| Racking, cabling, burn-in       | 1-2 weeks                                                    |
| ESXi install, config, vSAN join | 1-3 days                                                     |

**Total: 2-6 months from "we need capacity" to "capacity is available."**

This means your capacity model needs to project at least 6 months ahead. If you detect you'll run out in 3 months, you're already too late for a normal procurement cycle.

### Procurement Triggers

| Trigger                          | Action                                                             |
| -------------------------------- | ------------------------------------------------------------------ |
| Projected to hit 70% in 6 months | Start budget conversation                                          |
| Projected to hit 80% in 3 months | Initiate purchase order                                            |
| Currently at 80%+                | Emergency procurement, right-size aggressively, defer new projects |
| Currently at 90%+                | Stop deploying new workloads, escalate                             |

### Buy vs Extend

| Option                        | When It Makes Sense                                                                 |
| ----------------------------- | ----------------------------------------------------------------------------------- |
| Add hosts to existing cluster | Cluster not at max size, hosts still available for purchase                         |
| New cluster                   | Current cluster at desired max, need workload isolation, hardware generation change |
| Cloud burst                   | Temporary capacity need, seasonal spike, time-to-capacity is critical               |
| Right-size and reclaim        | Fastest, cheapest. Always do this first before buying.                              |

## Capacity Reporting

### Monthly Capacity Report (Minimum)

A useful capacity report fits on one page per cluster:

```
Cluster: Production-01
Report period: January 2026
Hosts: 6 × Dell R760 (2 × 28c, 512GB, 8 × 1.92TB NVMe)

Resource       Usable    Consumed    %Used    Trend(/mo)   Exhaustion
-----------    ------    --------    -----    ----------   ----------
CPU (GHz)       616       340        55%      +15/mo       18 months
Memory (GB)    2,542     2,200       87%      +100/mo      3.4 months  ← WARNING
Storage (TB)    31.8      18.2       57%      +1.2/mo      11 months
VM count         --       142         --      +5/mo           --

Planned additions: ERP project (Month 4, +640GB memory)
Recommendation: Order 2 additional hosts by end of February.
```

### Alerting Thresholds

| Threshold | Meaning  | Action                                            |
| --------- | -------- | ------------------------------------------------- |
| 60%       | Healthy  | No action. Monitor trends.                        |
| 70%       | Watch    | Begin planning for next capacity addition.        |
| 80%       | Warning  | Initiate procurement. Aggressive right-sizing.    |
| 90%       | Critical | Stop new deployments. Emergency procurement.      |
| 95%+      | Danger   | Performance degradation likely. Immediate action. |

These apply to any dimension (CPU, memory, storage). You're as constrained as your most-utilized resource.

## Tools

| Tool                    | What It Does                                                      | Cost                   |
| ----------------------- | ----------------------------------------------------------------- | ---------------------- |
| vCenter (built-in)      | Basic utilization and trend charts                                | Included               |
| Aria Operations (vROps) | Capacity analytics, what-if, right-sizing recommendations, alerts | Licensed               |
| RVTools                 | Snapshot of VM inventory and configuration (no trending)          | Free                   |
| PowerCLI scripts        | Custom reporting, automated data collection                       | Free (effort required) |
| LiveOptics (Dell)       | Workload profiling for hardware sizing                            | Free                   |
| VMware Sizer            | New deployment sizing calculator                                  | Free (web tool)        |

**Aria Operations** is the most capable option for ongoing capacity management. Its what-if scenarios, automated right-sizing recommendations, and cost analysis justify the license cost in environments with 100+ VMs.

**RVTools** is invaluable for quick inventory snapshots -- export every VM's configuration, disk, and network info to a spreadsheet. It doesn't do trending, but it's the fastest way to answer "what's deployed right now?"

## Common Mistakes

| Mistake                                            | Consequence                                                 | Fix                                                 |
| -------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------- |
| Measuring allocated instead of consumed            | Overestimate utilization, buy hardware you don't need       | Track consumed/active memory, not configured        |
| No growth projection                               | Surprised when capacity runs out                            | Maintain 6-month forward projection                 |
| Ignoring HA reserves in capacity math              | Think you have 100% when 17% is reserved for HA             | Always subtract HA reserve from usable capacity     |
| Right-sizing without owner buy-in                  | Angry application teams after month-end batch fails         | Always validate with application owners             |
| Procurement started too late                       | 3-month lead time + 3-month runway = too late               | Trigger procurement at 6-month projected exhaustion |
| vSAN capacity measured without protection overhead | Think you have 40TB usable when it's actually 20TB (RAID-1) | Always account for FTT multiplier and slack         |
| One capacity number for the whole environment      | Hides per-cluster hotspots                                  | Report capacity per cluster                         |
| Ignoring seasonal patterns                         | Under-provisioned for peak, over-provisioned for trough     | Use 12-month trend data, account for seasonality    |
