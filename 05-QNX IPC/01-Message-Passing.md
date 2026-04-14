# QNX Message Passing (IPC) - README

## Overview

Message passing is a **QNX native IPC (Inter-Process Communication)** mechanism that follows a **client/server based model**. It can be thought of as a **Remote Procedure Call (RPC)** architecture.

### Key Characteristics
- **Bidirectional** communication between client and server
- **Completely synchronous** communication
- Server thread runs at the **same priority** as the client that sent the message
- Data is **copied by the kernel** between buffers (no pointer passing)

---

## Client/Server Communication Model

```
Client                        Server
  |                              |
  |------- MsgSend() ----------->|
  |                              | (processes message)
  |<------ MsgReply() -----------|
  |                              |
```

### Flow
1. Client sends a message via `MsgSend()`
2. Server receives the message via `MsgReceive()`
3. Server processes the message
4. Server replies to the client via `MsgReply()`
5. Client unblocks and becomes runnable again

---

## Message Passing Scenarios

### Scenario 1: Idle Server (Not Busy)

The server is in a **RECEIVE blocked** state — waiting for messages to arrive.

```
Client State: RUNNABLE
Server State: RECEIVE blocked (idle, available)

Steps:
1. Client calls MsgSend()
2. Server immediately receives via MsgReceive()
3. Kernel copies data from client's buffer → server's buffer
4. Client transitions to REPLY blocked state
5. Server processes the message
6. Server calls MsgReply()
7. Kernel copies reply data → client's reply buffer
8. Client becomes RUNNABLE again
```

**State Transitions:**
```
Client: RUNNABLE → REPLY blocked → RUNNABLE
Server: RECEIVE blocked → Processing → RECEIVE blocked
```

---

### Scenario 2: Busy Server

The server is **not available** to receive messages immediately (not in MsgReceive loop).

```
Client State: RUNNABLE
Server State: Busy (NOT in RECEIVE blocked)

Steps:
1. Client calls MsgSend()
2. No data copy occurs immediately (server unavailable)
3. Client transitions to SEND blocked state
4. Server becomes available and calls MsgReceive()
5. Kernel copies data from client's buffer → server's buffer
6. Client transitions from SEND blocked → REPLY blocked
7. Server processes the message and calls MsgReply()
8. Client receives reply and becomes RUNNABLE again
```

**State Transitions:**
```
Client: RUNNABLE → SEND blocked → REPLY blocked → RUNNABLE
Server: Busy → MsgReceive() → Processing → MsgReply()
```

---

## Channels and Connections

### Concept
- A **server** creates a **channel** to receive messages from clients
- A **client** connects to a server's channel using a **connection**
- A server only needs **one channel** to communicate with multiple clients

### Connection Scenarios

```
Client A ──┬──(pipe)──────────────────┐
           ├──(I/O socket)────────────┤──► Server C (Channel)
           └──(another connection)────┘

Client B ──────────────────────────────► Server C (Channel)
        └──────────────────────────────► Server D (Channel)
```

- A single client can have **multiple connections** to the same server
- A single client can connect to **different servers**
- Multiple clients can talk to a **single server channel**

---

## Server & Client Pseudo-Code

### Server Side

```c
// Step 1: Create a channel
chid = ChannelCreate(flags);

// Step 2: Enter receive loop
while (1) {
    // Wait for messages (RECEIVE blocked state)
    rcvid = MsgReceive(chid, &msg, sizeof(msg), NULL);

    // Step 3: Process the message
    process_message(&msg);

    // Step 4: Reply back to client
    MsgReply(rcvid, status, &reply, sizeof(reply));
}
```

### Client Side

```c
// Step 1: Connect to server's channel
coid = ConnectAttach(0, server_pid, server_chid, _NTO_SIDE_CHANNEL, flags);

// Step 2: Send message
status = MsgSend(coid, &msg, sizeof(msg), &reply, sizeof(reply));

// Step 3: Use the reply
use_reply(&reply);
```

---

## API Reference

### `ChannelCreate(flags)`

Creates a channel attached to the current process. Multiple threads within a server process may receive messages; the kernel picks whichever thread is available.

#### Flags

| Flag | Description |
|------|-------------|
| `_NTO_CHF_PRIVATE` | Creates a **private** channel for internal process use. Prevents external processes from connecting. A public channel requires appropriate ability sets. |
| `_NTO_CHF_COID_DISCONNECT` | Used when a server is **also a client**. Delivers a pulse when a connection ID (coid) becomes invalid. |
| `_NTO_CHF_DISCONNECT` | Delivers a **pulse** when a client disconnects or goes away, allowing the server to clean up client data/structures. |
| `_NTO_CHF_UNBLOCK` | Delivers a **pulse** when a REPLY-blocked client wishes to unblock (e.g., due to timeout or signal). |

---

### `ConnectAttach(nd, pid, chid, index, flags)`

Establishes a connection from the client to a server's channel.

```c
coid = ConnectAttach(0, server_pid, server_chid, _NTO_SIDE_CHANNEL, 0);
```

| Parameter | Description |
|-----------|-------------|
| `nd` | Node descriptor — set to `0` for local node |
| `pid` | Process ID of the server |
| `chid` | Channel ID of the server |
| `index` | Set to `_NTO_SIDE_CHANNEL` (see below) |
| `flags` | Additional flags |

