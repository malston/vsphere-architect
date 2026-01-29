# Co-Stops and Skew

## What Is Co-Stop?

Co-stop is CPU scheduler wait time that occurs when a vCPU must pause and wait for its sibling vCPUs to be scheduled simultaneously on physical cores.

## Why Co-Stops Happen

The VMkernel CPU scheduler enforces **relaxed co-scheduling** for SMP (multi-vCPU) VMs. When a VM has multiple vCPUs, the scheduler tries to run them reasonably close together in time to prevent skew.

The problem:

1. VM has 8 vCPUs
2. Host has available pCPUs, but only 5 are free right now
3. Scheduler could run 5 vCPUs, but those 5 would race ahead of the other 3
4. To prevent excessive skew, the scheduler **stops** some vCPUs to let others catch up

This wait time is co-stop.

## What Is Skew?

Skew is when the vCPUs of a single VM get out of sync—some have received more CPU time than others.

Example with a 4-vCPU VM:

- vCPU 0 gets scheduled and runs for 10ms
- vCPU 1 gets scheduled and runs for 10ms
- vCPU 2 and vCPU 3 are waiting (no free pCPUs)
- Now vCPU 0 and vCPU 1 are 10ms "ahead" of vCPU 2 and vCPU 3

## Why Skew Is Bad

Inside the guest OS, the kernel assumes all CPUs progress at roughly the same rate. When they don't:

| Problem                     | What Happens                                                                                                    |
| --------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Spinlock breakdown**      | Thread on vCPU 0 grabs lock, vCPU 0 gets descheduled. Thread on vCPU 1 spins waiting, burning cycles on nothing |
| **Timer inconsistencies**   | Guest OS timekeeping gets confused when CPUs report different elapsed times                                     |
| **IPI stalls**              | vCPU 0 sends inter-processor interrupt to vCPU 2, but vCPU 2 isn't scheduled. Cascading delays                  |
| **Application sync issues** | Multi-threaded apps expecting threads to progress together see unpredictable behavior                           |

## The Tradeoff

The scheduler has two bad options:

| Option                       | Result                                                      |
| ---------------------------- | ----------------------------------------------------------- |
| Let skew happen              | Guest OS internals malfunction, spinlock waste, timer drift |
| Prevent skew (co-scheduling) | vCPUs wait for each other = co-stop                         |

VMware chose **relaxed co-scheduling**—allow _some_ skew but keep it bounded. When skew exceeds the threshold (~4ms), the scheduler stops the "ahead" vCPUs until the "behind" ones catch up.

## Co-Stop vs CPU Ready

| Metric        | What It Means                          | Root Cause            |
| ------------- | -------------------------------------- | --------------------- |
| **CPU Ready** | vCPU waiting because no pCPU available | Host oversubscribed   |
| **Co-Stop**   | vCPU waiting for sibling vCPUs to sync | VM has too many vCPUs |

You can have low Ready and high Co-stop—the host isn't overloaded, but that VM's vCPU count creates scheduling inefficiency.

## Thresholds

| %CSTP | Status   | Action                  |
| ----- | -------- | ----------------------- |
| < 3%  | Normal   | None needed             |
| 3-5%  | Warning  | Consider reducing vCPUs |
| > 5%  | Critical | Reduce vCPU count       |

## How to Check

```bash
esxtop → press 'c' for CPU view
```

Look at the **%CSTP** column for each VM.

## The Fix

**Reduce the VM's vCPU count.**

This is counterintuitive -- people assume more vCPUs = better performance—but giving a VM more vCPUs than its workload actually uses creates scheduling overhead that hurts performance.

**Right-sizing approach:**

1. Check actual CPU utilization inside the guest
2. If a 4-vCPU VM averages 25% utilization, it's only using ~1 vCPU worth of work
3. Reduce to 2 vCPUs—performance often _improves_ because scheduling becomes easier

## Key Insight

More vCPUs per VM = harder to schedule all together = more co-stop

Co-stop indicates the VM has more vCPUs than the workload or host can efficiently use. It's not about total host capacity—it's about scheduling complexity for that specific VM.
