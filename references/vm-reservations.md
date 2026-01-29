# VM-Level Reservations

Guidance for CPU and memory reservations on individual VMs and their HA implications.

## What Reservations Do

| Resource   | Reservation Effect                                       |
| ---------- | -------------------------------------------------------- |
| **Memory** | Guarantees physical RAM; exempt from ballooning/swapping |
| **CPU**    | Guarantees minimum MHz; priority during contention       |

**Key difference from Limits**: Limits cap maximum usage. Reservations guarantee minimum. Neither directly affects HA admission control calculation (only memory reservation affects slot sizing).

## When to Use Reservations

### Memory Reservations: Use For

| Workload                                | Why Reservation Helps                                             |
| --------------------------------------- | ----------------------------------------------------------------- |
| **Databases**                           | Ballooning causes catastrophic performance (buffer pool eviction) |
| **In-memory caches** (Redis, Memcached) | Swapping defeats the purpose                                      |
| **Latency-sensitive apps**              | Can't tolerate memory reclamation delays                          |
| **Oracle/SQL Server licensed by host**  | Often requires reservation for support                            |

### Memory Reservations: Avoid For

| Workload                    | Why Reservation Hurts                           |
| --------------------------- | ----------------------------------------------- |
| **General web/app servers** | Ballooning works fine; wastes capacity          |
| **Dev/test VMs**            | Overcommitment acceptable, reservation wasteful |
| **VDI desktops**            | Memory reclamation is part of efficient VDI     |
| **Bursty workloads**        | Peak reservation wastes off-peak capacity       |

### CPU Reservations: Rarely Needed

CPU reservations are less common because:

- CPU contention is brief (scheduler resolves in milliseconds)
- Shares handle priority during contention
- Reservation inflates HA slot size for marginal benefit

**Use CPU reservation only for:**

- Real-time workloads requiring guaranteed cycles
- License compliance requiring dedicated capacity proof

## Partial vs Full Reservations

**100% reservation** = VM gets dedicated physical resources (no overcommitment for that VM)

**Partial reservation** (e.g., 50%) = Guarantee minimum, allow burst above

### Recommended Approach

| Scenario                        | Reservation Level                         |
| ------------------------------- | ----------------------------------------- |
| Database with 128GB configured  | 80-90% of configured memory               |
| Latency-sensitive app           | 100% (guarantees no reclamation)          |
| Important but flexible workload | 50% (baseline guarantee with flexibility) |
| Everything else                 | 0% (use shares for priority)              |

**Why not always 100%?** Every reserved GB is unavailable for other VMs, even when the reserving VM isn't using it.

## HA Implications of Reservations

### Slot Size Inflation (Host Failures Policy)

```
Slot Memory Size = MAX(largest_vm_memory_reservation, 128MB)
```

**Example impact:**

| Cluster State         | Slot Size | Slots per 256GB Host |
| --------------------- | --------- | -------------------- |
| No reservations       | 128MB     | ~2000                |
| One 8GB reservation   | 8GB       | 32                   |
| One 64GB reservation  | 64GB      | 4                    |
| One 128GB reservation | 128GB     | 2                    |

**The fix**: If you need large reservations, use **Percentage-based admission control** instead of Host Failures policy. See [ha-admission-control.md](ha-admission-control.md).

### Failover Placement Requirements

When HA restarts a VM with reservations, the target host must have:

- Enough unreserved memory to satisfy the reservation
- Enough unreserved CPU (if CPU reservation set)

**Failure mode**: VM fails to restart after host failure because no surviving host has enough unreserved capacity.

**Prevention**: When sizing HA reserves, account for your largest reserved VM landing on any single surviving host.

### Reserved Memory and Reclamation

| Reservation State | Ballooning?                 | Compression?        | Swapping?           |
| ----------------- | --------------------------- | ------------------- | ------------------- |
| 100% reserved     | Never                       | Never               | Never               |
| 50% reserved      | Only for unreserved portion | Only for unreserved | Only for unreserved |
| No reservation    | Yes                         | Yes                 | Yes                 |

This is why DBAs want reservations: ballooning a database's buffer pool causes massive performance degradation as the database must re-read data from disk.

## Reservation vs Shares vs Limits

| Mechanism       | Purpose                    | HA Impact                             |
| --------------- | -------------------------- | ------------------------------------- |
| **Reservation** | Guarantee minimum          | Affects slot size, failover placement |
| **Shares**      | Priority during contention | None                                  |
| **Limit**       | Cap maximum                | None                                  |

**Best practice**: Use shares for priority differentiation, reservations only when you need guaranteed resources.

### Shares for Priority (Preferred)

```
Production VMs: High shares (2000/vCPU, 20 shares/MB)
Dev/Test VMs: Low shares (500/vCPU, 5 shares/MB)
```

During contention, production gets 4x the resources of dev/test without any HA impact.

### Reservations for Guarantee (Use Sparingly)

```
Database VM: 80% memory reservation
Everything else: No reservation
```

Database is guaranteed RAM; other VMs contend for the rest.

## Mixed Workload Strategy

For clusters with both reserved and unreserved VMs:

1. **Use Percentage-based admission control** (avoids slot inflation)

2. **Calculate total reserved capacity**:

   ```
   Total Reserved Memory = Sum of all VM memory reservations
   ```

3. **Ensure failover capacity exceeds largest reserved VM**:

   ```
   Reserve % × Cluster Memory > Largest VM Reservation
   ```

   Otherwise, that VM can't restart after failure.

4. **Consider reservation "tax"**:
   - Reserved memory is unavailable for other VMs even when idle
   - 10 VMs with 10GB reservation = 100GB permanently consumed

## Common Mistakes

| Mistake                           | Consequence                          | Fix                               |
| --------------------------------- | ------------------------------------ | --------------------------------- |
| 100% reservation on all VMs       | Zero flexibility, wasted capacity    | Reserve only what needs guarantee |
| Large reservation + slot-based HA | Blocked power-ons, capacity waste    | Switch to percentage-based HA     |
| CPU reservation "for performance" | Slot inflation with marginal benefit | Use shares instead                |
| No reservation on database        | Ballooning tanks performance         | Reserve 80-90% of DB memory       |
| Reservation > configured memory   | Invalid configuration                | Reservation must be ≤ configured  |
