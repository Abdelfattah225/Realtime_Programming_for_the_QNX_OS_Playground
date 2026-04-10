# QNX Thread Operations — Explained from the Lecture

---

## 1. How Threads Go Away (Two Ways)

```c
// Way 1: Explicitly call pthread_exit()
void *my_thread(void *arg) {
    // do work...
    pthread_exit((void *)42);  // Thread dies, return value = 42
}

// Way 2: Return from thread function
void *my_thread(void *arg) {
    // do work...
    return (void *)42;  // Same effect — C library calls pthread_exit() for you
}
```

**Important:** Returning from `main()` kills the **entire process**, not just the main thread. Returning from a `pthread_create()`'d thread function only kills **that thread**.

---

## 2. Key Thread Operations

| Function | What It Does |
|----------|-------------|
| **pthread_exit()** | Thread terminates itself, passes a return value |
| **pthread_join()** | Block until a thread dies, get its return value |
| **pthread_kill()** | Deliver a **signal** to a thread (misleading name — NOT termination!) |
| **pthread_cancel()** | Ask a thread to terminate (asynchronous, can be overridden) |
| **pthread_detach()** | Make thread non-joinable (auto-cleanup, no zombie) |
| **pthread_self()** | Get your own thread ID |
| **pthread_setname_np()** | Give your thread a name (for debugging) |

---

## 3. pthread_join() — Waiting for Thread Death

**Any thread can join any other thread** — no parent/child relationship needed!

```c
void *result;
pthread_t tid;

// Thread 1 creates Thread 2
pthread_create(&tid, NULL, worker_func, NULL);

// Thread 1 waits for Thread 2 to die
pthread_join(tid, &result);
// result now contains whatever Thread 2 passed to pthread_exit()
printf("Thread returned: %ld\n", (long)result);
```

### Key Rules:

```
Thread 1 creates Thread 2:
  Thread 1 can join Thread 2  ✅
  Thread 2 can join Thread 1  ✅  (no parent/child restriction!)

Join timing:
  Thread dies BEFORE join is called → join returns IMMEDIATELY with the info
  Join called BEFORE thread dies    → join BLOCKS until thread dies

A thread can only be joined ONCE:
  Thread A joins Thread 3  → gets the data ✅
  Thread B joins Thread 3  → gets an ERROR ❌ (already joined)
```

---

## 4. pthread_detach() — The "No Zombie" for Threads

Just like processes can become zombies waiting for `waitpid()`, threads can become "zombies" waiting for `pthread_join()`.

```c
// Thread that nobody will ever join:
pthread_t tid;
pthread_create(&tid, NULL, background_worker, NULL);

// "I'll never join this thread, so don't keep its info around"
pthread_detach(tid);

// When background_worker dies:
//   ❌ No thread zombie — auto-cleanup
//   ❌ Can NEVER join it or get its return value
```

The instructor draws the parallel:
> "Essentially makes it like a no-zombie process. A thread never becomes a zombie if it is detached."

---

## 5. pthread_self() and Thread ID 0

```c
void *my_thread(void *arg) {
    pthread_t my_id = pthread_self();  // "What's MY thread ID?"
    printf("I am thread %d\n", my_id);
    
    // QNX extension: thread ID 0 = "myself"
    // Since thread IDs start from 1, 0 is never a real thread ID
    pthread_setschedparam(0, ...);  // Change MY OWN scheduling
    // Same as:
    pthread_setschedparam(pthread_self(), ...);
}
```

---

## 6. Naming Threads (For Debugging)

```c
pthread_setname_np(pthread_self(), "sensor_reader");
pthread_setname_np(hw_thread, "hw_interrupt_handler");

// Now in debugger or pidin output:
//   Thread 1: "sensor_reader"
//   Thread 2: "hw_interrupt_handler"
// Instead of just seeing "Thread 1", "Thread 2"
```

---

## 7. Changing Priority While Running

```c
// Change current thread to priority 25, Round Robin
struct sched_param param;
param.sched_priority = 25;

pthread_setschedparam(0,           // 0 = myself (QNX extension)
                      SCHED_RR,    // Round Robin
                      &param);     // Priority 25

// Query current values
int policy;
struct sched_param current;
pthread_getschedparam(0, &policy, &current);
printf("Policy: %d, Priority: %d\n", policy, current.sched_priority);
```

---

## 8. pthread_kill() — NOT What You Think!

The instructor warns:

> "It looks like it should kill a thread, but that name really is misleading. 'Kill' here means deliver a signal."

