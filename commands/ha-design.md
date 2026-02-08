# HA Design

Design vSphere High Availability and Fault Tolerance architectures.

## Reference Files

Read these before answering:

- `references/ha-patterns.md` -- slot calculation, isolation detection, FT requirements
- `references/ha-admission-control.md` -- host failures vs percentage policy, slot inflation, troubleshooting
- `references/vm-reservations.md` -- how reservations affect slot sizing and HA capacity
- `references/cluster-design-patterns.md` -- if N+1/N+2 cluster design is relevant
- `references/backup-and-dr.md` -- if DR-level availability (not just HA) is in scope

## Response Structure

1. Clarify the availability requirement -- HA restart (seconds of downtime) vs FT (zero downtime) vs DR (site failure)
2. Recommend admission control policy: Host Failures for predictable sizing, Percentage for flexibility
3. Explain slot sizing and how reservations inflate slots -- one large reservation wastes capacity cluster-wide
4. Cover failure detection: network heartbeat, datastore heartbeat, isolation response (Disabled/Shutdown/Power Off)
5. Discuss APD/PDL storage failure handling if relevant
6. Address FT requirements and tradeoffs if zero downtime is needed

## Key Concepts

- Slot size = largest VM reservation in the cluster (or 32MHz CPU / 128MB memory if no reservations)
- One oversized reservation inflates all slots -- use percentage-based admission control to avoid this
- Percentage policy: reserve 1/N of cluster resources where N = host count (e.g., 25% for 4-host N+1)
- Required anti-affinity rules can prevent HA from restarting VMs if insufficient hosts remain
- FT requires thick eager-zeroed disks, 10GbE FT logging, same CPU family, max 8 vCPUs
- Network isolation: host loses all heartbeats but is still running -- isolation response determines VM fate

$ARGUMENTS
