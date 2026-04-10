# QNX Process Termination & Cleanup

---

## Two Types of Cleanup

When a process dies, there are **two kinds** of cleanup:

```
┌─────────────────────────────────────────────────────┐
│  INTERNAL CLEANUP              EXTERNAL CLEANUP      │
│  (your code runs)             (OS cleans up for you) │
│                                                      │
│  Only on NORMAL exit          ALWAYS happens         │
│  ❌ NOT on abnormal exit      (normal or abnormal)   │
└─────────────────────────────────────────────────────┘
```

---

## Internal Cleanup: Normal vs Abnormal Termination

### Normal Termination (Internal cleanup RUNS ✅)

Happens when `exit()` is called — either explicitly or implicitly:

```c
// Explicit: you call exit() yourself
void some_function() {
    exit(0);  // Triggers normal exit processing
}

// Implicit: main thread returns from main()
int main() {
    return 0;  // C runtime calls exit(0) for you
}
```

**What runs during normal exit:**

```
1. atexit() handlers
   ──────────────────
   atexit(my_cleanup);  // Registered earlier
   // When exit() happens → my_cleanup() gets called

2. stdio / C++ stream buffers get FLUSHED
   ────────────────────────────────────────
   printf("Important data");  // Sitting in buffer
   // On normal exit → buffer flushed → data written out

3. C++ global destructors run
   ──────────────────────────
   class MyClass {
       ~MyClass() { /* cleanup */ }  // Gets called on normal exit
   };
   MyClass global_obj;
```

---

### Abnormal Termination (Internal cleanup does NOT run ❌)

Happens when your process dies **without** calling `exit()`:

```
Causes of abnormal termination:
─────────────────────────────
• Last thread calls pthread_exit()
• Last thread returns from its thread function
• Last thread is cancelled or aborted
• Crash: divide by zero → SIGFPE
• Crash: bad pointer dereference → SIGSEGV
• Unhandled signal: kill -9 (SIGKILL — can't handle)
• Unhandled signal: SIGTERM with no handler
```

**What does NOT happen:**

```
❌ atexit() handlers do NOT run
❌ stdio buffers do NOT get flushed
❌ C++ global destructors do NOT run

Example consequence:
  printf("Critical log entry");   // In buffer, not yet written
  *bad_ptr = 42;                  // CRASH! SIGSEGV!
  
  → Log entry is LOST because buffer was never flushed!
```

---

## External Cleanup: OS Always Cleans Up

**Regardless** of normal or abnormal termination, the OS **guarantees** all process-owned resources are released:

```
ALWAYS cleaned up by the OS:
─────────────────────────────
✅ File descriptors closed         (all open files)
✅ QNX connections destroyed       (ConnectAttach connections)
✅ Channels destroyed              (ChannelCreate channels)
✅ Memory mappings unmapped:
    - Thread stacks
    - Heap (malloc'd memory)
    - Code and data segments
    - Shared memory mappings
    - Hardware mappings (mmap'd devices)
✅ Timers deleted
✅ Interrupt handlers detached
```

---

## The Exception: Named Objects SURVIVE Process Death

If a resource has a **pathname** associated with it, it **persists** after the process dies.

```
DIES WITH PROCESS:              SURVIVES PROCESS DEATH:
(unnamed, owned by process)     (named, exists in pathname space)

✅ malloc'd memory               ✅ Regular files
✅ Thread stacks                  ✅ Named shared memory (shm_open)
✅ File descriptors               ✅ Named semaphores (sem_open)
✅ Connections/channels           ✅ POSIX message queues (mq_open)
✅ Timers
✅ Interrupt handlers
```

### Why? This is expected behavior:

```c
// You create a file:
int fd = open("/tmp/report.txt", O_CREAT | O_WRONLY, 0666);
write(fd, "data", 4);
close(fd);
exit(0);

// After process dies → /tmp/report.txt STILL EXISTS
// That's what we expect! Files should survive.
```

### Same for other named objects:

```c
// Shared memory with a name:
int fd = shm_open("/my_shared_data", O_CREAT | O_RDWR, 0666);
// Process dies → "/my_shared_data" STILL EXISTS in /dev/shmem/

// Named semaphore:
sem_t *s = sem_open("/my_lock", O_CREAT, 0666, 1);
// Process dies → "/my_lock" STILL EXISTS in /dev/sem/

// POSIX message queue:
mqd_t q = mq_open("/my_queue", O_CREAT | O_RDWR, 0666, &attr);
// Process dies → "/my_queue" STILL EXISTS
```

---

## How Named Objects Actually Get Removed

Named objects require **explicit unlinking** AND **all users must close them**:

```c
// For files:
unlink("/tmp/report.txt");

// For shared memory:
shm_unlink("/my_shared_data");

// For semaphores:
sem_unlink("/my_lock");

// For message queues:
mq_unlink("/my_queue");
```

### The actual data is freed when BOTH conditions are met:

```
Condition 1: Name has been unlinked     ✅
Condition 2: ALL file descriptors to    ✅
             it have been closed
             (by all processes)

BOTH true → data is actually released from memory/disk
```

### Example:

```
Process A: shm_open("/data") → fd=3
Process B: shm_open("/data") → fd=5

Process A: shm_unlink("/data")  
  → Name "/data" removed from /dev/shmem/
  → No new process can open it
  → BUT the memory is NOT freed yet!
  → Process B still has fd=5 open

Process B: close(5)
  → Last file descriptor closed
  → NOW the actual memory is freed ✅
```

---

## Summary

| Situation | Internal Cleanup | External Cleanup |
|-----------|-----------------|-----------------|
| **Normal exit** (exit() or return from main) | ✅ atexit handlers, buffer flush, destructors | ✅ OS cleans everything |
| **Abnormal exit** (crash, signal, last thread dies) | ❌ Nothing runs | ✅ OS still cleans everything |

| Resource Type | Survives Process Death? |
|--------------|------------------------|
| malloc'd memory, stacks, heap | ❌ No — freed by OS |
| File descriptors, connections, channels | ❌ No — closed by OS |
| Timers, interrupt handlers | ❌ No — removed by OS |
| **Named files** | ✅ Yes — must `unlink()` |
| **Named shared memory** | ✅ Yes — must `shm_unlink()` |
| **Named semaphores** | ✅ Yes — must `sem_unlink()` |
| **Named message queues** | ✅ Yes — must `mq_unlink()` |

> **Key takeaway:** The OS **always** cleans up process-owned resources (even on crashes), but **named objects persist** in the pathname space until explicitly unlinked AND all users have closed them.