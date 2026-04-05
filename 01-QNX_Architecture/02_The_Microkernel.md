# The Microkernel

## 01 — What is a Kernel Call?

A kernel call is how a **thread asks the kernel to do something** on its behalf.

- It's like a **direct request** to the microkernel
- The thread says: *"Hey kernel, I need you to do this for me"*
- Examples:
  - **`ThreadCreate()`** — Create a new thread
  - **`MsgSend()`** — Send a message to another process
  - **`ClockTime()`** — Get the current time
  - **`InterruptAttach()`** — Register an interrupt handler
- The thread **cannot** do these things on its own — only the kernel has the **privilege**

> 🔑 **Simple analogy:** A kernel call is like **raising your hand in class** and asking the teacher (kernel) to do something only *they* are allowed to do.

---

## 02 — When Does the Kernel Run?

The kernel **does NOT run all the time**. It only runs when **called upon**:

- ✅ When a thread makes a **kernel call**
- ✅ When a **hardware interrupt** fires (e.g., timer tick, device signal)
- ✅ When an **exception** occurs (e.g., a fault or error)
- ❌ It does **NOT** run as a background process
- ❌ It has **NO** thread of its own

> 🔑 **Simple analogy:** The kernel is like a **light switch** — it only activates when someone flips it. It sits idle otherwise. The kernel **borrows** the calling thread's context to do its work.

---

## 03 — Services Provided by the Kernel

The QNX microkernel provides **only essential, minimal services**:

| Service | What It Does |
|---------|-------------|
| **Thread Management** | Create, destroy, schedule threads |
| **Message Passing (IPC)** | `MsgSend()`, `MsgReceive()`, `MsgReply()` — threads talking to each other |
| **Scheduling** | Decides which thread runs next on each core |
| **Interrupt Handling** | Routes hardware interrupts to the right handler |
| **Synchronization** | Mutexes, semaphores, condvars — keeping threads safe |
| **Timers & Clocks** | System time, alarms, timeouts |
| **Signal Delivery** | POSIX signals to processes/threads |

> 🔑 **Key point:** That's it! No file systems, no networking, no drivers. Everything else runs **outside** the kernel as separate processes. This is what makes QNX a **true microkernel**.

---

## 04 — How Threads and Cores Relate to a Kernel Call

This is where it gets interesting:

- A **thread** runs on a **core** (CPU core)
- When that thread makes a **kernel call**, the **same core** switches into **kernel mode**
- The kernel code executes **on that core**, using **that thread's context**
- While in the kernel call:
  - That core is **busy** handling the request
  - Other cores can continue running **their own threads** independently
- Once the kernel call completes, the core **returns to user mode** and the thread continues

### Multi-core behavior:
```
Core 0:  Thread A → makes kernel call → kernel runs on Core 0 → returns to Thread A
Core 1:  Thread B → running normally (unaffected) ✅
Core 2:  Thread C → also makes kernel call → kernel runs on Core 2 simultaneously
```

- Multiple kernel calls can happen **at the same time** on different cores
- The QNX kernel is **preemptible and SMP-safe** (Symmetric Multi-Processing)

> 🔑 **Simple analogy:** Each core is like a **worker at a desk**. When a thread needs kernel help, that worker pauses, puts on a **"kernel hat"**, does the job, takes the hat off, and goes back to the thread's normal work. Other workers at other desks are **unaffected**.

---

## Quick Summary

| # | Concept | One-Liner |
|---|---------|-----------|
| 01 | Kernel Call | A thread's request for the kernel to perform a privileged operation |
| 02 | When Kernel Runs | Only on kernel calls, interrupts, or exceptions — **never on its own** |
| 03 | Kernel Services | Threading, IPC, scheduling, interrupts, sync — **nothing more** |
| 04 | Threads & Cores | The kernel runs on the **same core** as the calling thread; other cores stay independent |

> **Bottom line:** The QNX kernel is **lean, fast, and only active when needed.** It does the bare minimum so everything else can run safely in user space.