# QNX Security Policies — Complete Explanation with Examples

---

## 1. WHY Security Policies Exist

### The Core Problem: Developers CAN'T Know the Final Security Needs

When a developer writes a driver or library, they have **no idea** how it will be used in the final product.

### Example 1: Serial Port Driver

```
DEVELOPER writes devc-ser8250 (generic serial driver)

When WRITING the code:                 When DEPLOYING on System A:
─────────────────────                  ──────────────────────────
"I need SOME interrupt..."             "Use interrupt 4 at 0xFF000000"
"I need to map SOME registers..."      Command: devc-ser8250 -b115200 0xFF000000,4

                                       When DEPLOYING on System B:
                                       ──────────────────────────
                                       "Use interrupt 7 at 0xAA000000"
                                       Command: devc-ser8250 -b9600 0xAA000000,7

Developer CANNOT hard-code:
  ❓ "Which interrupt privilege do I need?" → Don't know yet
  ❓ "Which physical address to map?"      → Don't know yet
  ❓ "Which privileges to keep vs drop?"   → Don't know yet
```

### Example 2: File System Driver

```
DEVELOPER writes a filesystem driver that SUPPORTS dynamic mounting
(e.g., USB stick plugged in at runtime)

CUSTOMER A (Car Infotainment):         CUSTOMER B (Industrial Controller):
┌──────────────────────────┐          ┌──────────────────────────┐
│ Has USB ports             │          │ No USB ports             │
│ Users plug in USB sticks  │          │ All filesystems fixed    │
│ Dynamic mounting NEEDED   │          │ at boot time             │
│                           │          │                          │
│ Driver MUST KEEP:         │          │ Driver should DROP:      │
│ ✅ Add pathnames at       │          │ ❌ Add pathnames at      │
│    runtime                │          │    runtime               │
│ ✅ Map new hardware       │          │ ❌ Map new hardware      │
│    devices                │          │    devices               │
└──────────────────────────┘          └──────────────────────────┘

SAME code, DIFFERENT security needs!
The developer CANNOT know which customer will use it.
```

---

### Who IS Responsible for Security?

```
COMPONENT DEVELOPERS                    SYSTEM INTEGRATOR
(write individual pieces)               (assembles the final product)
                                        
┌─────────────────┐                     
│ Serial driver   │──┐                  ┌──────────────────────────┐
│ developer       │  │                  │ "I know:                 │
└─────────────────┘  │                  │  - The exact hardware    │
┌─────────────────┐  │                  │  - Which features used   │
│ Filesystem      │──┤── Pieces ──→     │  - What security needed  │
│ developer       │  │                  │  - The final product"    │
└─────────────────┘  │                  │                          │
┌─────────────────┐  │                  │ Vendors:                 │
│ Middleware       │──┤                  │  NVIDIA, Qualcomm (HW)  │
│ (ROS, AUTOSAR) │  │                  │  QNX (OS)               │
└─────────────────┘  │                  │  Third parties (middle)  │
┌─────────────────┐  │                  │                          │
│ App developer   │──┘                  │ FINAL SECURITY is MY    │
└─────────────────┘                     │ responsibility!          │
                                        └──────────────────────────┘

Security policies = tool to give the SYSTEM INTEGRATOR
                    control over ALL components' security
```

---

## 2. What IS a Security Policy?

A security policy is a **text file** that defines:
- **Security types** (names assigned to processes)
- **Privileges** each type gets
- **Transitions** between types (init → run)
- **Path access** (what names can be registered)

### The Pipeline:

```
HOST (development PC)                    TARGET (embedded board)
─────────────────────                    ──────────────────────

┌────────────────────┐
│ policy.txt          │    Text file you write/edit
│ (human-readable)    │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ secpolcompile       │    Compiles text → binary
│ (safety QUALIFIED)  │    (like a compiler for policies)
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ policy.bin          │    Binary policy file
│ (machine-readable)  │
└─────────┬──────────┘
          │  (copy to target)
          ▼
┌────────────────────┐
│ secpolpush          │    Loads binary policy into kernel
│ (safety CERTIFIED)  │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ procnto (kernel)    │    Enforces the policy at runtime
│ Security enforced!  │
└────────────────────┘
```

