# QNX Security — Complete Explanation with Examples

---

## 1. POSIX File Permissions (The Basics)

QNX uses the **same file permission system as Linux/Unix** — the standard POSIX model.

### The Familiar Permission Model:
```bash
$ ls -l
-rwxr-xr--  1 john  engineers  4096  Jan 15 08:00  my_app
│││ ││││││
│││ │││└┘└─ Other:  r-- (read only)
│││ └┘└──── Group:  r-x (read + execute)
└┘└──────── User:   rwx (read + write + execute)
```

```
Permission applies to:     Files           Directories
─────────────────────     ─────           ───────────
r (read)                  Read content    List contents (ls)
w (write)                 Modify content  Create/delete files inside
x (execute)               Run as program  Enter directory (cd)
```

This works the same for **files, directories, AND devices** (like `/dev/ser1`).

---

## 2. The Boot Image Problem: Everything Runs as Root!

**By default**, everything in the IFS (Image File System) runs as **root** (UID 0).

```
Boot Script launches:
  devc-ser8250  → runs as ROOT ⚠️
  io-pkt        → runs as ROOT ⚠️
  my_app        → runs as ROOT ⚠️
  ksh           → runs as ROOT ⚠️

  EVERYTHING has FULL access to EVERYTHING!
```

### Why This Is Bad:
```
ROOT (UID 0) can:
  ✅ Access any file
  ✅ Kill any process
  ✅ Access any hardware
  ✅ Change any user ID
  ✅ Modify system settings
  ✅ Do literally ANYTHING

If a bug or attacker compromises ANY process → ENTIRE system is at risk!
```

### Good for Development, Bad for Production:
```
DEVELOPMENT:                        PRODUCTION:
┌──────────────────────┐           ┌──────────────────────┐
│ Everything as root   │           │ Each process gets     │
│ → Easy to develop    │           │ ONLY the permissions  │
│ → No permission      │           │ it actually needs     │
│   headaches          │           │ → "Principle of       │
│ → NOT SAFE!          │           │    Least Privilege"   │
└──────────────────────┘           └──────────────────────┘
```

---

## 3. How to Stop Running as Root

Several methods to drop from root:

### a) Create a Launcher Program
```c
// launcher.c — starts other programs as non-root users
#include <unistd.h>
#include <spawn.h>

int main() {
    // Drop to non-root user before launching the app
    setuid(1001);   // Switch to user "appuser" (UID 1001)
    setgid(100);    // Switch to group "appgroup" (GID 100)
    
    // Now launch the application — it inherits non-root identity
    execl("/usr/bin/my_app", "my_app", NULL);
}
```

### b) Use the `login` Utility
```bash
# Forces authentication before giving access
login    # Prompts for username/password
         # Shell runs as that user, NOT root
```

### c) Use `setuid()` C API
```c
// Inside your application, drop privileges after initialization
int main() {
    // Need root to bind to hardware initially
    init_hardware();  // Needs root access
    
    // Now drop to regular user — don't need root anymore!
    setuid(1001);
    setgid(100);
    
    // Rest of program runs as non-root
    main_loop();  // If compromised, limited damage
}
```

### d) Set the setuid Bit on an Executable
```bash
# Make an executable always run as a specific user
chmod u+s my_app     # setuid bit
chown specialuser my_app

# Now when ANYONE runs my_app, it runs as "specialuser"
# not as whoever launched it
```

---

## 4. REMOVE qconn in Production!

**qconn** is the Momentics IDE agent — it lets you debug, copy files, and control the target from your development PC.

```
qconn:
  ✅ Runs as ROOT by default
  ✅ Can launch debugger (pDebug)
  ✅ Can copy files to/from the system
  ✅ Can launch/kill processes
  ✅ Can inspect system state

  = BACKDOOR into your entire system!
```

```
DEVELOPMENT:                    PRODUCTION:
┌─────────────┐                ┌─────────────┐
│ qconn ✅    │                │ qconn ❌    │
│ (need it    │                │ NEVER ship  │
│  for debug) │                │ with qconn! │
└─────────────┘                └─────────────┘
```

