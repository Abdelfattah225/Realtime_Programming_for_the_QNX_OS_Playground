# QNX Boot Process — Complete Explanation with Examples

---

## Overview: The Boot Sequence

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  BOARD/SOC VENDOR          │           QNX                        │
│  (not QNX)                 │           (your system)              │
│                            │                                      │
│  ┌─────────────────┐       │    ┌─────┐   ┌─────────┐   ┌─────┐ │
│  │ ROM Monitor     │       │    │     │   │         │   │Boot │ │
│  │ (uBoot/RedBoot) │──────────→│ IPL │──→│ Startup │──→│procnto│──→│Script│
│  │ or BIOS/UEFI    │       │    │     │   │         │   │     │ │
│  │ or Bare Metal   │       │    └─────┘   └─────────┘   └─────┘ │
│  └─────────────────┘       │                                      │
│                            │    ◄── All inside the IFS ──►        │
│                            │    (Image File System)               │
└──────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Board/SOC Vendor Code (Before QNX)

This is **NOT QNX** — it's provided by your board or chip manufacturer.

| Option | What It Is | When Used |
|--------|-----------|-----------|
| **uBoot / RedBoot** | ROM monitor (bootloader) | ARM embedded boards |
| **BIOS** | Basic Input/Output System | Older x86 systems |
| **UEFI** | Modern replacement for BIOS | Newer x86 systems |
| **Bare Metal** | Jump directly, no bootloader | Very minimal systems |

**Its only job:** Get the hardware minimally running and **jump to the IPL**.

### Example:
```
Power ON → Board ROM executes →
  uBoot loads → "Where is the QNX image?"
  → Finds it on SD card / flash / network
  → Jumps to IPL
```

---

## Step 2: IPL (Initial Program Load)

The IPL is the **first QNX code** that runs. It does the bare minimum to get the system ready:

### What IPL does:
```
1. Configure CLOCK FREQUENCIES
   → "CPU runs at 1.2 GHz, bus at 400 MHz"

2. Program CHIP SELECTS
   → "This address range maps to flash, this one to RAM"

3. Set up RAM
   → "RAM is working, we can now have a stack and run real code"

4. Find the IFS (Image File System)
   → "Found it at address 0x40000000 on flash"
   → Copy/map it so the next stages can access it
```

### Example:
```
IPL runs:
  ✅ Clocks configured
  ✅ RAM initialized → stack works now
  ✅ Found IFS on flash memory
  → Jump to STARTUP (inside the IFS)
```

---

## Step 3: The IFS (Image File System)

The IFS is a **single file/image** that contains everything QNX needs to boot:

```
┌──── IFS (Image File System) ────────────────┐
│                                               │
│  ┌──────────┐                                │
│  │ Startup  │  Board-specific HW init code   │
│  └──────────┘                                │
│                                               │
│  ┌──────────┐                                │
│  │ procnto  │  Microkernel + Process Manager │
│  └──────────┘                                │
│                                               │
│  ┌──────────────┐                            │
│  │ Boot Script   │  List of drivers/apps     │
│  │               │  to launch at boot        │
│  └──────────────┘                            │
│                                               │
│  ┌──────────────┐                            │
│  │ Other files   │  Drivers, apps, configs   │
│  │ (binaries)    │  needed at boot           │
│  └──────────────┘                            │
└───────────────────────────────────────────────┘
```

---

## Step 4: Startup (Board-Specific Code)

Startup runs **after IPL** and does **more detailed hardware initialization**. This is **custom code** written for your specific board.

### What Startup configures:

```
┌─────────────────────────────────────────────────┐
│  STARTUP configures and tells the OS about:      │
│                                                   │
│  📝 RAM         → How much? Where? Layout?        │
│  📝 Interrupts  → Where is the interrupt          │
│                    controller? What IRQs exist?   │
│                    Priority levels?                │
│  📝 Timers      → Which timer chips? Frequencies? │
│  📝 Reserved    → Graphics memory? DMA regions?   │
│     memory                                        │
│  📝 Clusters    → CPU core groupings              │
│                    (big.LITTLE, cache clusters)    │
└─────────────────────────────────────────────────┘
```

### Example:
```
Startup on a custom automotive board:

  "System has 4GB RAM starting at 0x80000000"
  "Interrupt controller is at 0xFEC00000, 256 IRQs"
  "System timer runs at 24 MHz"
  "Reserve 256MB at 0x90000000 for GPU graphics"
  "Reserve 16MB at 0xA0000000 for DMA transfers"
  "Cores 0-3 are high-performance cluster"
  "Cores 4-7 are low-power cluster"
```

---

### The System Page (Key Concept!)

All the information that Startup configures gets written into a **System Page** — a **read-only data structure** that is **mapped into every process** in the system.

```
┌─────────────────────────────────────────────────┐
│              SYSTEM PAGE                          │
│         (read-only, mapped everywhere)            │
│                                                   │
│  RAM info:        4GB at 0x80000000               │
│  CPU info:        8 cores, ARMv8                  │
│  Timer info:      24 MHz system timer             │
│  Interrupt info:  GIC at 0xFEC00000, 256 IRQs     │
│  Cluster info:    cluster0={0,1,2,3}              │
│                   cluster1={4,5,6,7}              │
│  Reserved memory: GPU=256MB, DMA=16MB             │
│  Boot time:       2024-01-15 08:00:00             │
│  ... more hardware details ...                    │
└─────────────────────────────────────────────────┘
         │         │         │         │
         ▼         ▼         ▼         ▼
     Process A  Process B  Process C  Driver D
     (can read) (can read) (can read) (can read)
     (can't     (can't     (can't     (can't
      write!)    write!)    write!)    write!)
```

### Why Does This Matter?

**"So that if a process needs to have access to some of this information, it could look at that if it has the appropriate rights."**

## Summary

| Stage | Who Provides It | What It Does |
|-------|----------------|--------------|
| **BIOS/uBoot** | Board/SOC vendor | Minimal HW init, jump to IPL |
| **IPL** | QNX (board-specific) | Clocks, chip selects, RAM, find IFS |
| **Startup** | QNX (board-specific) | Detailed HW init, write **System Page** (RAM, IRQs, timers, clusters) |
| **procnto** | QNX | Starts microkernel + process manager, runs boot script |
| **Boot Script** | YOU (developer) | Lists all drivers/apps/services to launch |
| **System Page** | Written by Startup | Read-only HW info mapped into ALL processes so they can query it without kernel calls |

> **In one sentence:** QNX boots through **IPL** (basic HW init) → **Startup** (detailed HW config written to a **System Page** readable by all processes) → **procnto** (microkernel + process manager) → **Boot Script** (where YOU specify exactly which drivers and apps to launch), giving you **complete control** over what runs on your embedded system.