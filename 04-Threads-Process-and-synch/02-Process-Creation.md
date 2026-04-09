# QNX Process Creation — fork(), exec(), posix_spawn()

---

## The Three Ways to Create a Process

| Method | What It Does |
|--------|-------------|
| **fork()** | Creates a **copy** of the calling process |
| **exec()** | Loads a new program and **replaces** the caller |
| **posix_spawn()** | Loads a new program and **just runs it** (caller continues) |

---

## 1. fork() — Copy of Yourself

Child is an **identical copy** of the parent. Child does NOT start from `main()` — it **returns from fork()**.

### How to Tell Parent from Child:

```c
pid = fork();

if (pid > 0) {
    // PARENT executes this
    // pid = child's process ID (e.g., 5001)
    printf("I'm the parent. Child PID is %d\n", pid);
    
} else if (pid == 0) {
    // CHILD executes this
    // pid = 0
    printf("I'm the child!\n");
    
} else {
    // ERROR — fork failed
    printf("fork() failed!\n");
}
```

### What Happens Visually:

```
BEFORE fork():
┌──────────────────────────┐
│ Parent Process (PID 1000)│
│                          │
│ Code: [all program code] │
│ Data: [counter=5, x=10]  │
│                          │
│ → about to call fork()   │
└──────────────────────────┘

AFTER fork():
┌──────────────────────────┐    ┌──────────────────────────┐
│ Parent (PID 1000)         │    │ Child (PID 1001)          │
│                           │    │                           │
│ Code: [same code]         │    │ Code: [same code]         │
│ Data: [counter=5, x=10]  │───→│ Data: [counter=5, x=10]  │
│       (original)          │copy│       (COPY of parent's)  │
│                           │    │                           │
│ fork() returned 1001      │    │ fork() returned 0         │
│ → executes parent section │    │ → executes child section  │
└──────────────────────────┘    └──────────────────────────┘

From this point on, data segments may DIVERGE.
Parent modifies its data → child doesn't see it.
Child modifies its data → parent doesn't see it.
```

### fork() in a Multithreaded Process — AVOID!

The instructor warns strongly against this:

```
BEFORE fork():
┌──────────────────────────────────┐
│ Parent Process                    │
│                                   │
│ Thread 1: about to call fork()   │
│ Thread 2: CURRENTLY modifying    │
│           data segment!          │
│           [writing counter=5...  │
│            halfway done!]        │
└──────────────────────────────────┘

AFTER fork():
┌──────────────────────────┐    ┌──────────────────────────┐
│ Parent                    │    │ Child                     │
│                           │    │                           │
│ Thread 1 ✅               │    │ Thread 1 ONLY ✅          │
│ Thread 2 ✅               │    │ Thread 2 ❌ NOT inherited │
│                           │    │                           │
│ Data: [being modified     │    │ Data: [INCONSISTENT!      │
│        by Thread 2]       │    │  copied mid-modification] │
└──────────────────────────┘    └──────────────────────────┘

Child gets ONLY one thread (the one that called fork)
Child's data is a snapshot taken WHILE Thread 2 was mid-write
→ Data could be CORRUPT / INCONSISTENT!
```

### fork() Inheritance:

| Inherited ✅ | NOT Inherited ❌ |
|-------------|-----------------|
| Open file descriptors | Side channels |
| Priority & scheduling algorithm | Connections / Connection IDs |
| Signal mask | Channels |
| I/O privilege | Timers |
| User ID, Group ID | |
| Security Type ID | |
| Copy of parent's data segment | |

---

## 2. exec() — Replace Yourself with a Different Program

You give it a **program name**. That program gets loaded and **replaces** the caller entirely. Same PID, same process table entry — just different code.

```c
// Parent process (PID 1000) running my_launcher

printf("I'm about to be replaced...\n");

execl("/usr/bin/sensor_app", "sensor_app", "-config", "fast", NULL);

// THIS LINE NEVER EXECUTES (unless exec failed!)
printf("If you see this, exec() failed!\n");
```

### What Happens:

```
BEFORE exec():
┌──────────────────────────┐
│ PID 1000: my_launcher     │
│                           │
│ Code: [launcher code]     │
│ Data: [launcher data]     │
└──────────────────────────┘

AFTER exec():
┌──────────────────────────┐
│ PID 1000: sensor_app      │  ← SAME PID!
│                           │
│ Code: [sensor_app code]   │  ← Completely NEW code
│ Data: [fresh new data]    │  ← Completely NEW data
│                           │     (NOT a copy of parent)
│ Starts from main()        │
└──────────────────────────┘

my_launcher is GONE. Replaced. exec() does NOT return.
```

### exec() is a Family of Functions:

```c
execl()    // Pass args as a list
execv()    // Pass args as an array (vector)
execle()   // List + environment variables
execve()   // Vector + environment variables
execlp()   // List + search PATH
execvp()   // Vector + search PATH
```

### exec() Inheritance:

