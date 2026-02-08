# vCPU:pCPU Ratio

Explain vCPU:pCPU ratios and CPU oversubscription for vSphere environments.

## Reference Files

Read these before answering:

- `references/cpu-scheduler.md` -- scheduler internals, co-scheduling, HT, latency-sensitive tuning
- `references/numa-vm-design.md` -- if NUMA topology or wide VMs are relevant
- `references/capacity-planning.md` -- if the question involves sizing or growth

## Response Structure

1. Explain the ratio is emergent -- it's the sum of all vCPUs divided by physical cores, not a single setting
2. Describe what influences it: VM vCPU counts, resource pool reservations/limits, scheduler behavior, power management
3. Provide workload-specific ratio recommendations (general 4:1-6:1, CPU-intensive 2:1-3:1, VDI 6:1-8:1, dev/test 8:1-10:1)
4. Include validation metrics: CPU Ready >5% and Co-stop >3% indicate problems
5. Connect to capacity planning -- how the chosen ratio drives host count

## Key Concepts

- CPU Ready = vCPU waiting for a pCPU (host-level oversubscription)
- Co-stop = vCPU waiting for sibling vCPUs to be co-scheduled (VM has too many vCPUs)
- Hyperthreading logical cores count as roughly 0.5-0.75 of a physical core for ratio math
- NUMA boundaries matter for VMs crossing socket boundaries
- The ratio is a planning tool, not a hard limit -- validation metrics tell you if it's working

$ARGUMENTS
