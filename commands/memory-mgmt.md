# Memory Management

Explain ESXi memory management techniques and optimization.

## Reference Files

Read these before answering:

- `references/memory-sizing.md` -- reclamation hierarchy, TPS, ballooning, compression, swap, large pages
- `references/numa-vm-design.md` -- if NUMA memory locality or vNUMA is relevant
- `references/vm-reservations.md` -- if reservations or HA slot interaction comes up
- `references/capacity-planning.md` -- if the question involves memory sizing or growth

## Response Structure

1. Explain the reclamation hierarchy and when each technique activates (TPS > Balloon > Compress > Swap)
2. Clarify that ballooning requires VMware Tools and that TPS is intra-VM only by default
3. Provide the state thresholds: High >6%, Soft 4-6%, Hard 2-4%, Low <2% free
4. Discuss overcommitment honestly -- conservative (no overcommit for production) vs aggressive (1.2:1-1.5:1 with caveats)
5. Reference esxtop memory metrics for diagnosis: MCTLSZ, SWCUR, CACHEUSD, %ACTV

## Key Concepts

- Memory is almost always the first capacity constraint -- clusters run out of RAM before CPU
- Active memory (not consumed) is the right metric for right-sizing -- consumed includes guest OS caches
- Ballooning is cooperative and relatively benign; swapping is not
- Large pages (2MB) improve TLB performance but prevent TPS -- ESXi breaks them down under pressure
- Memory reservations guarantee physical RAM and affect HA slot sizing

$ARGUMENTS
