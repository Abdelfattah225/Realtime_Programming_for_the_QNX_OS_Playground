# QNX Message Passing - Designing a Message Passing Interface 1


## Overview

Designing a message passing interface involves three key components:
- Defining **message types and structures**
- Creating **reply structures**
- Implementing **server-side message handling**

---

## Designing Message Structures

### Step 1: Create a Shared Header File

All message types and structures should be defined in a **shared header file** that is included by **both** the client and the server.

```
project/
├── messages.h       ← shared header (included by both)
├── server.c         ← includes messages.h
└── client.c         ← includes messages.h
```

### Step 2: Define Message Types and Structures

Each message must start with a **message type** field, followed by a structure that matches that type.

#### Option A: Related Messages — Use Types and Subtypes

Use **message types and subtypes** when messages share a common structure or are related to each other.

```c
// messages.h

#define MY_MSG_TYPE_A    0x200
#define MY_MSG_TYPE_B    0x201

typedef struct {
    uint16_t type;      // message type — always first field
    uint16_t subtype;   // message subtype
    // ... additional fields
} my_msg_t;
```

#### Option B: Unrelated Messages — Use a Union

Use a **union** when messages are unrelated to each other.

```c
// messages.h

typedef struct {
    uint16_t type;
} my_base_msg_t;

typedef struct {
    uint16_t type;
    int data;
} my_msg_a_t;

typedef struct {
    uint16_t type;
    char name[64];
} my_msg_b_t;

typedef union {
    my_base_msg_t  base;
    my_msg_a_t     msg_a;
    my_msg_b_t     msg_b;
} my_msg_union_t;
```

### Step 3: Define Matching Reply Structures

For each message type that expects a response, define a **corresponding reply structure**.

```c
// messages.h

typedef struct {
    int result;
    // ... reply fields
} my_reply_t;
```

---

## Message Types and Ranges

### ⚠️ Avoid Overlapping with QNX System Message Range

All QNX messages start with an **unsigned 16-bit integer (`uint16_t`)**.

| Range | Usage |
|-------|-------|
| `0` to `_IO_MAX` (0–511) | **Reserved** — QNX system messages |
| `512` to `65,000` | ✅ **Safe** — Use for your custom messages |

```c
// ❌ DO NOT use — overlaps with QNX system range
#define MY_MSG  0x01   // conflicts with system messages

// ✅ CORRECT — safely above _IO_MAX (511)
#define MY_MSG  0x200  // 512 decimal, safe range
```

### Why Avoid the System Range?

1. **Debugging Tools** — QNX event tracing tools interpret messages in the system range as system messages. Your custom data would be misinterpreted.

2. **Accidental System Messages** — If a QNX system message is accidentally sent to your process, your server can gracefully ignore it rather than mishandle it.

> **Rule of Thumb:** Always define your message types **above `_IO_MAX`** (greater than 511).

---

## Server-Side Message Handling

On the server side, after calling `MsgReceive()`, branch on the **message type** using a `switch` statement.

### Basic Server Message Loop

```c
#include <sys/neutrino.h>
#include "messages.h"

void server_loop(int chid) {
    my_msg_union_t  msg;
    my_reply_t      reply;
    struct _msg_info info;
    int rcvid;

    while (1) {
        // Block waiting for a message
        rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);

        if (rcvid == -1) {
            // Handle error
            break;
        }

        // Branch based on message type
        switch (msg.base.type) {

            case MY_MSG_TYPE_A:
                // Process message type A
                // ...
                MsgReply(rcvid, EOK, &reply, sizeof(reply));
                break;

            case MY_MSG_TYPE_B:
                // Process message type B
                // ...
                MsgReply(rcvid, EOK, &reply, sizeof(reply));
                break;

            default:
                // Unknown message type — return error
                MsgError(rcvid, ENOSYS);
                break;
        }
    }
}
```

### Server Message Handling Flow

```
MsgReceive()
     │
     ▼
 msg.base.type
     │
     ├──► MY_MSG_TYPE_A ──► Process A ──► MsgReply()
     │
     ├──► MY_MSG_TYPE_B ──► Process B ──► MsgReply()
     │
     └──► default (unknown) ───────────► MsgError(ENOSYS)
```

---

## Summary

| Step | Action | Location |
|------|--------|----------|
| 1 | Create message types/structures | `messages.h` |
| 2 | Define reply structures | `messages.h` |
| 3 | Include `messages.h` | Both `client.c` and `server.c` |
| 4 | Use `uint16_t` as first field in every message | `messages.h` |
| 5 | Keep message type values **above 511** (`_IO_MAX`) | `messages.h` |
| 6 | Use `switch` on message type after `MsgReceive()` | `server.c` |
| 7 | Reply with `MsgReply()` or `MsgError()` | `server.c` |