---

## 3. All the Security Policy Tools

| Tool | Where It Runs | Safety Status | What It Does |
|------|--------------|---------------|-------------|
| **secpolcompile** | Host (dev PC) | Safety **Qualified** | Compiles text policy → binary policy |
| **secpolpush** | Target (board) | Safety **Certified** | Loads binary policy into the kernel |
| **secpol** | Target (board) | NOT certified (debug only) | View/inspect the loaded policy |
| **libsecpol.so** | Target (board) | Safety **Certified** | API to programmatically query policy (e.g., name → type number) |
| **secpolgenerate** | Target (board) | NOT certified (helper) | Auto-generates a starting policy by observing system behavior |
| **secpollaunch** | Target (board) | Certified | Privileged launcher — safely starts processes with higher privileges |

---

## 4. Security Types — The Core Concept

Every process gets assigned a **security type** (a name). The policy defines what privileges each type has.

### Example:
```
Process: random (supplies random numbers to the system)

Security Types assigned to it:
  random_t        ← During INITIALIZATION (high privilege)
  random_t__run   ← During OPERATION (low privilege)
```

---

## 5. The Two-Phase Lifecycle: Init → Run

Almost every long-running process has **two stages**:

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  INITIALIZATION PHASE              OPERATIONAL PHASE          │
│  (needs HIGH privileges)           (needs LOW privileges)     │
│                                                               │
│  ┌───────────────────────┐        ┌───────────────────────┐  │
│  │ Map hardware registers│        │ Handle client requests │  │
│  │ Attach to interrupts  │        │ Read/write data        │  │
│  │ Register pathname     │        │ Process messages       │  │
│  │ Allocate DMA buffers  │        │                        │  │
│  │ Open connections      │        │ Don't need:            │  │
│  └───────────┬───────────┘        │ ❌ Map more hardware   │  │
│              │                     │ ❌ Attach interrupts   │  │
│              │  secpol_            │ ❌ Register new paths  │  │
│              │  transition_type()  │                        │  │
│              │  (NULL,NULL,0)      │                        │  │
│              └────────────────────→│                        │  │
│                                    └───────────────────────┘  │
│                                                               │
│  Type: random_t                    Type: random_t__run        │
│  HIGH privileges                   LOW privileges             │
│                                    (only what's still needed) │
└──────────────────────────────────────────────────────────────┘
```

### Code Example (What the Developer Does):
```c
#include <secpol/secpol.h>

int main(int argc, char *argv[]) {
    
    // ═══════ INITIALIZATION PHASE ═══════
    // Running as type: random_t (HIGH privileges)
    
    // Map hardware registers (needs MEM_PHYS privilege)
    ptr = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PHYS, NOFD, HW_ADDR);
    
    // Attach to interrupt (needs INTERRUPT privilege)
    InterruptAttach(irq, handler, NULL, 0, 0);
    
    // Register in pathname space (needs PATHSPACE privilege)
    secpol_resmgr_attach(...);  // ← uses secpol version!
    
    // Open connection to slogger (needs CONNECTION privilege)
    slogger_fd = open("/dev/slog", O_WRONLY);
    
    
    // ═══════ TRANSITION ═══════
    // "I'm done initializing. Drop to lower privileges."
    secpol_transition_type(NULL, NULL, 0);
    // Now running as type: random_t__run (LOW privileges)
    
    
    // ═══════ OPERATIONAL PHASE ═══════
    // Only has: srandom ability (push random data to kernel)
    // Can NOT: map hardware, attach interrupts, register paths
    
    while (1) {
        // Normal operation — handle requests
        // If compromised here, attacker has VERY limited power!
        MsgReceive(channel, &msg, sizeof(msg), NULL);
        handle_request(&msg);
    }
}
```

---

## 6. What Developers Must Do (Just Two Things!)

### Thing 1: Call `secpol_transition_type()` After Init

```c
// After all initialization is done:
secpol_transition_type(NULL, NULL, 0);