---

## 5. Fine-Grained Abilities (Beyond Root/Non-Root)

### The Problem with Root/Non-Root:
```
Traditional Unix model:

  Root (UID 0):      Can do EVERYTHING        ← Too much power
  Non-root:          Can do almost NOTHING     ← Too little power

  What if a serial driver needs to:
    ✅ Access hardware (mmap)
    ✅ Handle interrupts
    ❌ But should NOT be able to kill other processes
    ❌ And should NOT be able to change user IDs

  With just root/non-root, you have to choose:
    Root → gives it interrupt access BUT ALSO lets it kill processes ⚠️
    Non-root → can't kill processes BUT ALSO can't access hardware ❌
```

### QNX Solution: `procmgr_ability()`

QNX breaks down "root powers" into **individual fine-grained abilities** that can be assigned **per process**:

```
ABILITY                          WHAT IT ALLOWS
────────                         ──────────────
PROCMGR_AID_INTERRUPT        →  Handle interrupts
PROCMGR_AID_IO               →  Direct I/O port access
PROCMGR_AID_MEM_PHYS         →  Map physical memory (mmap)
PROCMGR_AID_TIMER            →  Access/modify timers
PROCMGR_AID_SCHEDULE         →  Change thread priorities
PROCMGR_AID_SPAWN            →  Spawn new processes
PROCMGR_AID_KILL             →  Kill other processes
PROCMGR_AID_SETUID           →  Change user ID
PROCMGR_AID_TRACE            →  System trace/instrumentation
PROCMGR_AID_PROT_EXEC        →  Execute protected memory
... many more ...
```

### Example:
```
Serial Driver (devc-ser8250):
  ✅ PROCMGR_AID_INTERRUPT    (needs to handle serial interrupts)
  ✅ PROCMGR_AID_IO           (needs I/O port access)
  ✅ PROCMGR_AID_MEM_PHYS     (needs to mmap hardware registers)
  ❌ PROCMGR_AID_KILL         (should NOT kill other processes)
  ❌ PROCMGR_AID_SETUID       (should NOT change user IDs)
  ❌ PROCMGR_AID_SCHEDULE     (should NOT mess with priorities)

My Safety App (my_safety_app):
  ✅ PROCMGR_AID_SCHEDULE     (needs to set real-time priorities)
  ✅ PROCMGR_AID_KILL         (needs to restart failed processes)
  ❌ PROCMGR_AID_IO           (doesn't access hardware directly)
  ❌ PROCMGR_AID_MEM_PHYS     (doesn't need physical memory)
```

### How Root Relates to Abilities:
```
Root process      = ALL abilities enabled    (can do everything)
Non-root process  = NO abilities by default  (can do almost nothing)
Custom process    = SPECIFIC abilities only  (exactly what it needs)
                    ↑
                    THIS is what you want in production!
```

### Code Example:
```c
#include <sys/procmgr.h>

int main() {
    // Allow this process to handle interrupts
    procmgr_ability(0, 
        PROCMGR_ADN_ROOT | PROCMGR_AOP_ALLOW | PROCMGR_AID_INTERRUPT,
        PROCMGR_AID_EOL);
    
    // Deny this process from killing other processes
    procmgr_ability(0,
        PROCMGR_ADN_ROOT | PROCMGR_AOP_DENY | PROCMGR_AID_KILL,
        PROCMGR_AID_EOL);
    
    // Now this process CAN handle interrupts
    // but CANNOT kill other processes
}
```

---

## 6. Security Policies (The Best Way!)

Instead of manually setting abilities for each process, QNX provides a **policy-based system**.

### Step 1: Generate a Policy Automatically

```bash
# Run this on your minimal working system:
secpolgenerate
```

**What it does:**
```
secpolgenerate OBSERVES the running system:

  "devc-ser8250 used mmap()          → needs MEM_PHYS ability"
  "devc-ser8250 called InterruptAttach → needs INTERRUPT ability"
  "my_app called pthread_setschedparam → needs SCHEDULE ability"
  "devb-eide used mmap()             → needs MEM_PHYS ability"
  ...

Produces a TEXT FILE with all observed abilities:
```

