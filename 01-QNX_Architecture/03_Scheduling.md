# Complete QNX Scheduling — Full Explanation with Examples
# Scheduling

## 1. Thread States

The scheduler only cares about **threads**, not processes. Every thread is in one of these states:

```
                    ┌──────────┐
                    │  THREAD  │
                    └────┬─────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     ┌─────────┐   ┌──────────┐   ┌──────┐
     │ BLOCKED │   │ RUNNABLE │   │ DEAD │
     │(ignored │   │          │   │(gone)│
     │by sched)│   │  ┌────┐  │   └──────┘
     └─────────┘   │  │RUN │  │
                   │  └────┘  │
                   │  ┌─────┐ │
                   │  │READY│ │
                   │  └─────┘ │
                   └──────────┘
```

### Blocked
Thread is **waiting** for something. The scheduler **completely ignores** it.

**Examples of blocked states:**
- **Send-blocked**: called `MsgSend()`, waiting to deliver message
- **Reply-blocked**: sent a message, waiting for the reply
- **Receive-blocked**: called `MsgReceive()`, waiting for a message to arrive
- **Mutex-blocked**: trying to lock a mutex someone else holds
- **Semaphore-blocked**: waiting on a semaphore
- **Condvar-blocked**: waiting on a condition variable
- **Stopped**: another thread externally stopped it

### Running
Thread is **actively executing** on a specific core. **One running thread per core.**

### Ready
Thread **wants to run** but another thread got chosen instead. Could be **many** ready threads.

### Dead
Thread has **terminated** but hasn't been fully cleaned up. Can **never** run again.

### Example:
```
System with 2 cores, 6 threads:

T1 (priority 20) → RUNNING on Core 0
T2 (priority 18) → RUNNING on Core 1
T3 (priority 15) → READY (wants to run, but both cores occupied by higher priority)
T4 (priority 12) → READY
T5 (priority 20) → BLOCKED (waiting for a message via MsgReceive)
T6              → DEAD (terminated, being cleaned up)

Scheduler sees: T1, T2, T3, T4 (ignores T5 and T6)
```

---

## 2. Priority System (0–255)

```
Priority 255 ─── Reserved: IPI interrupt service threads (kernel)
Priority 254 ─── Reserved: Timer/clock threads + user interrupt service threads (kernel)
     ...
Priority 253 ┐
     ...      │  USER SPACE: Your threads live here
Priority   1 ┘   Higher number = Higher priority
Priority   0 ─── Reserved: Idle thread (one per core, runs when nothing else can)
```

### Key Rules:
- **Processes do NOT have priorities — only threads do**
- When you say "launch process at priority 20," you mean "launch the **first thread** in that process at priority 20"
- The scheduler doesn't care what process a thread belongs to

### Example:
```
Process A:
  Thread 1 → priority 20
  Thread 2 → priority 15

Process B:
  Thread 3 → priority 18

Scheduler sees them as:
  Thread 1 (pri 20) > Thread 3 (pri 18) > Thread 2 (pri 15)
  
It does NOT say "Thread 1 and Thread 2 are in same process, let them share."
```

---

## 3. Preemptive Scheduling (The Core Rule)

**Higher priority thread ALWAYS runs instead of lower priority thread. Period.**

- **NOT fair share**: We don't say "priority 10 gets 50%, priority 8 gets 40%"
- **NOT table-driven**: No pre-built schedule
- **NOT deadline-driven**: No deadlines considered
- **It's 100% preemptive**: Priority 11 beats priority 10, 100% of the time

### Example:
```
Core 0 is running Thread A (priority 10).
Thread B (priority 11) becomes ready.

→ Thread B IMMEDIATELY preempts Thread A.
→ Thread A goes to READY state.
→ Thread B runs on Core 0.

Thread A gets 0% CPU while Thread B wants to run.
Not 90/10, not 50/50... it's 100/0.
```

---

## 4. How Threads Share CPU (By Blocking!)

Since higher priority always wins, how do lower priority threads ever run? **By design: threads block when they have no work.**

### The Standard Thread Pattern:
```c
while (1) {
    // BLOCK: Wait for work (message, timer, interrupt, etc.)
    msg = MsgReceive(channel, &buffer, sizeof(buffer), NULL);
    
    // UNBLOCK: Work arrived! Do it.
    handle_message(msg, &buffer);
    
    // Loop back → block again → let other threads run
}
```

