# QNX Message Passing - Client Information Structure

## Overview

When a server calls `MsgReceive()`, the **fourth parameter** provides detailed information about the client that sent the message. This parameter is of type `struct _msg_info` and is filled in by the kernel at the time of message receipt.

```c
// Basic usage (info ignored)
rcvid = MsgReceive(chid, &msg, sizeof(msg), NULL);

// With client info
struct _msg_info info;
rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);
```

> **Important:** The `_msg_info` structure is **only updated for messages**, not for pulses. If a pulse was received, the structure may contain stale data from a previous client or garbage values.

---

## The `_msg_info` Structure

The kernel automatically copies client information into this structure during `MsgReceive()`. It is the **preferred** way to obtain client information — using a separate `MsgInfo()` call adds an unnecessary extra kernel call.

```
┌──────────────────────────────────────────────┐
│              struct _msg_info                │
├──────────────┬───────────────────────────────┤
│   pid        │ Client Process ID             │
│   tid        │ Client Thread ID              │
│   chid       │ Channel ID                    │
│   scoid      │ Server Connection ID          │
│   coid       │ Connection ID                 │
│   priority   │ Client's sending priority     │
│   flags      │ Message flags                 │
│   srcmsglen  │ Source message length         │
│   dstmsglen  │ Destination message length    │
└──────────────┴───────────────────────────────┘
```

---

## Fields and Their Meaning

| Field | Description |
|-------|-------------|
| `pid` | **Process ID** of the client that sent the message |
| `tid` | **Thread ID** of the client thread that made the `MsgSend()` call |
| `chid` | **Channel ID** the message was received on |
| `scoid` | **Server Connection ID** — server-side index used to track client connections |
| `coid` | **Connection ID** used by the client |
| `priority` | **Priority** at which the client is sending the message |
| `flags` | Message flags |
| `srcmsglen` | **Source message length** — how many bytes the client is sending |
| `dstmsglen` | **Destination message length** — how many bytes the client's reply buffer can hold |

### Understanding `scoid`

The `scoid` (Server Connection ID) is a **server-side identifier** for a client connection:
- Acts as an **index** into the server's internal data structures
- Maintained by the server for tracking active client connections
- Must be **freed/cleaned up** when a client disconnects
- Can be used for **client authentication**

---

## Message Length Information

The `srcmsglen` and `dstmsglen` fields allow the server to understand data sizing on both ends of the exchange.

### Source Message Length (`srcmsglen`)

How many bytes the **client is sending** to the server.

```
Client sends:     100 bytes  (srcmsglen = 100)
Server buffer:    200 bytes  (can accommodate)

Kernel copies: min(100, 200) = 100 bytes  ✓
```

### Destination Message Length (`dstmsglen`)

How many bytes the **client's reply buffer** can hold.

```
Server wants to reply: 150 bytes
Client reply buffer:   200 bytes  (dstmsglen = 200)

Kernel copies: min(150, 200) = 150 bytes  ✓
```

### Visual Flow

```
CLIENT                          KERNEL                         SERVER
  │                               │                              │
  │── MsgSend(msg, 100 bytes) ───►│                              │
  │                               │── copies min(src, dst) ─────►│
  │                               │   into server buffer         │
  │                               │                              │  info.srcmsglen = 100
  │                               │                              │  info.dstmsglen = 200
  │                               │                              │  (server checks sizes)
  │                               │◄─ MsgReply(reply, 150 bytes)─│
  │◄── reply arrives (150 bytes) ─│                              │
```

> **Rule:** Always ensure the client's reply buffer is large enough to hold the full reply from the server.

---

## Use Cases

### 1. Client Tracking with `scoid`

The server uses `scoid` as an index to manage per-client data structures.

```c
struct _msg_info info;
rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);

// Use scoid to look up client-specific data
client_data_t *client = lookup_client(info.scoid);
```

### 2. Client Authentication