### Step 2: Review and Edit the Policy File

```
# security_policy.txt (simplified example)

[serial_driver]
  path = /proc/boot/devc-ser8250
  abilities = interrupt, io, mem_phys

[disk_driver]
  path = /proc/boot/devb-eide
  abilities = interrupt, io, mem_phys, spawn

[safety_app]
  path = /usr/bin/my_safety_app
  abilities = schedule, kill

[user_app]
  path = /usr/bin/user_interface
  abilities = none

# You can EDIT this:
# - Remove abilities that seem unnecessary
# - Add abilities if you know they're needed later
# - It's just a text file!
```

### Step 3: Load at Boot Time

The policy file is **loaded by the kernel during boot**. Every process gets assigned its abilities automatically:

```
BOOT:
  Kernel loads security policy file
  
  devc-ser8250 starts → Kernel checks policy →
    Assigned: interrupt ✅, io ✅, mem_phys ✅, kill ❌, setuid ❌
    
  my_safety_app starts → Kernel checks policy →
    Assigned: schedule ✅, kill ✅, interrupt ❌, io ❌
    
  user_interface starts → Kernel checks policy →
    Assigned: (nothing) → most restricted ✅
```

### The Full Workflow:
```
┌────────────────────────────────────────────────────┐
│ DEVELOPMENT                                         │
│                                                     │
│  1. Build your system (everything runs as root)     │
│  2. Get it working properly                         │
│  3. Run secpolgenerate → observe → create policy    │
│  4. Edit policy file (remove unnecessary access)    │
│                                                     │
├────────────────────────────────────────────────────┤
│ PRODUCTION                                          │
│                                                     │
│  5. Include policy file in IFS                      │
│  6. Remove qconn                                    │
│  7. Stop running as root                            │
│  8. Kernel enforces policy at boot                  │
│  9. Each process gets ONLY what it needs            │
│                                                     │
│  RESULT: Secure system! ✅                          │
└────────────────────────────────────────────────────┘
```

---

## 7. How It All Fits Together

```
SECURITY LAYERS IN QNX:

Layer 1: POSIX File Permissions
         ├─ User/Group/Other
         ├─ Read/Write/Execute
         └─ Applied to files, directories, devices
         
Layer 2: Process Abilities (procmgr_ability)
         ├─ Fine-grained: interrupt, io, mmap, kill, schedule, etc.
         ├─ Per-process control
         └─ Much better than just "root or not root"
         
Layer 3: Security Policies (secpolgenerate)
         ├─ Auto-generated from system observation
         ├─ Editable text file
         ├─ Loaded by kernel at boot
         └─ Assigns abilities to processes automatically

Layer 4: Memory Protection (from Process Manager)
         ├─ Each process isolated in its own address space
         └─ Can't corrupt other processes
         
Layer 5: Microkernel Architecture
         ├─ Drivers in user space (not kernel)
         └─ Crashed driver doesn't crash system
```

---

## Summary

| Security Feature | What It Does |
|-----------------|--------------|
| **POSIX Permissions** | Standard rwx for user/group/other on files, dirs, devices |
| **Don't run as root** | Use launcher, login, setuid() to drop privileges |
| **Remove qconn** | NEVER ship with the debug agent in production |
| **procmgr_ability()** | Fine-grained abilities instead of root/non-root binary choice |
| **secpolgenerate** | Auto-generates a security policy by observing your running system |
| **Security Policy File** | Text file loaded at boot that assigns abilities per process |
| **Memory Protection** | Each process isolated — can't corrupt others |

> **In one sentence:** QNX security works in layers — **POSIX permissions** for file access, **fine-grained process abilities** (`procmgr_ability`) instead of just root/non-root, and a **security policy system** (`secpolgenerate`) that automatically observes your system and creates a policy file loaded at boot to ensure every process gets **only the minimum access it needs**.