### Example Timeline:
```
Thread A (priority 20): handles sensor data
Thread B (priority 10): does background logging

Time 0ms:   A is BLOCKED (waiting for sensor interrupt)
             B is RUNNING (doing logging work)     ← B gets CPU because A is blocked!

Time 5ms:   Sensor interrupt fires → A becomes READY
             A (pri 20) > B (pri 10) → A PREEMPTS B
             A is RUNNING, B is READY

Time 6ms:   A finishes handling sensor data → A BLOCKS again (waits for next interrupt)
             B becomes RUNNING again         ← B gets CPU back!

Time 11ms:  Sensor interrupt fires again → cycle repeats
```

**This is how QNX shares CPU.** Not by dividing time equally, but by **blocking when idle**.

---

## 5. Multicore & SMP

Cores are independent processors but **share**: RAM, system bus, devices, network, disks.

**SMP (Symmetric Multiprocessing)**: All cores treated as identical and interchangeable.

> QNX **defaults** to treating all cores as SMP, even when they're not truly identical.

---

## 6. Why Cores Aren't Always Equal

### a) Cache Clustering
```
Cores 0,1,2,3 share L3 Cache A    |    Cores 4,5,6,7 share L3 Cache B
                                    
Thread on Core 1 ↔ Core 2: FAST (same cache)
Thread on Core 1 ↔ Core 6: SLOW (different cache)
```

### b) big.LITTLE
```
Cores 0–3: 3.0 GHz (fast, power-hungry)
Cores 4–7: 1.5 GHz (slow, power-efficient)
```

### c) Core-Specific Hardware
Some cores have special registers (crypto, performance counters) others don't.

**Solution**: **Thread Affinity** — bind threads to specific cores/clusters.

---

## 7. Cluster-Based Scheduling (QNX's Unique Solution)

### The Problem with Other Approaches:

| Approach | Problem |
|----------|---------|
| **Global ready queue** (one list for all cores) | Worst-case: must search through ALL threads to find one that can run on a specific core. Gets slower as system gets busier. **Bad for real-time.** |
| **Per-core ready queue** (each core has its own list) | Doesn't distribute load well. Hard to move threads between cores. |

### QNX Solution: Clusters

A **cluster** = a named group of related CPU cores.

**Every system has at minimum:**
1. **Global cluster** (all cores) — fallback to SMP behavior
2. **Per-core clusters** (one for each individual core) — for idle threads, IPI threads, timer threads

### Example on an 8-core system:
```
ALWAYS exist:
  Global cluster:   {0,1,2,3,4,5,6,7}   ← all cores
  Core 0 cluster:   {0}
  Core 1 cluster:   {1}
  Core 2 cluster:   {2}
  ... and so on for each core

CUSTOM (defined in startup/BSP):
  cluster0:          {0,1,2}             ← 0x7 = binary 0111
  cluster1:          {0,3}               ← 0x9 = binary 1001
```

### Configuration via startup:
```bash
startup-boardname -c cluster0:0x7,cluster1:0x9
```

### Cluster Rules:
- Each cluster must be **unique** (no two clusters with same core list)
- Each core can belong to **maximum 8 clusters**
- Starts with 2 (global + per-core), so **6 more** custom clusters max
- Configuration is **fixed at boot time** → scheduling cost is **constant and predictable** (critical for real-time!)

### View clusters:
```bash
pidin syspage=cluster    # Shows custom clusters (not the default ones)
```

---

## 8. How Scheduling Decisions Work

### Every thread belongs to ONE cluster.

