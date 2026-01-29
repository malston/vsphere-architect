# Spinlocks

## What Is a Spinlock?

A spinlock is a synchronization mechanism where a thread **actively waits in a loop** ("spins") checking if a lock is available, rather than going to sleep and being woken up later.

```
Thread A: Acquire lock → do work → release lock
Thread B: Try lock → BUSY → spin spin spin → try again → got it!
```

## Why "Spin"?

The thread literally runs a tight loop:

```c
while (lock->held == true) {
    // do nothing, just keep checking
}
lock->held = true;  // got it!
```

It's burning CPU cycles doing nothing useful—just waiting.

## Why Use Spinlocks Instead of Sleep?

Spinlocks are used when:

1. **The lock is held for very short periods** (microseconds)
2. **The cost of sleeping/waking is higher than spinning**

Going to sleep involves:

- Context switch (save registers, switch stacks)
- Scheduler involvement
- Wake-up latency when lock releases

For very short critical sections, spinning is cheaper than all that overhead.

## Where Spinlocks Are Used

- **OS kernels** — Protecting data structures accessed by interrupt handlers
- **Multi-threaded applications** — Short critical sections
- **Device drivers** — Hardware register access
- **Database engines** — Latch management

## The vSphere Problem

This is why spinlocks matter for co-stops:

1. **Thread on vCPU 0** acquires a spinlock
2. **vCPU 0 gets descheduled** (hypervisor takes it off a pCPU)
3. **Thread on vCPU 1** tries to acquire the same lock
4. **vCPU 1 spins**... and spins... and spins...
5. vCPU 0 isn't running, so it can't release the lock
6. vCPU 1 burns its entire time slice spinning on nothing

This is called **Lock Holder Preemption (LHP)** — the thread holding the lock got preempted, and everyone waiting for it wastes CPU.

## Why This Is Worse in VMs

On bare metal:

- If a thread holds a spinlock, it's actually running
- Spinners wait briefly, then get the lock

In a VM with skew:

- Lock holder's vCPU might not be scheduled
- Spinners burn entire time slices waiting
- The more vCPUs, the more potential spinners wasting cycles

## The Fix

This is one reason why **reducing vCPU count helps performance**:

- Fewer vCPUs = easier to schedule together = less skew
- Less skew = lock holders stay scheduled = spinners wait less
- Result: Less wasted CPU, better throughput

## Key Insight

Spinlocks assume the lock holder is making progress. In a VM with scheduling skew, that assumption breaks down—the lock holder might not be running at all, and spinners waste enormous amounts of CPU waiting for something that isn't happening.
