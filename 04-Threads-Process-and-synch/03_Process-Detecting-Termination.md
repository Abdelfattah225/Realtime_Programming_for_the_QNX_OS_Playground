

# Detecting Process Death in QNX — Three Methods

---

## Overview

| Method | Relationship | Standard? | Detects |
|--------|-------------|-----------|---------|
| **waitpid() / SIGCHLD** | Parent → Child | ✅ POSIX | Death of YOUR child processes |
| **Client/Server notification** | Client ↔ Server | ❌ QNX-specific | Death of your client or server |
| **Death pulses** | Any → Any | ❌ QNX-specific | Death of ANY process in the system |

---

## Method 1: Parent Detects Child Death (POSIX)

When a child process dies, the parent receives a **SIGCHLD signal**.

Unlike most signals, SIGCHLD does **NOT** terminate the parent — it's just a notification.

### Using waitpid():

```c
#include <sys/wait.h>
#include <spawn.h>

int main() {
    pid_t child_pid;
    
    // Launch some child processes
    posix_spawn(&child_pid, "/usr/bin/sensor_app", NULL, NULL, argv1, environ);
    posix_spawn(&child_pid, "/usr/bin/motor_app", NULL, NULL, argv2, environ);
    posix_spawn(&child_pid, "/usr/bin/logger", NULL, NULL, argv3, environ);
    
    // Now wait for any child to die
    while (1) {
        int status;
        pid_t dead_child = waitpid(-1, &status, 0);  // BLOCKS until a child dies
        
        if (WIFEXITED(status)) {
            // Child exited normally (called exit())
            printf("Child %d exited with status %d\n", 
                   dead_child, WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            // Child was killed by a signal (crash, etc.)
            printf("Child %d killed by signal %d\n", 
                   dead_child, WTERMSIG(status));
        }
        
        // Restart the dead child, log it, alert someone, etc.
    }
}
```

### What Happens Step by Step:

```
1. Parent spawns children:
   Parent (PID 1000) → spawns → sensor_app (PID 1001)
                      → spawns → motor_app  (PID 1002)
                      → spawns → logger     (PID 1003)

2. Parent calls waitpid() → BLOCKS (waiting)

3. motor_app (PID 1002) crashes! (divide by zero)
   
   Parent receives SIGCHLD signal (doesn't terminate parent)
   waitpid() RETURNS:
     dead_child = 1002
     status tells us: killed by signal (SIGFPE)

4. Parent handles it → restarts motor_app → calls waitpid() again
```

---

### The Zombie Problem

**What if the parent isn't waiting when a child dies?**

```
Timeline:
  Parent is busy handling motor_app's death...
  Meanwhile, logger (PID 1003) also dies!
  Parent hasn't called waitpid() yet for this one!

  ┌─────────────────────────────────────────────┐
  │ Logger (PID 1003) becomes a ZOMBIE           │
  │                                               │
  │ ❌ Not using CPU                              │
  │ ❌ Memory freed                               │
  │ ❌ File descriptors closed                    │
  │ ❌ Timers deleted                             │
  │ ✅ Process table entry STILL EXISTS           │
  │    → stores: which child (PID 1003)           │
  │    → stores: why it died (signal? exit code?) │
  └─────────────────────────────────────────────┘

  Later, parent calls waitpid():
    → Returns PID 1003 and why it died
    → Zombie is cleaned up and goes away completely
```

A zombie is like the movies — a **dead process still hanging around** just enough to deliver its death information.

---

### If You Don't Care About Child Deaths:

If you never plan to call `waitpid()`, prevent zombies from accumulating:

```c
// Tell the OS: "I don't care about child deaths, no zombies please"
signal(SIGCHLD, SIG_IGN);

// Now when children die:
//   ❌ No zombie created
//   ❌ No SIGCHLD delivered
//   Children just disappear completely
```

---

## Method 2: Client/Server Relationship Notification

If you have a **QNX message passing** relationship (client connected to server's channel), either side can detect when the other dies.

```
Server has a CHANNEL
Client is CONNECTED to that channel

What's really detected: the RELATIONSHIP going away
(not just the process dying)
```

```
┌────────────┐    connection    ┌────────────┐
│   Client   │ ──────────────→ │   Server   │
│            │                  │ (channel)  │
└────────────┘                  └────────────┘

Client dies → Server gets notified (connection gone)
Server dies → Client gets notified (server gone)
```

---

## Method 3: Death Pulses — Most Flexible

You can register to be notified when **ANY process** in the system dies — no parent/child or client/server relationship needed.

### How to Use:

```c
#include <sys/procmgr.h>

int main() {
    // Create a channel to receive pulses
    int chid = ChannelCreate(0);
    int coid = ConnectAttach(0, 0, chid, 0, 0);
    
    // Set up the pulse event
    struct sigevent event;
    SIGEV_PULSE_INIT(&event, coid, SIGEV_PULSE_PRIO_INHERIT, 
                     MY_DEATH_PULSE_CODE, 0);
    
    // Register: "Tell me when ANY process dies"
    procmgr_event_notify(PROCMGR_EVENT_PROCESS_DEATH, &event);
    
    // Now wait for death notifications
    while (1) {
        struct _pulse pulse;
        MsgReceivePulse(chid, &pulse, sizeof(pulse), NULL);
        
        if (pulse.code == MY_DEATH_PULSE_CODE) {
            pid_t dead_pid = pulse.value.sival_int;
            printf("Process %d just died!\n", dead_pid);
            
            // Take action: restart it, log it, alert someone, etc.
        }
    }
}
```

### What Happens:

```
Health Monitor registers for death notifications

System running:
  PID 1001: sensor_app
  PID 1002: motor_app
  PID 1003: logger
  PID 2000: health_monitor (registered for death pulses)

PID 1002 crashes!

  OS sends a PULSE to health_monitor:
    pulse.code = MY_DEATH_PULSE_CODE
    pulse.value = 1002  (PID of dead process)

Health monitor receives it:
  "Process 1002 died! That's motor_app. Restarting..."
  → posix_spawn() to restart motor_app
```

---

## Comparison

```
Method 1: Parent/Child (POSIX)
  ┌────────┐  SIGCHLD   ┌────────┐
  │ Parent │ ◄────────── │ Child  │ dies
  │        │  waitpid()  │        │
  └────────┘             └────────┘
  Only YOUR children. Must have parent/child relationship.

Method 2: Client/Server (QNX IPC)
  ┌────────┐  notification  ┌────────┐
  │ Client │ ◄────────────→ │ Server │ either side dies
  └────────┘                └────────┘
  Must have message passing connection.

Method 3: Death Pulses (Any process)
  ┌─────────────────┐  pulse   ┌──────────┐
  │ Health Monitor   │ ◄─────── │ ANY      │ dies
  │ (registered)     │          │ process  │
  └─────────────────┘          └──────────┘
  No relationship needed. Most flexible.
```

---

## Summary

| Method | When to Use |
|--------|------------|
| **SIGCHLD + waitpid()** | You spawned children and need to know when they die. Classic launcher pattern. |
| **Client/Server notification** | You have a message-passing connection and need to know if the other side disappears. |
| **Death pulses (procmgr_event_notify)** | You need a system-wide health monitor that watches ALL processes, regardless of relationship. |

> **Key points:** Zombies exist to preserve death information until the parent calls `waitpid()`. Use `signal(SIGCHLD, SIG_IGN)` if you don't care. For system-wide monitoring, **death pulses** are the most flexible — register once, get notified about **any** process death via a pulse containing the dead process's PID.