**Ready threads** sit in their **cluster's ready queue**, ordered by:
1. **Priority** (higher first)
2. **Timestamp** (lower/earlier first — who's been waiting longer)

The timestamp uses a hardware clock cycle counter (like `RDTSC` on x86) **synchronized across all cores**.

---

### Case 1: Thread Leaves Running State (Easy Case)

A thread **blocks itself** (calls MsgReceive, mutex lock, etc.) or gets **stopped via IPI**.

```
BEFORE:
  Core 1 running Thread A (priority 20)
  
  Cluster X ready queue:
    T5 (priority 18, timestamp 1000)
    T3 (priority 15, timestamp 900)
    T7 (priority 15, timestamp 1100)

AFTER Thread A blocks:
  Core 1 needs a new thread.
  Core 1 checks ALL clusters it belongs to.
  Finds T5 (priority 18) is highest priority across all clusters.
  Core 1 now runs T5.
  
  (If no ready thread existed, idle thread would be chosen — it's GUARANTEED 
   to exist in the per-core cluster)
```

**If two clusters have the same highest priority?** → Pick the one with **lower timestamp** (waited longer).

**If new thread is in a different process?** → Triggers a **context switch** (MMU reconfiguration). But the scheduler doesn't care — it just picks the best thread.

---

### Case 2: Thread Becomes Runnable (More Complex)

Someone posts a semaphore, signals a condition variable, etc. A thread moves from BLOCKED → READY.

**Decision flowchart:**

```
Thread T becomes READY, added to its cluster
           │
           ▼
   Is T the MOST ELIGIBLE
   thread in its cluster?
      (highest priority,          NO ──→ Do nothing. T just waits
       lowest timestamp)                  in the ready queue.
           │
          YES
           │
           ▼
   Can T preempt the thread
   on the CURRENT core?           YES ──→ LOCAL PREEMPTION
   (T's priority > current              (switch threads on this core)
    thread's priority)
           │
           NO
           │
           ▼
   Is there an IDLE core             YES ──→ Send IPI to that idle core
   in T's cluster?                          → T runs on the idle core
   (a core running the idle thread)
           │
           NO
           │
           ▼
   Is there a core in T's cluster    YES ──→ Send IPI to that core
   running a LOWER priority thread?         → Preempt lowest priority thread
   (pick the lowest one)                    → T runs there
           │
           NO
           │
           ▼
   T just WAITS in the ready queue
   until a higher priority thread blocks
   and T gets its turn later.
```

### Full Example:
```
System: 4 cores, Cluster A = {Core 0, Core 1, Core 2, Core 3}

Currently:
  Core 0 → running T1 (priority 25)    ← current core where decision happens
  Core 1 → running T2 (priority 20)
  Core 2 → running idle thread (priority 0)
  Core 3 → running T4 (priority 15)

T1 on Core 0 unlocks a mutex → T8 (priority 22) becomes READY in Cluster A.

Step 1: Is T8 most eligible in Cluster A? YES (priority 22 is highest in ready queue)

Step 2: Can T8 preempt Core 0? 
        Core 0 runs T1 (priority 25). T8 is priority 22.  
        22 < 25 → NO

Step 3: Is there an idle core in Cluster A?
        Core 2 is running idle thread → YES!
        
Step 4: Core 0 sends IPI to Core 2.
        Core 2 receives IPI → preempts idle → runs T8.

Result:
  Core 0 → T1 (priority 25)    ← unchanged
  Core 1 → T2 (priority 20)    ← unchanged  
  Core 2 → T8 (priority 22)    ← was idle, now runs T8 ✅
  Core 3 → T4 (priority 15)    ← unchanged
```

---

## 9. Thread Affinity (Binding Threads to Clusters)

Use `ThreadCtl()` with `_NTO_TCTL_RUNMASK`:

```c
uint64_t runmask = 0x9;  // binary 1001 = Core 0 and Core 3

ThreadCtl(_NTO_TCTL_RUNMASK, (void *)runmask);
```

**The runmask MUST match an existing cluster definition!** If no cluster with exactly {Core 0, Core 3} exists, this call **fails**.

### Example:
```
Startup defined: cluster1 = 0x9 (cores 0 and 3)

ThreadCtl(_NTO_TCTL_RUNMASK, (void *)0x9);  → ✅ SUCCESS (matches cluster1)
ThreadCtl(_NTO_TCTL_RUNMASK, (void *)0xA);  → ❌ FAILS (no cluster for cores 1,3)
```

### Use Cases for Affinity:

| Use Case | Example |
|----------|---------|
| **Performance cores** | Bind heavy computation threads to big cores |
| **Safety segregation** | ASIL threads on cores 4–7, QM threads on cores 0–3 |
| **Hypervisor isolation** | Hypervisor on cores 0–1, apps on cores 2–3 |
| **Cache optimization** | Bind threads sharing data to same L3 cache cluster |
| **Core-specific hardware** | Thread accessing Core 0's performance registers → bind to Core 0 |

---

## 10. Scheduling Algorithms

Each thread has a scheduling algorithm **in addition to** its priority.

The algorithm is **secondary** — it affects **eligibility/ordering** of threads, but the core rule is still **"highest priority wins."**

---

### a) FIFO (First In, First Out) — Simplest

**Rule:** Thread runs until it **blocks itself**. Nothing else happens.

```
Thread A (priority 20, FIFO) starts running.
It will run FOREVER until:
  - It blocks (calls MsgReceive, waits on mutex, etc.)
  - A HIGHER priority thread preempts it

It will NEVER voluntarily give up CPU to same-priority threads.
```

---

### b) Round Robin — Threads Take Turns

**Rule:** Like FIFO, but after a **time slice** (typically 4ms), the thread moves to the **tail** of the ready queue at its priority, letting other same-priority threads run.

### Single Core Example:
```
T_A, T_B, T_C — all priority 15, Round Robin, 1 core

Time 0–4ms:    T_A runs (time slice = 4ms)
Time 4–8ms:    T_B runs
Time 8–12ms:   T_C runs
Time 12–16ms:  T_A runs again
Time 16–20ms:  T_B runs again
...

Pattern: A → B → C → A → B → C → ...
```

### Multi-Core Example (2 cores, 3 threads):
```
T_A, T_B, T_C — all priority 15, Round Robin
Cluster has Core 1 and Core 2.

Time 0ms:     Core 1 picks T_A, Core 2 picks T_B
Time 4ms:     T_A's time slice expires → T_C replaces T_A on Core 1
Time ~4ms:    T_B's time slice expires → T_A replaces T_B on Core 2

Core 1 pattern: A → C → B → A → C → B → ...
Core 2 pattern: B → A → C → B → A → C → ...
```

### Preemption Doesn't Count Against Time Slice!
```
Thread A (priority 15, Round Robin, 4ms time slice)
Thread F (priority 25, FIFO)

Time 0ms:   A starts running
Time 2ms:   F preempts A (higher priority)
Time 4ms:   F blocks → A resumes
Time 6ms:   A's time slice finally expires (only 4ms of ACTUAL running: 0–2ms + 4–6ms)

The 2ms that A was preempted does NOT count against its budget.
```

---

### c) Sporadic — Budget-Based Priority Switching

Has **4 parameters:**

| Parameter | Meaning |
|-----------|---------|
| **Priority (high)** | Normal running priority |
| **Low priority** | Drops to this when budget exhausted |
| **Budget** | How long it can run at high priority |
| **Replenishment period** | When it gets new budget |

### Example:
```
Thread S: priority=20, low_priority=5, budget=3ms, replenishment=10ms

Timeline:
┌──────────────────────────────────────────────────────┐
│ Time 0ms:   S runs at priority 20                    │
│ Time 3ms:   Budget exhausted! S drops to priority 5  │
│             (may or may not run — other threads       │
│              likely have higher priority than 5)      │
│ Time 10ms:  Replenishment! S gets new 3ms budget     │
│             S jumps back to priority 20               │
│ Time 13ms:  Budget exhausted again → drops to 5      │
│ Time 20ms:  Replenishment → back to 20               │
│ ...                                                   │
└──────────────────────────────────────────────────────┘
```

```
Priority
  20 ██████░░░░░░░░██████░░░░░░░░██████
   5       ░░░░░░░       ░░░░░░░       
     0   3ms      10ms 13ms     20ms 23ms
         ▲               ▲              ▲
       budget          replenish     replenish
       runs out        + new budget  + new budget
```

**Again, preempted time does NOT count against the budget.**

### Why use Sporadic?
It lets you **guarantee** a thread gets a certain amount of CPU time within a period, while **limiting** how much it can starve other threads. It's useful for **rate-limiting** CPU-intensive threads.

---

### d) High Priority IST (Special Kernel Algorithm)

- Used only by **in-kernel interrupt service threads** (priority 254–255)
- IPI handlers (priority 255), timer threads (priority 254)
- Behaves like **FIFO** but is allowed to use the reserved priority range
- Not for user threads

---

## 11. Summary & Best Practices

### Scheduling Rules Recap:
```
1. Higher priority ALWAYS preempts lower priority (100%)
2. Preempted thread goes to HEAD of ready list (keeps eligibility)
3. Threads should BLOCK when they have no work → lets others run
4. Ready threads ordered by: Priority → Timestamp
5. Scheduling cost is FIXED (determined by cluster config at boot)
```

### Priority Assignment Guidelines:

```
HIGH PRIORITY (e.g., 200+):
  └─ Safety-critical threads (ASIL-rated)
  └─ Time-critical threads (must respond within microseconds)

MEDIUM PRIORITY (e.g., 50–150):
  └─ Normal application threads

LOW PRIORITY (e.g., 1–30):
  └─ CPU-heavy computation (long algorithms, rendering)
  └─ Background tasks (logging, telemetry upload)

WHY? So time/safety-critical threads can PREEMPT
     the CPU-heavy threads whenever they need to run.
```

### Bad Design (Polling):
```c
// ❌ BAD: Wastes CPU, starves lower priority threads
while (1) {
    if (check_for_work()) {
        do_work();
    }
    // Thread NEVER blocks! Burns CPU doing nothing!
}
```

### Good Design (Event-Driven):
```c
// ✅ GOOD: Blocks when idle, lets other threads run
while (1) {
    msg = MsgReceive(channel, &buf, sizeof(buf), NULL);  // BLOCKS here
    handle(msg, &buf);  // Does work
    // Loops back → blocks again
}
```

---

## One-Sentence Summary:

> QNX schedules **threads** (not processes) using **priority-based preemptive scheduling** across **clusters of cores**, where threads **block when idle** to share CPU, and the **FIFO/Round Robin/Sporadic** algorithms handle same-priority contention, all with a **fixed scheduling cost** for real-time predictability.