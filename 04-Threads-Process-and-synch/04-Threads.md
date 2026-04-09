

# QNX Thread APIs — Quick Explanation

---

## Three Layers of Thread APIs

QNX provides **three levels** of APIs for working with threads:

```
┌─────────────────────────────────────────────┐
│  Layer 3: C11 Threading (most portable)      │
│  thrd_create(), mtx_lock()                   │
├─────────────────────────────────────────────┤
│  Layer 2: POSIX Threading (recommended)      │
│  pthread_create(), pthread_mutex_lock()      │
├─────────────────────────────────────────────┤
│  Layer 1: Kernel Functions (don't use directly)│
│  ThreadCreate(), SyncMutexLock()              │
└─────────────────────────────────────────────┘

C11 and POSIX are built ON TOP OF kernel calls.
```

---

## Why NOT Use Kernel Functions Directly?

The kernel functions are **incomplete by design**. They're just the **kernel-side piece** — the portable functions above them do essential setup work first.

### The Example from the Instructor:

```
What happens when you call pthread_create():

┌──────────────────────────────────────────────────┐
│  pthread_create()  (POSIX — what you call)        │
│                                                    │
│  1. Allocates a STACK for the new thread           │
│     (malloc memory for the thread's stack)         │
│                                                    │
│  2. Calls kernel ThreadCreate()                    │
│     telling it WHERE that stack is                 │
│                                                    │
└──────────────────────────────────────────────────┘

│  ThreadCreate()  (Kernel — what you DON'T call)   │
│                                                    │
│  Just creates the thread using the stack           │
│  it was given. Does NOT allocate the stack itself! │
│  INCOMPLETE without pthread_create's setup!        │
└──────────────────────────────────────────────────┘
```

If you called `ThreadCreate()` directly **without** first allocating a stack, it wouldn't work properly — it expects someone else to have done that setup.

### Same Pattern for Mutexes:
```
pthread_mutex_lock()  → does setup → calls SyncMutexLock()
                                      (kernel does the actual lock)
```

---

## Recommendation

| API | Use It? | Why |
|-----|---------|-----|
| **Kernel functions** (ThreadCreate, SyncMutexLock) | ❌ No | Incomplete, not meant for direct use |
| **POSIX functions** (pthread_create, pthread_mutex_lock) | ✅ **Yes** | Complete, portable, recommended |
| **C11 functions** (thrd_create, mtx_lock) | ✅ Yes | Also portable, newer standard |

> **In one sentence:** Always use **POSIX** (or C11) thread functions because they handle essential setup (like stack allocation) before calling the incomplete kernel functions underneath — the kernel functions are just internal building blocks, not meant to be called directly.