The server can use `_msg_info` fields to **accept or reject** specific clients.

```c
struct _msg_info info;
rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);

// Restrict access by process ID
if (!is_authorized_client(info.pid)) {
    MsgError(rcvid, EPERM);
    continue;
}

// Limit number of allowed clients
if (info.scoid > MAX_CLIENTS) {
    MsgError(rcvid, ECONNREFUSED);
    continue;
}
```

### 3. Reply Space Verification

Before replying, the server checks that the client's reply buffer is large enough.

```c
struct _msg_info info;
rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);

// Verify client has enough space for the reply
if (info.dstmsglen < sizeof(my_reply_t)) {
    MsgError(rcvid, EMSGSIZE);
    continue;
}

MsgReply(rcvid, EOK, &reply, sizeof(reply));
```

### 4. Debug Logging

Use `pid` and `tid` for detailed server-side debug logs.

```c
struct _msg_info info;
rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);

// Log client info for debugging
log_message("Received from PID=%d, TID=%d, Priority=%d",
            info.pid,
            info.tid,
            info.priority);
```

---

## Additional Client Information

For **extended client details** (user ID, group ID, privileges), use the `ConnectClientInfo()` call with the `scoid` from `_msg_info`.

```c
struct _client_info client_info;

// Get extended info using scoid
ConnectClientInfo(info.scoid, &client_info, NGROUPS_MAX);

// Now you have access to:
// client_info.cred.uid  — User ID
// client_info.cred.gid  — Group ID
// client_info.cred.euid — Effective User ID
// + privilege/ability information
```

### Common Uses of `ConnectClientInfo()`

| Information | Field | Use Case |
|-------------|-------|----------|
| User ID | `cred.uid` | Restrict access by user |
| Group ID | `cred.gid` | Restrict access by group |
| Effective UID | `cred.euid` | Check root privileges |
| Abilities/Permissions | ability fields | Fine-grained privilege checking |

---

## Complete Example

```c
#include <sys/neutrino.h>
#include <sys/iomsg.h>

void server_loop(int chid) {
    my_msg_t         msg;
    my_reply_t       reply;
    struct _msg_info info;
    int              rcvid;

    while (1) {
        // Receive with full client info
        rcvid = MsgReceive(chid, &msg, sizeof(msg), &info);

        if (rcvid == -1) {
            break;  // error
        }

        if (rcvid == 0) {
            // Pulse received — info is NOT valid here
            handle_pulse(&msg.pulse);
            continue;
        }

        // Message received — info IS valid
        // ─── Authentication ───────────────────────────
        if (!is_authorized(info.pid)) {
            MsgError(rcvid, EPERM);
            continue;
        }

        // ─── Reply space check ─────────────────────────
        if (info.dstmsglen < sizeof(reply)) {
            MsgError(rcvid, EMSGSIZE);
            continue;
        }

        // ─── Debug logging ─────────────────────────────
        log_message("MSG from PID=%d TID=%d scoid=%d priority=%d srclen=%d dstlen=%d",
                    info.pid, info.tid, info.scoid,
                    info.priority, info.srcmsglen, info.dstmsglen);

        // ─── Process and reply ─────────────────────────
        process_message(&msg, &reply);
        MsgReply(rcvid, EOK, &reply, sizeof(reply));
    }
}
```

---

## Summary

| Method | When to Use | Extra Kernel Call? |
|--------|-------------|-------------------|
| `MsgReceive()` fourth parameter | Basic client info (pid, tid, scoid, lengths) | ❌ No — filled automatically |
| `MsgInfo()` | When info is needed after `MsgReceive()` | ✅ Yes — additional overhead |
| `ConnectClientInfo()` | Extended info (UID, GID, privileges) | ✅ Yes — use only when needed |

> **Best Practice:** Always pass a valid `struct _msg_info` pointer to `MsgReceive()` rather than `NULL` when you need client information. Avoid `MsgInfo()` as it adds an unnecessary kernel call.