Same as fork() **EXCEPT**:
- **NO copy of parent's data** — it's a completely new program with fresh data
- File descriptors inherited **unless** the "close-on-exec" flag is set

```c
// Creator can control file descriptor inheritance:
int fd = open("/dev/ser1", O_RDWR);
fcntl(fd, F_SETFD, FD_CLOEXEC);  // Set "close on exec" flag

// When exec() happens:
// fd will be CLOSED automatically, NOT inherited
```

---

## 3. posix_spawn() — Just Run a Program (Recommended!)

You give it a program name. It **creates a new process** running that program. The caller **continues running**.

```c
pid_t child_pid;
char *argv[] = {"sensor_app", "-config", "fast", NULL};

// Launch sensor_app as a new process
int result = posix_spawn(&child_pid,           // returns child's PID here
                         "/usr/bin/sensor_app", // program to run
                         NULL,                  // file actions (optional)
                         NULL,                  // attributes (optional)
                         argv,                  // command line args
                         environ);              // environment

if (result == 0) {
    printf("Launched child with PID %d\n", child_pid);
    // Parent CONTINUES running here!
    // Child is running sensor_app independently
} else {
    printf("posix_spawn failed!\n");
}
```

### What Happens:

```
BEFORE posix_spawn():
┌──────────────────────────┐
│ PID 1000: my_launcher     │
│ (running)                 │
└──────────────────────────┘

AFTER posix_spawn():
┌──────────────────────────┐    ┌──────────────────────────┐
│ PID 1000: my_launcher     │    │ PID 1001: sensor_app      │
│ (STILL running!)          │    │ (NEW process, running!)   │
│                           │    │                           │
│ Continues its own code    │    │ Starts from main()        │
│ after posix_spawn()       │    │ Fresh new data            │
└──────────────────────────┘    └──────────────────────────┘

Both processes exist independently!
```

### Inheritance for posix_spawn():

Follows the rules of **fork() then exec()** combined — but done in a **single operation**.

---

## 4. Why posix_spawn() is Recommended Over fork()+exec()

### The Traditional Unix Way (1970s):

```c
// Step 1: fork() — create a copy of myself
pid_t pid = fork();

if (pid == 0) {
    // Child: I'm a copy of parent, but I don't want to be!
    // Step 2: exec() — replace myself with what I actually wanted
    execl("/usr/bin/sensor_app", "sensor_app", NULL);
}
```

### What Actually Happens:

```
Step 1: fork()
┌─────────────┐         ┌─────────────┐
│ Parent       │ ──copy──→ │ Child (copy │
│ (PID 1000)  │         │  of parent)  │
│              │         │  PID 1001    │
└─────────────┘         └──────┬──────┘
                               │
Step 2: exec()                 │ "Replace me with sensor_app"
                               ▼
                        ┌─────────────┐
                        │ sensor_app   │
                        │ PID 1001     │
                        │ (fresh start)│
                        └─────────────┘

WASTED WORK:
  ❌ Created a copy of parent (fork) — then THREW IT AWAY (exec)
  ❌ Copied data segment — then REPLACED it with new data
  ❌ Two process creations instead of one
  ❌ Multiple messages to process manager
```

### The posix_spawn() Way:

```
One step:
┌─────────────┐         ┌─────────────┐
│ Parent       │ ──spawn──→ │ sensor_app   │
│ (PID 1000)  │         │  PID 1001    │
│ continues    │         │ (fresh start)│
└─────────────┘         └─────────────┘

✅ ONE operation
✅ No wasted copy of data segment
✅ Single message to process manager: "run this"
✅ No intermediate process that gets torn down
✅ Works fine in multithreaded processes (unlike fork!)
```

### Important Note from the Instructor:

> "Don't be misled. You might think that posix_spawn() is implemented by doing a fork() and then an exec(), but it's not. posix_spawn() communicates directly with the process manager saying run this and the process manager just runs it."

---

## Comparison Table

| | fork() | exec() | posix_spawn() |
|---|--------|--------|---------------|
| **What it does** | Copies calling process | Replaces caller with new program | Creates new process running new program |
| **Caller continues?** | Yes (both parent and child run) | No (caller is replaced) | Yes (caller continues) |
| **Child starts from** | Returns from fork() | main() of new program | main() of new program |
| **Child's data** | Copy of parent's data | Fresh new data | Fresh new data |
| **Child's PID** | New PID | SAME PID as caller | New PID |
| **Returns** | Child's PID to parent, 0 to child | Does NOT return | Child's PID via parameter |
| **Multithreaded safe?** | ❌ Avoid! | ✅ Yes | ✅ Yes |
| **Efficiency** | Creates copy (wasteful if followed by exec) | Replaces in place | Single operation, most efficient |
| **POSIX?** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Recommended?** | Only when you need a copy | When you want to replace yourself | ✅ **YES — recommended for process creation** |

---

## Summary

> **posix_spawn() is the recommended way** to create processes in QNX because it's a **single, efficient operation** that sends one message to the process manager, avoids the wasted work of fork()+exec(), and works safely in multithreaded processes.