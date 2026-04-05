# QNX Shared Objects — Explained Simply with Examples

---

## 1. What ARE Shared Objects?

A shared object is **code that gets loaded into a process at RUNTIME**, rather than being baked into the program at compile time.

```
STATIC LINKING (compile time):          SHARED OBJECT (runtime):
┌─────────────────────┐                ┌─────────────────────┐
│ my_program          │                │ my_program          │
│                     │                │                     │
│ ┌─────────────────┐ │                │ "I need libc.so"    │
│ │ copy of libc    │ │                │ "I need tcpip.so"   │
│ │ (baked in)      │ │                │                     │
│ │ BIG executable! │ │                │ SMALL executable!   │
│ └─────────────────┘ │                └─────────┬───────────┘
└─────────────────────┘                          │
                                        At launch time:
                                        Loader pulls in libc.so
                                        and tcpip.so from system
```

---

## 2. Two Ways to Use Shared Objects

### a) Linked at Compile Time (loaded automatically at launch)

You tell the compiler "I need this library." The system loader **automatically** loads it when the process starts.

```c
// You compile with:
// gcc my_app.c -o my_app -ltcpip

// This puts a note in the executable:
// "When launching, also load lib-tcpip.so"

// At process launch:
//   Loader sees the note → finds lib-tcpip.so → maps it into the process
//   Your code can call TCP/IP functions immediately

#include <tcpip.h>
int main() {
    connect(sock, &addr, sizeof(addr));  // Function from lib-tcpip.so
}
```

### b) DLL — Explicitly Loaded at Runtime (via dlopen)

Your program **manually** loads a shared object while it's already running.

```c
#include <dlfcn.h>

int main() {
    // Program is already running...
    
    // Step 1: Load the shared object
    void *handle = dlopen("my_plugin.so", RTLD_NOW);
    
    // Step 2: Find a function inside it
    void (*plugin_func)() = dlsym(handle, "do_something");
    
    // Step 3: Call it!
    plugin_func();
    
    // Step 4: Unload when done
    dlclose(handle);
}
```

**Use case:** Plugins! Load different functionality based on configuration without recompiling.

---

## 3. The BIG Benefit: Memory Sharing (Code Only!)

If **5 processes** all use `libc.so`, the system only keeps **ONE copy** of the code in physical RAM, mapped into all 5 processes as **read-only**.

### Example:
```
Process A    Process B    Process C    Process D    Process E
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│my_app_A│  │my_app_B│  │my_app_C│  │my_app_D│  │my_app_E│
│        │  │        │  │        │  │        │  │        │
│ libc ──│──│─ libc ─│──│─ libc ─│──│─ libc ─│──│─ libc ─│
│(mapped)│  │(mapped)│  │(mapped)│  │(mapped)│  │(mapped)│
└────┬───┘  └────┬───┘  └────┬───┘  └────┬───┘  └────┬───┘
     │           │           │           │           │
     └───────────┴───────────┼───────────┴───────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │  Physical RAM:       │
                  │  ONE copy of         │
                  │  libc.so CODE        │
                  │  (read-only)         │
                  └─────────────────────┘

WITHOUT shared objects: 5 copies × 2MB = 10MB RAM used
WITH shared objects:    1 copy × 2MB  =  2MB RAM used  ← HUGE savings!
```

---

## 4. The IMPORTANT Catch: Data is NOT Shared!

The **code** is shared (read-only). But any **global variables or data** inside the shared library becomes **private to each process**.

### Example:
```c
// Inside my_library.so:
int counter = 0;  // Global variable

void increment() {
    counter++;
}

int get_count() {
    return counter;
}
```

```
Process A loads my_library.so:
  calls increment() → counter = 1
  calls increment() → counter = 2
  calls get_count() → returns 2

Process B loads my_library.so:
  calls get_count() → returns 0  ← NOT 2!
  
  Process B has its OWN PRIVATE copy of "counter"
  It doesn't see Process A's changes!
```

### Why?
```
Physical RAM:
┌────────────────────────────────────────────────┐
│ libc.so CODE (shared, read-only)  ← ONE copy   │
├────────────────────────────────────────────────┤
│ Process A's libc DATA (counter=2) ← PRIVATE    │
├────────────────────────────────────────────────┤
│ Process B's libc DATA (counter=0) ← PRIVATE    │
└────────────────────────────────────────────────┘

CODE = shared (everyone reads the same instructions)
DATA = private (each process gets its own copy)
```

---

## 5. If You NEED to Share Data — Use Shared Memory

If processes using the same shared library need to share actual data, you must **explicitly** set up shared memory (just like any other inter-process data sharing):

```c
// Inside my_library.so:

#include <sys/mman.h>
#include <fcntl.h>

int *shared_counter;

void init_shared() {
    // Create shared memory explicitly
    int fd = shm_open("/my_counter", O_CREAT | O_RDWR, 0666);
    ftruncate(fd, sizeof(int));
    shared_counter = mmap(NULL, sizeof(int), 
                          PROT_READ | PROT_WRITE, 
                          MAP_SHARED, fd, 0);
    *shared_counter = 0;
}

void increment() {
    (*shared_counter)++;  // Now ALL processes see this change!
}
```

```
Process A calls increment() → shared_counter = 1
Process B calls increment() → shared_counter = 2  ← BOTH see 2!

Because now it's explicit shared memory,
not a private global variable.
```

---

## 6. Works Just Like Linux/Unix

The instructor emphasizes this is **very similar** to how Linux and other Unix systems handle shared objects (`.so` files). If you know Linux shared libraries, QNX works the same way.

```
Linux:    libc.so.6,  libpthread.so,  loaded by ld-linux.so
QNX:      libc.so,    lib-tcpip.so,   loaded by QNX loader

Same concept, same behavior.
```

---

## Summary

| Aspect | How It Works |
|--------|-------------|
| **What** | Code loaded at runtime, not baked in at compile time |
| **Two types** | Auto-loaded (linked with `-l`) or explicit (`dlopen`/`dlsym`) |
| **Code sharing** | ONE physical copy mapped read-only into ALL processes that use it → saves RAM |
| **Data sharing** | **NO!** Each process gets its own PRIVATE copy of global variables |
| **If you need shared data** | Use explicit shared memory (`shm_open` + `mmap`) |
| **Like Linux?** | Yes, very similar to Unix/Linux `.so` files |

> **In one sentence:** Shared objects let multiple QNX processes share **one read-only copy of library code** in RAM to save memory, but any **data/global variables** inside the library are **private per process** — if you need actual data sharing, you must use **explicit shared memory**.