// This tells the security system:
// "I'm moving from my INIT type to my RUN type"
// The policy file defines what privileges each type has
```

### Thing 2: Use `secpol_resmgr_attach()` Instead of `resmgr_attach()`

```c
// BEFORE (developer controls permissions):
resmgr_attach(dpp, &rattr, "/dev/ser1", ...);
// Developer hard-codes permissions → CAN'T know correct ones!

// AFTER (system integrator controls permissions via policy):
secpol_resmgr_attach(dpp, &rattr, "/dev/ser1", ...);
// Permissions come from the POLICY FILE
// System integrator decides user ID, group ID, permissions
```

**That's it!** These two calls are all a developer needs to add to support security policies.

---

## 7. What a Policy File Looks Like

### Example: Policy for the `random` Process

```
# ─── Define Security Types ───
type random_t              # Init type (high privilege)
type random_t__run         # Run type (low privilege)

# ─── INIT Phase Privileges ───
# When running as random_t (initialization):
allow random_t ability {
    map_phys           # Can map hardware memory
    public_channel     # Can create channels others connect to
    srandom            # Can push random data into kernel
    connect_slogger    # Can connect to system logger
}

# ─── Allow Transition ───
# random_t is allowed to become random_t__run
allow random_t settype {
    random_t__run      # Can transition to run type
}

# ─── What Pathnames Can Be Registered ───
allow_attach random_t {
    /dev/random        # Can register /dev/random in pathname space
}

# ─── Derived Type Rule ───
# When secpol_transition_type(NULL,NULL,0) is called:
# Take current type, append "__run"
derived_type random_t random_t__run

# ─── RUN Phase Privileges ───
# When running as random_t__run (normal operation):
allow random_t__run ability {
    srandom            # Can ONLY push random data to kernel
                       # Nothing else! 
                       # ❌ No more map_phys
                       # ❌ No more public_channel
                       # ❌ No more connect_slogger
}
```

### Visual Representation:
```
random process lifecycle:

  STARTS as random_t:
  ┌────────────────────────────────────┐
  │ Privileges:                         │
  │   ✅ map_phys (map hardware)        │
  │   ✅ public_channel (accept clients)│
  │   ✅ srandom (seed kernel RNG)      │
  │   ✅ connect_slogger (logging)      │
  │   ✅ Can register /dev/random       │
  │   ✅ Can transition to __run type   │
  │                                     │
  │ Does: maps HW, attaches interrupts, │
  │       registers /dev/random         │
  └──────────────┬─────────────────────┘
                 │
                 │ secpol_transition_type(NULL, NULL, 0)
                 ▼
  BECOMES random_t__run:
  ┌────────────────────────────────────┐
  │ Privileges:                         │
  │   ✅ srandom (seed kernel RNG)      │
  │   ❌ Everything else DROPPED!       │
  │                                     │
  │ If attacker compromises this        │
  │ process, they can ONLY push         │
  │ random seeds. They CANNOT:          │
  │   ❌ Map new hardware               │
  │   ❌ Attach to interrupts           │
  │   ❌ Register new pathnames         │
  │   ❌ Connect to other services      │
  │                                     │
  │ MINIMAL attack surface! ✅          │
  └────────────────────────────────────┘
```

---

## 8. secpolgenerate — Auto-Generate a Starting Policy

It's **hard** to figure out all the right privileges manually. `secpolgenerate` helps:

```
STEP 1: Run your system with ALL privileges (everything as root)
        Make sure all drivers and apps are doing their normal work.

