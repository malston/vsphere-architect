# vCPU:pCPU Ratio

Explain vCPU:pCPU ratios and CPU oversubscription for vSphere environments.

Use the vsphere-architect skill to provide:
- How the ratio emerges from VM settings, resource pools, and scheduler behavior
- Recommended ratios by workload type (general: 4:1-6:1, CPU-intensive: 2:1-3:1, VDI: 6:1-8:1)
- Validation metrics: CPU Ready >5% and Co-stop >3% indicate problems
- How reservations, limits, and shares affect scheduling during contention

$ARGUMENTS
