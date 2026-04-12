

# Synchronization in QNX

## Overview

Threads within a process share all of the process's resources, including:

- File descriptors
- Channels
- Timers
- Memory
- And more...

**Example:** If a global variable (declared outside of any function, in the C/C++ sense) exists in memory, any thread within that process can read from or write to it.

> While shared resources introduce powerful solutions, they also introduce **synchronization problems**.

---

## The Problem

When multiple threads share resources, several issues can arise:

| Problem | Description |
|---|---|
| **Multiple Writers** | Multiple threads writing to a shared resource simultaneously results in corrupted/inconsistent data. |
| **Reader-Writer Conflict** | A reader may read while a writer is mid-write, resulting in **invalid or stale data**. |
| **Shared Resource Conflicts** | Similar problems occur with any shared resource (e.g., GPIO, file descriptors). |

These are known as **synchronization problems**.

---

## Synchronization Tools

### Primary Tools (Covered in This Section)

| Tool | Purpose |
|---|---|
| **Mutexes** | Mutual exclusion — ensures only one thread accesses a critical section at a time. |
| **Condition Variables (Condvars)** | Allow threads to wait for a specific condition to become true. |
| **Atomic Operations** | Perform read-modify-write operations that are guaranteed to be indivisible. |

---

### Other Synchronization Mechanisms (Not Covered in Detail)

#### 1. Semaphores
- Provide a **notification-type mechanism** on dispatch.
- Include a **counter**, which adds flexibility beyond simple lock/unlock semantics.

#### 2. Reader/Writer Locks
A more granular locking mechanism that distinguishes between read and write access:

- ✅ **Multiple readers** can access the data **simultaneously** (safe — they don't modify data).
- ❌ **No writers** are allowed while readers are active.
- ✅ Only **one writer** at a time is permitted.
- ❌ **No readers** are allowed while a writer is writing.

```
Multiple Readers → Allowed (no writers at the same time)
Single Writer    → Allowed (no readers and no other writers at the same time)
```

#### 3. Barriers
- Block threads until a **required number of threads** have reached the barrier.
- Once all required threads arrive, they are **all released at once**.

**Use Case — Initialization:**
> Multiple threads perform initialization work. They all wait at a barrier for each other to finish before proceeding to their main loops.

#### 4. Once Control
- Ensures a provided function is called **only once** for the entire life of the process.
- **Use Case:** Library initialization that must execute exactly once, regardless of how many threads call it.

#### 5. Thread Local Storage (TLS)
- Stores and retrieves an object on a **per-thread basis**.
- Each thread gets its own copy of the data.
- **Use Case:** Writing a library that may be used in either single-threaded or multi-threaded processes — TLS ensures each thread has its own private data.

---

## Synchronization Object Release: Who Gets It?

When a synchronization object is released (e.g., a semaphore is posted, a mutex is unlocked, a condition variable is signaled, or a reader/writer lock is unlocked), the question arises:

> **Which blocked thread receives the synchronization object?**

### Rules (QNX-Specific Behavior)

1. **Highest priority thread** wins.
2. If multiple threads share the **same highest priority**, the **longest waiting thread** wins.

> **"Blocked"** in QNX means the thread is receiving **no CPU time** — it is suspended, waiting for the resource.

```
Multiple threads blocked waiting
        │
        ▼
  Highest Priority?
        │
   ┌────┴────┐
   │ Unique  │ Tied
   │         │
   ▼         ▼
 That      Longest
 thread    waiting
 gets it   thread
           gets it
```

---

## Key Takeaways

- Shared resources between threads are powerful but require careful synchronization.
- **Mutexes**, **condvars**, and **atomic operations** are the primary tools for solving synchronization problems.
- QNX resolves contention on synchronization objects using **priority** first, then **wait time** as a tiebreaker.
- Additional tools like **semaphores**, **reader/writer locks**, **barriers**, **once control**, and **thread local storage** provide specialized solutions for specific synchronization patterns.