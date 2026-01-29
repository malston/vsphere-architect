# HA Admission Control Reserves

Practical guidance for setting cluster-level HA reserves. For mechanics and formulas, see [ha-patterns.md](ha-patterns.md).

## Policy Selection Decision Framework

```
Do you have VMs with large memory/CPU reservations?
├── Yes → Do ALL VMs have similar reservation sizes?
│   ├── Yes → Host Failures policy works fine
│   └── No (mixed) → Use Percentage-based policy
└── No reservations → Either policy works; Host Failures is simpler
```

**Key insight**: Host Failures policy uses slot sizing based on largest reservation. One 128GB reserved VM among 4GB VMs makes every slot 128GB, wasting massive capacity.

### Policy Comparison

| Scenario                               | Recommended Policy | Why                                          |
| -------------------------------------- | ------------------ | -------------------------------------------- |
| Homogeneous VMs, no reservations       | Host Failures      | Simple, predictable                          |
| Mixed VM sizes, no reservations        | Either             | Host Failures uses 32MHz/128MB default slots |
| One or few VMs with large reservations | **Percentage**     | Avoids slot inflation                        |
| All VMs have similar reservations      | Host Failures      | Slots match actual VM needs                  |
| Compliance requires dedicated standby  | Dedicated Hosts    | Regulatory/audit requirement                 |
| Heterogeneous host hardware            | Percentage         | Consistent capacity math across hosts        |

## Calculating Percentage Reserves

**Formula:**

```
Reserve % = (Failures to Tolerate / Host Count) × 100
```

| Hosts | N+1 (1 failure) | N+2 (2 failures) |
| ----- | --------------- | ---------------- |
| 3     | 33%             | 67%              |
| 4     | 25%             | 50%              |
| 5     | 20%             | 40%              |
| 6     | 17%             | 33%              |
| 8     | 13%             | 25%              |

**Set CPU and memory percentages independently** if your workloads are memory-heavy or CPU-heavy:

- Memory-bound workloads (databases): May need higher memory reserve
- CPU-bound workloads (analytics): May need higher CPU reserve
- Balanced workloads: Use same percentage for both

## Troubleshooting: Admission Control Blocking Power-Ons

**Symptom**: "Insufficient resources to satisfy HA failover" despite apparent capacity

### Diagnostic Steps

1. **Check current admission control state**:

   ```
   vCenter > Cluster > Monitor > vSphere HA > Summary
   ```

   Look for: Current failover capacity, Configured failover capacity

2. **If using Host Failures policy, check slot size**:

   ```
   vCenter > Cluster > Monitor > vSphere HA > Slot sizes
   ```

   - Compare slot size to your typical VM size
   - If slot >> average VM, you have slot inflation

3. **Find the culprit reservation**:
   - Sort VMs by memory reservation (Cluster > VMs tab)
   - The largest reservation sets the slot size

### Common Causes and Fixes

| Cause                                  | Diagnostic                                  | Fix                                                |
| -------------------------------------- | ------------------------------------------- | -------------------------------------------------- |
| Slot inflation from large reservation  | Slot size >> average VM                     | Switch to Percentage policy, or reduce reservation |
| Required DRS rules limiting host usage | DRS shows rule violations                   | Change rules from Required to Preferred            |
| Host in maintenance/disconnected       | Host status shows degraded                  | Return host to service or adjust failover level    |
| Percentage set too high                | Reserve > actual need                       | Recalculate percentage for current host count      |
| Memory vs CPU mismatch                 | CPU reserve met, memory not (or vice versa) | Align percentages with workload characteristics    |

### Slot Size Override

If you must use Host Failures policy with mixed reservations, you can manually set slot size:

```
vCenter > Cluster > Configure > vSphere HA > Admission Control
Edit > Slot Policy > Fixed slot size
```

**Set values slightly above your largest "typical" VM** (excluding outliers):

- Example: If most VMs are 4-16GB but one database is 128GB, set slot to 20GB
- The 128GB VM will consume multiple slots (ceiling(128/20) = 7 slots)

**Warning**: Manual slot sizing requires maintenance as VM sizes change.

## Admission Control Override

**Temporarily disabling admission control** (Advanced Settings):

```
das.ignoreInsufficientResourcesForHa = true
```

**When acceptable:**

- Emergency VM power-on during planned maintenance window
- Temporary capacity shortage with imminent host addition

**When NOT acceptable:**

- Ongoing production operation (defeats HA purpose)
- "We'll fix it later" (you won't)

**Better alternative**: Power off low-priority VMs to free capacity rather than disabling admission control.

## Interaction with VM Reservations

Admission control reserves capacity at the cluster level. VM reservations guarantee resources at the VM level. They interact:

1. **VM reservation → slot size** (Host Failures policy)
2. **VM reservation → failover placement**: Reserved VM must land on host with enough unreserved capacity
3. **Total reservations → admission control**: Sum of all reservations counts against cluster capacity

See [vm-reservations.md](vm-reservations.md) for VM-level reservation guidance.
