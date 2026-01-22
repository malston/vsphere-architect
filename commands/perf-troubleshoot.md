# Performance Troubleshooting

Troubleshoot vSphere performance issues using esxtop and metrics.

Use the vsphere-architect skill to cover:
- esxtop views: c (CPU), m (Memory), n (Network), d/u/v (Disk)
- Critical thresholds: %RDY >5%, %CSTP >3%, DAVG >20ms, etc.
- Diagnostic flow: symptom → scope → resource → metrics → root cause
- Common issues: high ready, co-stop, ballooning, storage latency
- CPU troubleshooting: Ready vs Co-stop interpretation
- Memory troubleshooting: reclamation state identification
- Storage troubleshooting: DAVG vs KAVG vs GAVG analysis
- Log locations and vm-support bundle collection

$ARGUMENTS
