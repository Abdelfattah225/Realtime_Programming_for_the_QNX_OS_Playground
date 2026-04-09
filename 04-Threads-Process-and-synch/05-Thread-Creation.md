

# QNX Thread Creation — Explained from the Lecture

---

## pthread_create() — The Function

```c
int pthread_create(
    pthread_t *thread_id,      // 1. OUTPUT: ID of new thread (NULL = don't care)
    const pthread_attr_t *attr, // 2. CONFIG: How to create it (NULL = defaults)
    void *(*start_func)(void*), // 3. FUNCTION: Where thread starts running
    void *arg                   // 4. ARGUMENT: Passed to start_func (unexamined)
);
```

### Simple Example:

```c
void *my_thread_func(void *arg) {
    int *value = (int *)arg;
    printf("Thread running! Value = %d\n", *value);
    return NULL;
}

int main() {
    pthread_t tid;
    int data = 42;
    
    pthread_create(&tid,          // Get thread ID back
                   NULL,          // Default attributes
                   my_thread_func,// Thread starts HERE
                   &data);        // Passed to my_thread_func as "arg"
    
    // tid now contains the ID of the new thread
    pthread_join(tid, NULL);  // Wait for it to finish
}
```

### Key Points About the 4th Parameter (arg):

```
You pass in a void pointer → OS hands it to your thread function UNTOUCHED

The OS does NOT:
  ❌ Validate it's a valid pointer
  ❌ Dereference it
  ❌ Look at it at all

You pass in 64 bits → you get back the same 64 bits.
```

---

## The Attribute Structure (pthread_attr_t)

Used to **customize** how the new thread is created.

### The Pattern:

```c
// 1. Declare it
pthread_attr_t attr;

// 2. Initialize it
pthread_attr_init(&attr);

// 3. Modify what you want
pthread_attr_set___(&attr, ...);  // Various set functions

// 4. Create thread with it
pthread_create(&tid, &attr, my_func, my_arg);

// 5. Destroy when done
pthread_attr_destroy(&attr);
```

**If you init but change nothing** → same as passing NULL → **default behavior**.

---

## Setting Priority & Scheduling Algorithm

By default, a new thread **inherits** the priority and scheduling algorithm of the thread that created it.

To override this, you must do **three things**:

```c
pthread_attr_t attr;
pthread_attr_init(&attr);

// Step 1: Say "I'm setting scheduling explicitly, don't inherit"
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

// Step 2: Set the scheduling policy
pthread_attr_setschedpolicy(&attr, SCHED_RR);  // Round Robin

// Step 3: Set the scheduling parameters (priority)
struct sched_param param;
param.sched_priority = 15;
pthread_attr_setschedparam(&attr, &param);

// Now create the thread: priority 15, Round Robin
pthread_create(&tid, &attr, my_func, my_arg);

pthread_attr_destroy(&attr);
```

### What Happens If You Skip Step 1:

```
Without PTHREAD_EXPLICIT_SCHED:
  Creator thread is priority 20, FIFO
  New thread → inherits → priority 20, FIFO
  (Steps 2 and 3 are IGNORED!)

With PTHREAD_EXPLICIT_SCHED:
  Creator thread is priority 20, FIFO
  New thread → uses attr values → priority 15, Round Robin ✅
```

### Available Scheduling Policies:

| Policy | Parameters | Description |
|--------|-----------|-------------|
| **SCHED_FIFO** | priority only | Run until blocked |
| **SCHED_RR** | priority only | Round Robin with time slice |
| **SCHED_OTHER** | priority only | Same as FIFO in QNX |
| **SCHED_SPORADIC** | priority, low_priority, replenishment period, budget | Budget-based priority switching |
| **SCHED_NOCHANGE** | parameters only | Keep current policy, just change parameters |

---

## Stack Size

### Defaults:

```
Main thread (first thread):    512 KB stack
Additional threads:            256 KB stack
Guard page:                      4 KB (always allocated)
```

### What is the Guard Page?

```
┌──────────────────────┐  ← Top of stack
│                      │
│   Thread's Stack     │
│   (256 KB of RAM)    │  ← Actual usable stack memory (committed RAM)
│                      │
├──────────────────────┤
│   Guard Page (4 KB)  │  ← Unmapped/unusable memory
│   CRASH if touched!  │     Catches stack overflows
└──────────────────────┘  ← Bottom

If your code uses too much stack and "overflows":
  → Hits the guard page
  → Unmapped memory → CRASH (SIGSEGV)
  → You know you overflowed your stack
  
Without guard page:
  → Overflow silently corrupts other memory
  → Hard-to-find bugs!
```

### Customizing Stack Size:

```c
pthread_attr_t attr;
pthread_attr_init(&attr);

// Set stack to 64 KB instead of default 256 KB
pthread_attr_setstacksize(&attr, 64 * 1024);

// Rules:
// - Must be at least PTHREAD_STACK_MIN
// - Rounded UP to multiple of page size (4 KB)
// - Guard page is ALWAYS allocated (in addition to stack)

pthread_create(&tid, &attr, my_func, my_arg);
pthread_attr_destroy(&attr);
```

### Customizing Guard Page Size:

```c
// Want a bigger guard page? (default is 4 KB)
pthread_attr_setguardsize(&attr, 16 * 1024);  // 16 KB guard
```

---

## Summary

| Aspect | Default | How to Change |
|--------|---------|---------------|
| **Thread ID** | Returned via first param | Pass NULL if you don't care |
| **Priority** | Inherits from creator | `setinheritsched(EXPLICIT)` + `setschedparam` |
| **Scheduling** | Inherits from creator | `setinheritsched(EXPLICIT)` + `setschedpolicy` |
| **Stack size** | 256 KB (additional threads), 512 KB (main) | `pthread_attr_setstacksize()` |
| **Guard page** | 4 KB (always present) | `pthread_attr_setguardsize()` for larger |
| **Argument** | Passed untouched to thread function | 4th param of `pthread_create()` |