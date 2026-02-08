# DRS Rules

Configure DRS, affinity rules, and resource pools for vSphere clusters.

## Reference Files

Read these before answering:

- `references/drs-tuning.md` -- migration thresholds, share calculations, affinity patterns
- `references/cluster-design-patterns.md` -- if cluster-level design context is relevant
- `references/ha-patterns.md` -- if HA interaction with DRS rules comes up
- `references/vm-reservations.md` -- if resource pool reservations or shares are discussed

## Response Structure

1. Understand the placement constraint needed -- keep together, keep apart, pin to hosts, avoid hosts
2. Recommend the specific rule type: VM-VM affinity/anti-affinity, VM-Host affinity/anti-affinity
3. Discuss Required vs Preferred -- Required rules can block HA failover and DRS evacuation
4. Warn about HA interaction: Required anti-affinity with too few hosts can prevent restart
5. Provide specific configuration guidance for the scenario

## Key Concepts

- DRS automation levels: Manual (recommendations only), Partially Automated (initial placement only), Fully Automated
- Migration threshold 1-5: Level 1 = only mandatory moves, Level 5 = all improvements including minor ones
- Required rules are hard constraints -- DRS will not violate them even during maintenance mode
- Preferred rules are soft constraints -- DRS tries to honor them but will violate if necessary
- Resource pools: reservations guarantee minimums, limits cap maximums, shares set relative priority during contention
- Anti-pattern: mixing VMs directly in a cluster with resource pools (orphaned VMs get unfair share allocation)

$ARGUMENTS
