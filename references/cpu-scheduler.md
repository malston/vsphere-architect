# CPU Scheduler Deep Dive

## VMkernel Scheduler Architecture

The VMkernel CPU scheduler uses a proportional-share algorithm with multiple scheduling contexts:

### Scheduling Contexts

1. **World**: Basic schedulable entity (VM vCPU, VMkernel thread, etc.)
2. **Resource Pool**: Group of worlds sharing resources
3. **Cell**: NUMA node-aware scheduling domain

### Scheduler Algorithm

The scheduler uses **Borrowed Virtual Time (BVT)** algorithm:

```
Effective Virtual Time = Actual Virtual Time - Warp (based on shares/reservations)
```

Worlds with lower effective virtual time get scheduled first.

### Co-scheduling (Relaxed Co-scheduling)

For SMP VMs, the scheduler attempts to run sibling vCPUs together but allows "skew":

- **Skew threshold**: Maximum time difference between sibling vCPUs
- **Co-stop**: When skew exceeds threshold, faster vCPUs are halted
- Default skew threshold: ~4ms

#### What Is Skew?

Skew occurs when the vCPUs of a single VM get out of sync—some vCPUs have received more CPU time than others.

Example with a 4-vCPU VM:

1. vCPU 0 and vCPU 1 get scheduled and run for 10ms
2. vCPU 2 and vCPU 3 are waiting (no free pCPUs)
3. vCPU 0 and vCPU 1 are now 10ms "ahead" of vCPU 2 and vCPU 3

#### Why Skew Is a Problem

Inside the guest OS, the kernel assumes all CPUs progress at roughly the same rate. When they don't:

| Problem                     | What Happens                                                                                                    |
| --------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Spinlock breakdown**      | Thread on vCPU 0 grabs lock, vCPU 0 gets descheduled. Thread on vCPU 1 spins waiting, burning cycles on nothing |
| **Timer inconsistencies**   | Guest OS timekeeping gets confused when CPUs report different elapsed times                                     |
| **IPI stalls**              | vCPU 0 sends inter-processor interrupt to vCPU 2, but vCPU 2 isn't scheduled. Cascading delays                  |
| **Application sync issues** | Multi-threaded apps expecting threads to progress together see unpredictable behavior                           |

#### The Co-Stop Tradeoff

The scheduler must choose between two bad options:

| Option                       | Result                                                      |
| ---------------------------- | ----------------------------------------------------------- |
| Let skew happen              | Guest OS internals malfunction, spinlock waste, timer drift |
| Prevent skew (co-scheduling) | vCPUs wait for each other = co-stop overhead                |

VMware chose **relaxed co-scheduling**—allow some skew but keep it bounded. When skew exceeds the threshold, the scheduler stops the "ahead" vCPUs until the "behind" ones catch up.

#### Co-Stop vs CPU Ready

| Metric        | What It Means                          | Root Cause            |
| ------------- | -------------------------------------- | --------------------- |
| **CPU Ready** | vCPU waiting because no pCPU available | Host oversubscribed   |
| **Co-Stop**   | vCPU waiting for sibling vCPUs to sync | VM has too many vCPUs |

You can have low Ready and high Co-stop—the host isn't overloaded, but that VM's vCPU count creates scheduling inefficiency.

#### Co-Stop Thresholds

| %CSTP | Status   | Action                  |
| ----- | -------- | ----------------------- |
| < 3%  | Normal   | None needed             |
| 3-5%  | Warning  | Consider reducing vCPUs |
| > 5%  | Critical | Reduce vCPU count       |

#### The Fix: Right-Size vCPUs

More vCPUs ≠ better performance. Giving a VM more vCPUs than its workload uses creates scheduling overhead.

Right-sizing approach:

1. Check actual CPU utilization inside the guest
2. If a 4-vCPU VM averages 25% utilization, it's only using ~1 vCPU worth of work
3. Reduce to 2 vCPUs—performance often _improves_ because scheduling becomes easier

**Implications:**

- VMs with many vCPUs relative to workload waste cycles in co-stop
- Right-size vCPUs to actual parallel thread count needed

### NUMA Scheduling

ESXi is NUMA-aware and tries to:

1. Keep VM memory and vCPUs on same NUMA node
2. Migrate VMs to less loaded NUMA nodes
3. Respect NUMA home node for memory allocation

**NUMA-related settings:**

```
Numa.PreferHT = 1  # Prefer hyperthreads on same NUMA node
Numa.RebalanceEnable = 1  # Enable NUMA rebalancing
Numa.RebalancePeriod = 2000  # Rebalance check interval (ms)
```

### CPU Shares Calculation

During contention, CPU time is allocated proportionally to shares:

| Level  | Shares per vCPU |
| ------ | --------------- |
| Low    | 500             |
| Normal | 1000            |
| High   | 2000            |

**Example**: Two 4-vCPU VMs, one High (8000 total), one Low (2000 total)

- High VM gets: 8000/(8000+2000) = 80%
- Low VM gets: 2000/(8000+2000) = 20%

### Reservations and Limits

**Reservations:**

- Guarantee minimum MHz
- Scheduler won't start VMs if reservation can't be met
- Admission control considers reservations

**Limits:**

- Cap maximum MHz
- VM is throttled when hitting limit
- Shows as %MLMTD in esxtop

### Hyperthreading Considerations

With HT enabled:

- Each logical processor (HT) is a schedulable pCPU
- Core co-scheduling: Scheduler aware of HT pairs
- HT efficiency: ~1.25x single thread, not 2x

**HT sharing modes:**

- None: vCPU gets whole core
- Any: vCPU can share with any other vCPU
- Internal: vCPU only shares with same VM's vCPUs

### Advanced Settings

```
# Scheduler parameters (Advanced Settings)
Cpu.HTWholeCoreThreshold = 50  # Minimum %busy before HT sharing
Cpu.CreditAgePeriod = 3000  # Credit aging period (ms)
Cpu.CreditBorrowMin = 50  # Minimum credits to borrow
```

## Latency-Sensitive Workloads

For VMs requiring consistent low latency:

1. **Latency sensitivity = High**
   - Exclusive pCPU scheduling
   - No HT sharing
   - Memory pre-reservation

2. **CPU affinity** (use sparingly)
   - Pins vCPUs to specific pCPUs
   - Breaks DRS, complicates HA
   - Only for extreme requirements

3. **Power management = High Performance**
   - Disables CPU frequency scaling
   - All cores at max frequency

## Monitoring Scheduler Health

### Key esxtop Metrics

| Metric | Description               | Target            |
| ------ | ------------------------- | ----------------- |
| %RDY   | Wait time for pCPU        | <5%               |
| %CSTP  | Co-stop time              | <3%               |
| %MLMTD | Limited by resource limit | 0%                |
| %SWPWT | Waiting on swapped memory | 0%                |
| %WAIT  | I/O and idle wait         | Varies            |
| %USED  | Actual CPU consumed       | Context dependent |

### Calculating Real CPU Demand

```
True Demand = %USED + %RDY + %CSTP
```

If True Demand >> %USED, the VM is CPU-constrained.

### Batch Mode esxtop

For historical analysis:

```bash
esxtop -b -d 5 -n 100 > esxtop.csv
```

- `-b`: Batch mode
- `-d 5`: 5-second interval
- `-n 100`: 100 iterations
