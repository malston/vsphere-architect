# Storage Design

Provide storage architecture guidance for vSphere environments.

## Reference Files

Read these before answering:

- `references/storage-patterns.md` -- datastore types, VMDK provisioning, multipathing PSPs
- `references/vsan-architecture.md` -- OSA vs ESA, fault domains, storage policies, object model, rebuilds
- `references/capacity-planning.md` -- if storage capacity or IOPS sizing is relevant
- `references/monitoring-and-observability.md` -- if storage performance metrics are relevant

## Response Structure

1. Clarify requirements first: performance (IOPS, latency), capacity, features (replication, encryption), operational model
2. Compare relevant datastore options (VMFS 6 vs NFS v3/v4.1 vs vSAN vs vVols) for the specific use case
3. Discuss VMDK types and their tradeoffs (thick eager zeroed for FT/MSCS, thin for capacity)
4. Cover vSAN storage policies if applicable -- FTT, RAID-1 vs RAID-5/6, stripe width, capacity math
5. Address multipathing for SAN: Round Robin vs Fixed vs MRU, when each applies

## Key Concepts

- vSAN capacity math must account for FTT protection multiplier and 25-30% slack for rebuilds
- DAVG = device (array) latency, KAVG = kernel (host) latency, GAVG = guest-observed total
- KAVG >> DAVG means a host-side bottleneck (HBA, queue depth, multipathing)
- Thin provisioning saves capacity but has first-write penalty and requires monitoring to prevent overrun
- vSAN ESA eliminates the separate cache tier and supports RAID-5 erasure coding at smaller scale

$ARGUMENTS
