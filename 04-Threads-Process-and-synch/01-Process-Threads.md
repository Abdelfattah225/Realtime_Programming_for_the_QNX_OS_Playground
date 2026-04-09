# QNX Processes & Threads — Explained from the Lecture

---

## 1. What is a Process?

A program loaded into memory and executing. Identified by a **PID (Process ID)**.

A process does **two things**:

### a) Owns Resources
- Memory (code, data, heap, stack)
- Hardware mappings
- Open files
- System timers
- IPC context (connections, channels)
- Synchronization (mutexes, condition variables)

### b) Provides a Security Context

**Identity** (who is this?):
- User ID, Group ID, supplementary groups
- Controls: **what can this process OPEN?**

**Type + Abilities** (what can it do?):
- Controls: **what can this process ASK the OS to do?**
- Attach interrupts, map hardware, kill other processes, etc.

### What Does "Owning" Mean?

**Protection:** Another process **cannot** see, modify, or damage your resources.

**Cleanup:** When your process dies — whether normal (`exit()`), crash (divide by zero, bad pointer), or killed by a signal — **everything it owns gets cleaned up automatically**.

---

## 2. What is a Thread?

The **flow of execution** through your process — an instruction pointer walking through code, calling functions, running loops, accessing hardware.

Threads have **attributes** (not resources), related to **how/where/when/why** they get to run:

- **Priority & scheduling algorithm** (FIFO, Round Robin)
- **Register set** (instruction pointer, stack pointer, general purpose registers — stored away when preempted)
- **CPU affinity mask** (which cores it can run on)
- **Signal mask** (which signals it can handle)
- **Hardware access privileges**

---

## 3. The Key Distinction

> **Threads run code. Processes own resources.**

Threads within the same process **share all resources**:

As the instructor says:
> "Thread one could open a file, thread two could read and write from it, and thread five could close that file. Thread two could allocate some memory, thread three could update that memory, and thread one could free that memory."

A process **must** have at least one thread. If the last thread exits, the process dies and everything is cleaned up.

---

## 4. The Assembly Line Example

The instructor designs a system to control an assembly line with: a conveyor belt motor, a stamper, a small drill, and a large drill.

### Bad Approach: One Big Process
```
One process "assembly_line_control":
  Thread 1 → conveyor belt
  Thread 2 → stamper
  Thread 3 → big drill
  Thread 4 → small drill
```

### Why the instructor rejects this:

> "Controlling a stamper and controlling a drill probably come with completely different code, probably almost completely different data. Similarly, stamper and conveyor belt motor: different code, different data. These things have different resources, they belong in different processes."

### Better Approach: Separate Processes

```
Process 1: motor_control       → controls conveyor belt
Process 2: stamper_control     → controls stamper
Process 3: drill -big          → controls big drill    ← same program,
Process 4: drill -small        → controls small drill  ← different config!
Process 5: line_control        → overall coordination
Process 6: logger              → system logging
Process 7: user_interface      → operator display
```

Note about the drill: **one program, run twice** with different command-line arguments = **two separate processes**.

### The Stamper Jam Scenario (IPC in Action)

The instructor walks through what happens when the stamper jams:

```
1. Stamper process → tells Line Control: "I'm jammed!"

2. Line Control:
   → tells Motor: "STOP"
   → tells Drills: "STOP"  
   → tells Logger: "Log this jam event"
   → tells UI: "Alert a human!"

3. UI:
   → shows alert on screen
   → starts siren / flashing yellow light

4. Human operator:
   → physically unjams the stamper
   → tells UI: "I've fixed it"

5. UI → tells Line Control: "Operator says it's fixed"

6. Line Control → tells Stamper: "Recheck yourself"

7. Stamper → rechecks → tells Line Control: "All good"

8. Line Control:
   → tells Motor: "Start again"
   → tells Drills: "Start again"
   → tells Logger: "Log restart"

System resumes!
```

All of this communication happens through **IPC** (messages, pulses, shared memory, etc.).

---

## 5. Processes as Opaque Objects

The instructor emphasizes a design principle:

> "I should never know that I want to talk to thread 3 in another process. I should talk to the process as a whole."

Like object-oriented design, but **enforced by the OS**:

- In C++: "I don't look there" (convention)
- In QNX: "I **can't** see there" (hardware-enforced via MMU)

### Benefits:
- **Scalability** — add threads internally without changing external interfaces
- **Configurability** — reimplementation without external changes
- **Maintainability** — update, fix, or improve one process without touching others

---

## 6. Single-Threaded vs Multi-Threaded: When to Use Each

### Single-threaded is BETTER when possible:

The instructor is clear:

> "A single threaded process is a lot easier to write, a lot easier to debug, a lot easier to fix... A single threaded process is a nice predictable system. You run it, and you run it again with the same configuration, and it does the same stuff."

> "A multi threaded process is essentially a chaotic system. You can run it the same way 15 times and get 15 different paths through it."

Also for safety: unit testing frameworks are essentially single-threaded.

### So when DO you need multithreading?

> **"I need to start something new before I finish something old."**

The instructor gives **two examples**:

---

### Example 1: Driver with Hardware Deadline

**Situation:** A driver handles client requests (take ~1ms each) AND hardware interrupts (must respond within 200μs).

