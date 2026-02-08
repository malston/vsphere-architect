# Performance Troubleshooting

Troubleshoot vSphere performance issues using esxtop and metrics.

## Reference Files

Read these before answering:

- `references/troubleshooting.md` -- esxtop interpretation, diagnostic workflows, log analysis
- `references/monitoring-and-observability.md` -- esxtop field reference, Aria Operations dashboards, alerting
- `references/cpu-scheduler.md` -- if CPU scheduling issues (Ready, Co-stop, NUMA) are involved
- `references/memory-sizing.md` -- if memory reclamation or pressure is involved
- `references/storage-patterns.md` -- if storage latency (DAVG, KAVG, GAVG) is involved

## Response Structure

1. Identify the symptom and scope: which VMs, which hosts, which resource type
2. Reference the specific esxtop view and key columns for that resource type
3. Compare observed values against critical thresholds
4. Walk through the diagnostic flow: symptom > scope > resource > metrics > root cause
5. Recommend specific actions based on the root cause

## Key Concepts

- CPU: %RDY >5% = host oversubscribed, %CSTP >3% = VM has too many vCPUs, %MLMTD >0% = resource limit hit
- Memory: MCTLSZ >0 = ballooning active, SWCUR >0 = swapping (severe), %ACTV vs consumed shows true demand
- Storage: DAVG = array latency, KAVG = host latency, GAVG = DAVG + KAVG (guest-observed). KAVG >> DAVG = host-side issue
- Network: %DRPTX/%DRPRX >0.1% = drops, check ring buffer, NIC saturation, or traffic shaping
- High Ready + Low Co-stop = host oversubscribed (add hosts or reduce total vCPUs)
- High Co-stop + Normal Ready = VM oversized (reduce that VM's vCPU count)
- High Ready + High MLMTD = resource pool limit hit (raise limit or reservation)
- Always check esxtop in batch mode for trending, not just point-in-time snapshots

$ARGUMENTS
