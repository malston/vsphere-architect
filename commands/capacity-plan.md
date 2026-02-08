# Capacity Planning

Provide vSphere capacity planning, sizing, and growth guidance.

## Reference Files

Read these before answering:

- `references/capacity-planning.md` -- usable vs raw capacity formulas, right-sizing, growth modeling, procurement triggers
- `references/cpu-scheduler.md` -- if CPU ratio guidance is needed for sizing
- `references/memory-sizing.md` -- if memory sizing or overcommitment is relevant
- `references/vsan-architecture.md` -- if vSAN capacity math (FTT, slack) is involved
- `references/cluster-design-patterns.md` -- if cluster sizing or host count decisions come up

## Response Structure

1. Identify the capacity dimensions: CPU, memory, storage, network -- any one can be the constraint
2. Calculate usable capacity: raw minus ESXi overhead minus HA reserves
3. Apply workload-specific ratios for CPU (4:1-6:1 general, 2:1-3:1 database, etc.)
4. Size memory conservatively -- it's almost always the first constraint
5. Include growth buffer (20-30%) and project known step-function additions
6. Provide procurement trigger thresholds: 70% = plan, 80% = buy, 90% = stop deploying

## Key Concepts

- Usable capacity = Raw - ESXi overhead (~5-10% CPU, ~2-4GB + per-VM overhead memory) - HA reserve (1/N hosts)
- Memory is almost always the first constraint -- CPU overcommit is efficient, memory overcommit is not
- vSAN usable = (raw - overhead) / FTT multiplier \* (1 - slack), where slack = 25-30%
- Right-sizing is the highest-leverage capacity activity -- oversized VMs waste resources
- Active memory (not consumed) is the right metric for right-sizing
- Procurement lead time is 2-6 months -- if you'll hit 80% in 3 months, you're already late
- Report capacity per cluster, not as a single environment number

$ARGUMENTS