**Single-threaded attempt (BAD):**
```
Thread is processing a 1ms client request...
  Every 20μs, polls: "Hardware event? No."
  Continue working...
  Every 20μs, polls: "Hardware event? No."
  Continue working...
  Every 20μs, polls: "Hardware event? YES!"
  
  Now must:
  - Save client request state somehow
  - Handle hardware event
  - Restore client request state
  - Continue

Problems:
  ❌ Wastes time polling ("Is there work? No" = wasted cycles)
  ❌ Lost 20-30μs between polls (less time to meet 200μs deadline)
  ❌ Complex state saving/restoring code
```

**Multi-threaded solution (GOOD):**
```
Thread A (lower priority): handles client requests
Thread B (higher priority): waits for hardware interrupts

Case 1: No hardware interrupt during client request
  Thread A processes the 1ms request uninterrupted. Done. Easy.

Case 2: Hardware interrupt arrives mid-request
  Thread B unblocks immediately (it was waiting for the interrupt)
  Thread B is higher priority → PREEMPTS Thread A
  Thread B handles hardware (has full 200μs, no wasted poll time!)
  Thread B finishes → blocks again
  Thread A resumes exactly where it was
  (OS saved and restored Thread A's state automatically!)

Benefits:
  ✅ No polling waste
  ✅ Full 200μs for hardware response
  ✅ OS handles state saving (no manual save/restore)
```

---

### Example 2: File System with Multiple Priority Clients

**Situation:** A filesystem has a low-priority client reading a 20MB file and a high-priority client needing to write a 20-byte critical log quickly.

**Single-threaded (BAD):**
```
Thread starts reading 20MB file for low-priority client...
High-priority client arrives: "Write 20 bytes NOW!"
  
  Must poll for new requests while reading
  Must save state of the big read
  Must handle the small write
  Must restore the big read
  
  ❌ Complex, hard to meet deadlines
```

**Multi-threaded (GOOD):**
```
Pool of threads waiting on the same channel for requests:

Thread A picks up: "Read 20MB" from low-priority client
  → Thread A runs at low priority, starts the big read

Thread B picks up: "Write 20 bytes" from high-priority client  
  → Thread B runs at high priority
  → PREEMPTS Thread A (or runs in parallel on another core!)
  → Writes the 20 bytes quickly
  → Done!

Thread A continues the 20MB read in the background.

✅ Both requests handled properly
✅ High-priority client not delayed
✅ Could even run in PARALLEL on multicore!
```

### The Rule from the Instructor:

> "If I can complete every one of my operations fast enough that I get to the next one without taking too long, write my code single threaded, and I'll be far happier and it's easier to maintain."

> "But very, very often we have this demand, this need to start new requests before we've finished old requests. That's what multithreading gets us."

---

## 7. Process Memory Layout

Each process has virtual addresses containing these blocks:

```
┌────────────────────────────────┐
│ Thread Stacks                   │  Each thread gets its own stack
├────────────────────────────────┤
│ Program Code (read-only)        │  Your compiled instructions
├────────────────────────────────┤
│ Program Data                    │  Global variables, static data
├────────────────────────────────┤
│ Heap (grows upward)            │  malloc/new — dynamic allocation
├────────────────────────────────┤
│ Shared Memory                   │  Shared objects, HW mappings
├────────────────────────────────┤
│ Shared Libraries                │  libc.so, libgcc.so, ldqnx.so
│ (code + data for each)         │  (every process has at least these 3)
└────────────────────────────────┘
```

### ASLR (Address Space Layout Randomization)

**Default: ON** — each block is placed at a **random** location in each process.

```
WITHOUT ASLR:                       WITH ASLR:
Process A:                          Process A:
  Stack at 0x1000                     Stack at 0x7A30
  Code at  0x5000                     Code at  0x2F10
  Heap at  0x8000                     Heap at  0xB400

Process B:                          Process B:
  Stack at 0x1000  ← same!           Stack at 0x4C80  ← different!
  Code at  0x5000  ← same!           Code at  0x9120  ← different!
  Heap at  0x8000  ← same!           Heap at  0x6E00  ← different!
```

### The Safety vs Security Tradeoff:

The instructor raises an interesting point:

> "In a safety system, that's an interesting question. You may or may not want it because it makes everything different every run. And one of the things we want in a safety system is: everything is the same as the way we tested in every run."

```
SECURITY (wants ASLR ON):
  ✅ Attacker can't predict memory locations
  ✅ Makes exploits much harder

SAFETY (might want ASLR OFF):
  ✅ Same layout every run = predictable = testable
  ✅ "Same as we tested" = trustworthy

Compromise: ASLR ON for non-safety (QM) components
            ASLR OFF for safety-critical (ASIL) components
```

---

## Summary

| Concept | Key Point from the Lecture |
|---------|--------------------------|
| **Process** | Owns resources + provides security context |
| **Thread** | Flow of execution, has attributes (priority, registers, affinity) |
| **Key distinction** | "Threads run code, processes own resources" |
| **Design principle** | Separate by functional units (different code/data = different process) |
| **Drill example** | Same program, different config = two processes |
| **Processes = opaque** | Never talk to "thread 3 in another process" — talk to the process |
| **Single-threaded** | Easier, predictable, testable — use when possible |
| **Multi-threaded** | When you need to "start something new before finishing something old" |
| **Driver example** | HW thread (high priority) preempts client thread — OS saves state |
| **Filesystem example** | Thread pool — high priority write preempts or runs parallel to big read |
| **ASLR** | On by default (security), but may turn off for safety (predictability) |