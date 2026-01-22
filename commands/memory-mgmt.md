# Memory Management

Explain ESXi memory management techniques and optimization.

Use the vsphere-architect skill to cover:
- Reclamation hierarchy: TPS → Ballooning → Compression → Swapping
- Memory state thresholds: High (>6%), Soft (4-6%), Hard (2-4%), Low (<2%)
- Ballooning requirements (VMware Tools) and behavior
- Overcommitment guidelines and when it's acceptable
- Key metrics: MCTLSZ, SWCUR, CACHEUSD, %ACTV

$ARGUMENTS