STEP 2: Run secpolgenerate
        It OBSERVES everything:

        "random called mmap()              → needs map_phys"
        "random called InterruptAttach()   → needs interrupt"
        "random registered /dev/random     → needs pathspace"
        "devc-ser8250 called mmap()        → needs map_phys"
        "devc-ser8250 called InterruptAttach() → needs interrupt"
        ...

STEP 3: secpolgenerate outputs a TEXT POLICY FILE
        This is a STARTING POINT — not the final policy!

STEP 4: System integrator REVIEWS and EDITS:
        "This process doesn't really need X... remove it"
        "This process needs Y in edge cases... add it"

STEP 5: secpolcompile → secpolpush → ENFORCED!
```

**Important:** `secpolgenerate` gives you a **starting point**, NOT a final policy. The system integrator must review and tighten it.

---

## 9. secpollaunch — The Privileged Wedge

### The Problem:
```
Scenario: A process crashes and needs to be restarted.

Restarting requires HIGH privileges:
  - Launch new process
  - Assign it the right type
  - Give it access to hardware

But the WATCHDOG/MONITOR shouldn't have high privileges
permanently (security risk!)
```

### The Solution: secpollaunch
```
┌──────────────────┐     ┌───────────────────┐     ┌──────────────┐
│ Watchdog Monitor │     │ secpollaunch       │     │ Restarted    │
│ (LOW privilege)  │────→│ (privileged wedge) │────→│ Process      │
│                  │     │ Carefully written  │     │ (proper type)│
│ "Please restart  │     │ to safely launch   │     │              │
│  the serial      │     │ with correct       │     │              │
│  driver"         │     │ security type      │     │              │
└──────────────────┘     └───────────────────┘     └──────────────┘

secpollaunch has HIGH privilege BUT:
  - Only does ONE thing: launch processes securely
  - Carefully written and safety certified
  - Minimal attack surface
```

---

## 10. Complete Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    QNX SECURITY LAYERS                            │
│                                                                   │
│  Layer 1: POSIX File Permissions (rwx, user/group/other)         │
│           └─ Files, directories, devices                         │
│                                                                   │
│  Layer 2: Security Policies                                      │
│           └─ Types assigned to processes                         │
│           └─ Privileges per type (fine-grained)                  │
│           └─ Init→Run transitions (drop privileges)              │
│           └─ Pathname access control                             │
│                                                                   │
│  Layer 3: Memory Protection (Process Manager / MMU)              │
│           └─ Each process in isolated address space              │
│                                                                   │
│  Layer 4: Microkernel Architecture                               │
│           └─ Drivers in user space (crash ≠ system crash)        │
│                                                                   │
│  Layer 5: No qconn in production!                                │
│                                                                   │
│  CONTROLLED BY: System Integrator (NOT individual developers)    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Concept | Explanation |
|---------|------------|
| **Why policies?** | Developers CAN'T know final security needs; system integrator CAN |
| **Security Type** | A name (like `random_t`) assigned to a process, defining its privileges |
| **Two phases** | Init type (high privilege) → Run type (low privilege) via `secpol_transition_type()` |
| **Policy file** | Text file defining types, privileges, transitions, path access |
| **secpolcompile** | Compiles text policy → binary (safety qualified) |
| **secpolpush** | Loads binary policy into kernel (safety certified) |
| **secpolgenerate** | Auto-generates starting policy by observing system (helper tool) |
| **secpollaunch** | Privileged wedge for safely restarting processes |
| **Developer's job** | Just TWO calls: `secpol_transition_type()` and `secpol_resmgr_attach()` |
| **System integrator's job** | Write/edit the policy file that controls ALL processes' security |

> **In one sentence:** QNX security policies let the **system integrator** (not individual developers) define a **text-based policy** that assigns **security types** to processes, giving each one **only the privileges it needs** during initialization and then **dropping to minimal privileges** during normal operation, all enforced by the kernel at runtime.