# Mutexes in QNX

## What is a Mutex?

**Mutex** is short for **mutual exclusion**. It ensures that only **one thread at a time** can:

- Execute a critical section of code, or
- *(far more commonly)* Access a critical piece of data

> "Data" here is used in the most general sense — it could be a structure, a class, a linked list, an array of pointers to buffers, or even hardware (e.g., related hardware registers or hardware buffers on a device).

---

## POSIX Mutex API

QNX uses **standard POSIX calls** for mutex manipulation (though other choices are available).

### Core Functions

| Function | Purpose |
|---|---|
| `pthread_mutex_init()` | Initializes the mutex data structure before use. |
| `pthread_mutex_destroy()` | Destroys the mutex (undo of init). Mutexes are also destroyed automatically when the process exits. |
| `pthread_mutex_lock()` | Locks the mutex. |
| `pthread_mutex_unlock()` | Unlocks the mutex. |

---

### How `pthread_mutex_lock()` Works

| Scenario | Behavior |
|---|---|
| Mutex is **not locked** | Marks it as locked by the calling thread. Returns success. ✅ |
| Mutex is **already locked by the calling thread** | Returns `EDEADLK` error (QNX implementation). ❌ |
| Mutex is **locked by another thread** | **Blocks** the calling thread until the mutex is unlocked. ⏳ |

### How `pthread_mutex_unlock()` Works

| Scenario | Behavior |
|---|---|
| Mutex is **locked by the calling thread**, no waiters | Marks mutex as unlocked. ✅ |
| Mutex is **locked by the calling thread**, threads waiting | Marks as unlocked; the **highest priority, longest waiting** thread acquires the lock. ✅ |
| Mutex is **NOT locked by the calling thread** | Returns an error. ❌ |

> **Key Rule:** Only the thread that locked the mutex can unlock it. This is how **ownership** of the mutex (and by extension, the protected data) is enforced.

---

## Basic Usage Example

```c
/* Declare a global mutex */
pthread_mutex_t myMutex;

/* Initialization */
pthread_mutex_init(&myMutex, NULL);  // NULL = default attributes

/* Critical section */
pthread_mutex_lock(&myMutex);
// ... manipulate critical data ...
pthread_mutex_unlock(&myMutex);

/* Cleanup */
pthread_mutex_lock(&myMutex);    // Ensure no one else has it
pthread_mutex_destroy(&myMutex); // QNX allows destroying a self-locked mutex
```

> **Note:** You are **not** allowed to destroy a mutex that another thread has locked. However, QNX **does** allow you to destroy a mutex that **you** have locked.

---

## Why Mutexes? — The Buffer Allocation Problem

### The Problem: Single-Threaded Code Breaks in Multi-Threaded Context

Consider a simple buffer allocator that walks a free list:

```c
void *buf_alloc(int nbytes) {
    // Walk the free list looking for a matching entry
    while (free_list != NULL) {
        if (free_list->size == nbytes)
            break;
        free_list = free_list->next;
    }

    if (free_list != NULL) {
        // Remove entry from list
        // Return pointer to the block
    }
    return NULL;
}
```

**In a single-threaded program** → Works perfectly fine. ✅

**In a multi-threaded program** → Two threads call `buf_alloc()` for the same size simultaneously:

1. Both threads walk the list and find the **same entry**.
2. Both threads remove that entry.
3. Both threads receive a pointer to the **same block**.
4. Both threads write their own data → **Data corruption!** 💥

### The Fix: Protect with a Mutex

```c
pthread_mutex_t alloc_mutex = PTHREAD_MUTEX_INITIALIZER;

void *buf_alloc(int nbytes) {
    pthread_mutex_lock(&alloc_mutex);

    // Walk the free list
    while (free_list != NULL) {
        if (free_list->size == nbytes)
            break;
        free_list = free_list->next;
    }

    if (free_list != NULL) {
        // Remove entry, get pointer
        pthread_mutex_unlock(&alloc_mutex);
        return block_ptr;
    }

    pthread_mutex_unlock(&alloc_mutex);
    return NULL;
}
```

Now when two threads call `buf_alloc()`:

1. Only **one** thread acquires the lock and walks the list.
2. The other thread **blocks** until the first unlocks.
3. When the second thread runs, the first block is already removed → it finds a **different** block.
4. No corruption! ✅

> ⚠️ **Important:** Make sure you unlock the mutex on **every code path** after the lock (both success and failure paths).

---

## Mutex Initialization Options

### Explicit Initialization

```c
pthread_mutex_t myMutex;
pthread_mutex_init(&myMutex, NULL);  // Default: will not fail
```

- For default mutexes, `pthread_mutex_init()` simply initializes the data structure — **no kernel allocation**, so it **cannot fail**.
- Special mutexes (e.g., with `PTHREAD_MUTEX_ROBUST`) require kernel resources and **can fail**.

### Automatic Initialization (Static Initializer)

```c
pthread_mutex_t alloc_mutex = PTHREAD_MUTEX_INITIALIZER;
```

- Assigns an initialization state to the mutex at declaration time.
- No explicit `pthread_mutex_init()` call needed.
- Useful when you don't have a convenient place to call init, or you don't know which thread will use the mutex first.

---

## Process-Shared Mutexes (Shared Memory)