```c
pthread_kill(tid, SIGUSR1);  // Delivers SIGUSR1 to thread tid

// What happens depends on how the signal is set up:
//
// Case 1: Signal is MASKED     → held pending (nothing happens yet)
// Case 2: Signal is IGNORED    → discarded (nothing happens)
// Case 3: Signal HANDLER exists → thread runs the handler
// Case 4: NONE of the above    → ENTIRE PROCESS DIES!
```

---

## 9. Thread Termination: Self vs External

### Self-Termination (Preferred):
```c
void *my_thread(void *arg) {
    // Setup...
    
    // Do work...
    
    // Clean up (BEFORE dying):
    unlock_mutexes();
    free(my_buffer);
    close(my_file);
    
    // Then die:
    pthread_exit(NULL);
    // OR: return NULL;
}
```

### External Termination:
```c
// From another thread:
pthread_cancel(tid);   // "Please die" — can be controlled/ignored
pthread_abort(tid);    // "Please die" — harder to ignore
```

The instructor recommends **against** async cancellation:

> "The problem with something like cancellation is it's asynchronous: We don't know when it's going to happen, and it's really hard to leave any thread-owned resources in a consistent state."

**Better approach — synchronous termination using a flag:**

---

## 10. The Standard Thread Pattern

```c
volatile int done = 0;  // Flag: should this thread stop?

void *my_thread(void *arg) {
    
    // ══════ SETUP ══════
    int fd = open("/dev/sensor", O_RDONLY);
    char *buffer = malloc(4096);
    pthread_mutex_lock(&init_mutex);
    // ... initialization work ...
    pthread_mutex_unlock(&init_mutex);
    
    
    // ══════ MAIN LOOP ══════
    while (!done) {
        // BLOCK: Wait for work (message, pulse, interrupt, timer, etc.)
        int n = read(fd, buffer, 4096);
        
        // DO WORK: Handle whatever came in
        process_sensor_data(buffer, n);
        
        // Check: should I stop?
        // (done flag might be set by another thread)
    }
    
    
    // ══════ CLEANUP ══════
    // Reverse everything from setup:
    free(buffer);           // Release allocated memory
    close(fd);              // Close opened files
    // Unlock any held mutexes
    // Release any other resources this thread owns
    
    return NULL;  // Thread dies cleanly
}

// Another thread tells this thread to stop:
void stop_sensor_thread() {
    done = 1;  // Set the flag — thread will see it and exit cleanly
}
```

---

## 11. What to Clean Up Before Thread Death

| Resource | Clean Up How |
|----------|-------------|
| **Locked mutexes** | `pthread_mutex_unlock()` |
| **Reader/writer locks** | `pthread_rwlock_unlock()` |
| **Allocated memory** | `free()` |
| **Opened files/streams** | `close()` / `fclose()` |
| **POSIX message queues** | `mq_close()` |
| **Semaphores** | `sem_close()` |
| **Thread stack** | ❌ Automatic — don't manually free! |
| **Local variables** | ❌ Automatic — cleaned up on exit |

### Exception: Main Thread's Stack

```
Created threads (pthread_create):
  Stack is auto-freed when thread exits ✅

Main thread:
  Stack is kept for the ENTIRE process lifetime ⚠️
  Why? Command line args (argv) and environment variables (environ)
  live on the main thread's stack. The C runtime needs them
  for the whole life of the process.
```

---

## 12. Design Tip: Main Thread Should Be Permanent

The instructor advises:

> "If you have any thread that is going to be permanent — there for the entire lifecycle of your process — the main thread should be such a permanent thread."

```c
int main() {
    // Main thread = permanent, manages the process lifetime
    
    // Create worker threads
    pthread_create(&sensor_tid, NULL, sensor_thread, NULL);
    pthread_create(&motor_tid, NULL, motor_thread, NULL);
    
    // Main thread lives forever (or until shutdown)
    while (!shutdown) {
        // Could: manage other threads, wait for signals,
        // handle admin commands, etc.
        pause();  // Or some other blocking call
    }
    
    // Shutdown: signal threads to stop
    done = 1;
    pthread_join(sensor_tid, NULL);
    pthread_join(motor_tid, NULL);
    
    return 0;  // Process exits, all resources cleaned up
}
```

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Thread exit** | `pthread_exit()` or `return` from thread function |
| **Join** | Any thread can join any other (no parent/child). Only once. |
| **Detach** | No zombie, no join possible. Auto-cleanup. |
| **pthread_kill()** | Delivers a SIGNAL, does NOT kill the thread! |
| **Cancel** | Async termination — avoid if possible |
| **Best practice** | Use a "done" flag for synchronous, clean shutdown |
| **Cleanup** | Unlock mutexes, free memory, close files BEFORE exiting |
| **Main thread** | Stack lives forever; make it a permanent thread |