---

### `MsgSend(coid, smsg, sbytes, rmsg, rbytes)`

Sends a message to a server and blocks the client until a reply is received.

```c
int status = MsgSend(coid, &send_msg, sizeof(send_msg), &reply_buf, sizeof(reply_buf));
```

| Parameter | Description |
|-----------|-------------|
| `coid` | Connection ID to the server |
| `smsg` | Pointer to the message to send |
| `sbytes` | Size of the outgoing message in bytes |
| `rmsg` | Buffer to hold the server's reply |
| `rbytes` | Size of the reply buffer in bytes |

**Returns:**
- `status` value on successful `MsgReply()` from server
- `-1` on error (with `errno` set)

---

### `MsgReceive(chid, msg, bytes, info)`

Server blocks waiting to receive a message from a client.

```c
int rcvid = MsgReceive(chid, &msg_buf, sizeof(msg_buf), &info);
```

| Parameter | Description |
|-----------|-------------|
| `chid` | Channel ID to receive on |
| `msg` | Buffer to store the received message |
| `bytes` | Maximum number of bytes to receive |
| `info` | Additional client information (`NULL` if not needed) |

**Returns:**
- `rcvid` — Receive ID used later in `MsgReply()` to identify the client

---

### `MsgReply(rcvid, status, msg, bytes)`

Server replies to the client identified by `rcvid`.

```c
MsgReply(rcvid, EOK, &reply_msg, sizeof(reply_msg));
```

| Parameter | Description |
|-----------|-------------|
| `rcvid` | Receive ID obtained from `MsgReceive()` |
| `status` | Acknowledgment status — **must be non-negative** for success (negative values are reserved for errors) |
| `msg` | Reply message buffer |
| `bytes` | Number of bytes to reply with |

> **Note:** If no data needs to be sent back (e.g., output-only operation), pass `NULL` and `0` for `msg` and `bytes`.

---

### `MsgError(rcvid, error)`

Used instead of `MsgReply()` when the server cannot handle the received message.

```c
MsgError(rcvid, ENOSYS);
```

| Parameter | Description |
|-----------|-------------|
| `rcvid` | Receive ID of the client to unblock |
| `error` | Error code (e.g., `ENOSYS`) |

**Effect:**  
- Unblocks the client
- `MsgSend()` on the client side returns `-1` with `errno` set to the specified error value

---

## Connection ID Types

There are **two types of Connection IDs (coids)**:

| Type | Description |
|------|-------------|
| **File Descriptor range** | Used by library functions like `open()` |
| **Side Channel range** | Used for explicit message passing connections |

### Why Use `_NTO_SIDE_CHANNEL`?

Specifying `_NTO_SIDE_CHANNEL` in `ConnectAttach()`:
- Ensures the coid falls into the **side channel range**, not the file descriptor range
- Prevents conflicts with standard I/O file descriptors
- Avoids potential **deadlock** conditions on `fork()`

---

## Data Copying Mechanism

The kernel **copies data directly** between client and server buffers. No pointers are shared.

```
Client Buffer                    Server Buffer
[  send data  ] ──(kernel copy)──► [  recv data  ]

Server Reply Buffer              Client Reply Buffer
[  reply data ] ──(kernel copy)──► [  reply data ]
```

### Copy Rules
- The kernel copies the **lesser** of the two sizes:
  - Bytes the client is sending
  - Bytes the server's buffer can accommodate

```
bytes_copied = min(client_send_bytes, server_recv_bytes)
```

> **Note:** If the server needs more data than what was received, it can use `MsgRead()` to retrieve additional data.

---

## Viewing System Information

### Using QNX IDE
1. Open the **QNX System Information** perspective
2. Expand processes and select a process
3. Navigate to the **Connection Information** view
4. To view side channels: click the **three dots (⋯)** and select **"Full Side Channels"**
   - Side channels appear in **hexadecimal** range values

### Using Command Line (Target)

#### View file descriptors and side channels
```bash
pidin fd | less
```

#### View all processes, threads, priorities, and states
```bash
pidin
```

**Output columns:** `PID`, `TID`, `Name`, `Priority`, `State`, `Block Parameter`

### Thread State Reference

| State | Block Parameter | Description |
|-------|----------------|-------------|
| `RECEIVE blocked` | `(chid)` | Channel ID the thread is blocked on |
| `REPLY blocked` | `(pid)` | Process ID of the server the thread is waiting for |
| `SEND blocked` | — | Client waiting for server to become available |

#### Example
```
PID   TID  Name       Priority  State           Blocked
1     1    procnto    255       RECEIVE blocked  (1)
2     1    myserver   10        REPLY blocked    (1)  ← waiting for procnto
```

---

## Summary Diagram

```
SERVER                                CLIENT
  │                                     │
  │  ChannelCreate()                    │  ConnectAttach(pid, chid)
  │       │                             │       │
  │  [RECEIVE blocked]                  │  [RUNNABLE]
  │       │                             │       │
  │       │◄────────── MsgSend() ───────────────┤
  │  MsgReceive()                       │  [REPLY blocked]
  │       │                             │
  │  [Processing]                       │
  │       │                             │
  │  MsgReply() ────────────────────────►  [RUNNABLE]
  │       │                             │
  │  [RECEIVE blocked]                  │
```