When protecting data in **shared memory between processes**, the mutex must be:

1. **Stored in the shared memory** (visible to all processes).
2. Marked as **process-shared** so the kernel tracks it appropriately.

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);

// shmem_addr = pointer to shared memory region
pthread_mutex_t *mutex_ptr = (pthread_mutex_t *)shmem_addr;

int ret = pthread_mutex_init(mutex_ptr, &attr);  // Check return value!
```

> ⚠️ Process-shared mutex init **can fail** because it requires a kernel-side object for tracking. Always check the return value.

---

## Mutexes Don't Actually "Protect" Data

Unlike process-level memory protection (where the OS enforces isolation), **the OS has no knowledge of the association between a mutex and the data it protects**.

### The Analogy

Think of a mutex like a **door with a key**:

- First thread grabs the key, enters the room, locks the door, works on the files.
- Other threads arrive, try the door, find it locked, and **line up in priority order**.
- When the first thread leaves, the next in line enters.

**But really, there's no wall.** There's only a **painted line on the floor** that everyone agrees not to step over.

> If any code accesses the protected data **without locking the mutex**, it can cause corruption — and these bugs are:
> - Extremely **rare** and **intermittent**
> - Nearly **impossible to reproduce**
> - The kind that customers report but you spend hours/days failing to reproduce

**The more usage your code gets, the more likely these rare bugs will surface.**

---

## Deadlock

### The Problem

When threads need to lock **two or more mutexes**, deadlock can occur:

```
Thread 1: locks A → tries to lock B → BLOCKS (B held by Thread 2)
Thread 2: locks B → tries to lock A → BLOCKS (A held by Thread 1)

💀 DEADLOCK: Each thread waits for the other to unlock.
```

> **The OS does NOT detect deadlocks.** Detecting deadlocks in the general case (arbitrary threads, across multiple processes, with various synchronization objects) is an **O(N!) complexity** algorithm — unacceptable for a real-time system.

### The Solution: Locking Hierarchy

Establish a **consistent locking order** for all mutexes. If the rule is **"always lock A before B"**:

```
Thread 1: locks A → tries to lock B → BLOCKS (B held by Thread 2)
Thread 2: has B, needs A → must UNLOCK B first (to respect order A→B)
           → Thread 1 acquires B → Thread 1 has A & B ✅
           → Thread 1 finishes, unlocks both
           → Thread 2 locks A → locks B → proceeds ✅
```

> **No deadlock!** This must be solved at the **design level**. Static analysis tools may warn about potential conflicts, but nothing at the OS level detects this.

---

## Priority Inversion & Priority Inheritance

### The Problem: Priority Inversion

```
Thread A (low priority):   locks mutex → gets preempted
Thread B (medium priority): runs (preempted A)
Thread C (high priority):   preempts B → tries to lock mutex → BLOCKS

Result: Thread C (HIGH) is waiting for Thread A (LOW),
        but Thread A can't run because Thread B (MEDIUM) is running.
        → High priority thread waits on medium priority thread!
```

> This is **priority inversion** — the kind of bug that made the **Mars Pathfinder** continually reboot.

### The Solution: Priority Inheritance (Default in QNX)

When Thread C tries to lock the mutex held by Thread A:

1. The kernel identifies Thread A as the mutex **owner** (ownership tracking).
2. Thread A's priority is **boosted** to Thread C's priority.
3. Thread A runs, finishes its work, and **unlocks** the mutex.
4. Thread A's priority **drops back** to its original level.
5. Thread C **acquires** the mutex and continues.

> ✅ **Priority inheritance is enabled by default** for QNX mutexes.

---

## Keep Mutex Lock Durations Short

Holding mutexes for extended periods causes several problems:

| Issue | Impact |
|---|---|
| **Reduced parallelism** | Other threads waiting on the mutex can't proceed — negates the benefit of multithreading. |
| **Increased priority inversions** | Low-priority threads holding locks are more likely to get boosted, doing work at the wrong priority. |
| **More kernel involvement** | Short locks benefit from the **fast path** (local atomic operations); long locks increase contention and kernel calls. |

### The Fast Path vs. Kernel Path

```
pthread_mutex_lock():
  ├── Fast Path: local atomic "test and set"
  │   → If unlocked: mark as locked, return success (CHEAP ✅)
  │
  └── Slow Path: SyncMutexLock_r() kernel call
      → If contention or special mutex type (EXPENSIVE ❌)
      → Requires context switches and internal atomic operations

pthread_mutex_unlock():
  ├── Fast Path: local unlock, no waiters (CHEAP ✅)
  └── Slow Path: kernel call to unblock waiting thread (EXPENSIVE ❌)
```

> **The shorter your lock periods, the more likely you hit the fast (local, no-kernel) path.**

---

## Key Takeaways

1. **Mutexes** provide mutual exclusion — only one thread accesses protected data at a time.
2. **Only the locking thread can unlock** the mutex (ownership semantics).
3. Mutexes don't provide OS-enforced protection — it's a **programmer's contract**.
4. Always **unlock on every code path** after a lock.
5. **Deadlocks** must be prevented at design time using a **locking hierarchy**.
6. QNX provides **priority inheritance by default** to prevent priority inversion.
7. **Keep lock durations short** to maximize parallelism and hit the fast (atomic, no-kernel) path.