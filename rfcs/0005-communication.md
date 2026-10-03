# RFC 0005: Communication

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-09-29
- **Last changed:** 2026-09-29
- **Supersedes / superseded by:** —

## Summary

How the parts of pnut-os talk. Requests and streams go over local stream
sockets; high-rate streams go through shared memory; events go through
uORB. Interfaces are defined once in `.proto` files. Messages that leave a
program are encoded with nanopb; inside a program and on uORB they stay
plain structs. Everything is asynchronous.

## Problem

[RFC 0004](0004-layers.md) fixed three kinds of communication (requests,
events and streams) and the rules they follow: bounded queues, caller
identity, versions, restarts. This RFC chooses the mechanisms, on a device
with little compute to spare.

## Proposal

### Requests and streams: local stream sockets

Each service listens on a local stream socket; a client connects and keeps
the connection open.

- **The caller is named by the kernel:** a service asks for the peer's
  credentials (`SO_PEERCRED`) and the manager maps the task to a program
  or app (RFC 0004).
- **Open files can be passed** over the socket (`SCM_RIGHTS`): this is how
  devices are lent and how shared memory for streams is handed over.
- **A service that dies drops its connections,** so its clients notice at
  once and reconnect.
- **Sockets are waited on in the event loop** with the program's other
  sources.

Both features above need NuttX's `NET_LOCAL_SCM`. A stream socket keeps no
message boundaries, so each message carries a small fixed header: its
length, the interface and method, a request number that matches a reply to
its request, and a status in replies.

### High-rate streams: shared memory

Data that is continuous and either fast or latency-bound, such as audio or
screen buffers, goes through shared memory. Every such stream is built the
same way:

1. **Set up over the socket.** The producer creates the shared memory
   (`memfd_create`, or `shm_open` and `mmap`) and passes it as an open
   file.
2. **A ring buffer** with one producer and one consumer, of fixed-size
   blocks holding plain structs.
3. **Signalled with `eventfd`:** "data available" and "space available",
   waited on in the event loop; no polling.
4. **The ring's free space is the credit:** the producer never writes past
   it. Each stream states what happens when the ring is full: live audio
   drops the oldest block, a recording makes the producer wait.
5. **One consumer per ring.** A second consumer gets a ring of its own,
   fed by the owner.

A stream's own details (formats, block sizes, latency) are set in the RFC
of the service that owns it. WebAssembly apps cannot map shared memory: the
runtime copies from the ring into the app.

Other streams run over the socket, with credit (RFC 0004).

### Events: uORB

Events are uORB topics: each topic is a device file with fixed-size
messages and a fixed queue depth; readers wait on it in the event loop,
can tell when they missed messages, and read the latest value when they
subscribe. A topic that carries state has a depth of one.

- **Any native program can read or publish any topic,** so topics carry
  nothing that needs permission to read.
- **WebAssembly apps never open topics.** The runtime subscribes for them
  and passes on only the topics the app's permissions allow.
- **Events meant for one client** (a reply, a private stream) go over that
  client's socket, not uORB.

### Finding a service

Each service has a **well-known socket path** derived from its name. The
manager knows which services are ready (RFC 0004). There is no registry.

### Interfaces: `.proto` files

Each interface is defined once, in a `.proto` file: its messages, its
methods (as a Protocol Buffers `service`), and for each method its version
and the permission it needs. From it a generator produces:

- the fixed-size C structs (nanopb, with a maximum size for every string
  and list);
- the asynchronous client and service code;
- the bindings the WebAssembly runtime gives apps;
- the uORB topic definitions;
- the reference documentation.

### Encoding

- **Leaving a program** (between programs, and to and from apps):
  messages are encoded with **nanopb**. Fields are identified by number, so
  a field can be added without breaking older readers; a field number is
  never reused and a field's type never changes. Apps in any language can
  read them with a standard Protocol Buffers library.
- **Inside a program:** a call is delivered to the other module directly,
  as the struct, with no socket and no encoding.
- **On uORB topics:** plain structs, as uORB requires.

### Everything is asynchronous

No part of the system waits for an answer: a request returns at once, and
its reply arrives later as an event on the caller's loop. This holds for
services, the system UI and apps alike. It is also what makes calls
between modules of one program possible: a module waiting for another on
the same loop would wait forever.

### Sparing the processor

- no copying inside a program: calls are delivered directly;
- no encoding inside a program or on uORB;
- state is merged, not queued: a reader wakes once and gets the latest
  value;
- bulk data goes through shared memory, not socket copies;
- work is batched, and timers are gathered so the processor wakes once
  instead of many times;
- long lists are paged, never sent whole.

## Measurements

nanopb 0.4.8 on the T-Deck Max (ESP32-S3 at 240 MHz, the rest of the
system running idle, code running from flash; averages of thousands of
repetitions, three or more runs; 2026-09-29):

| Message | Struct | Encoded | Copying the struct | Encode | Decode |
|---|---|---|---|---|---|
| Battery state (5 numbers) | 24 B | 25 B | 0.23 µs | 14.2 µs | 25.0 µs |
| SMS send request (140-character text) | 196 B | 159 B | 0.72 µs | 15.4 µs | 13.6 µs |
| Stored message (600-character text) | 712 B | 632 B | 2.1 µs | 37.6 µs | 28.8 µs |
| List of 10 contacts | 644 B | 360 B | 1.9 µs | 196 µs | 142 µs |

- A local socket round trip between two threads took **48–96 µs**, hardly
  depending on size (32–700 B).
- nanopb's code is **7.7 KB** (decoder 4.2 KB, encoder 2.8 KB, common
  0.8 KB), plus about **55 B** of tables per message type.

So encoding a typical message costs about as much as one socket round
trip: a real cost per call, but small in absolute terms (100 messages a
second is about 0.5% of the processor). Lists cost about 34 µs per entry.

## Alternatives

- **POSIX message queues** for requests. Message boundaries and a built-in
  bound, but no caller identity, no passing of open files, no sign when the
  other side dies, and a reply queue for every client.
- **Local datagram sockets.** Message boundaries, but no connection: no
  caller identity and no sign when a service dies.
- **Plain structs everywhere.** The cheapest, but changes must be
  append-only, and apps in other languages need a decoder of ours.
- **nanopb for every call, inside programs too.** One path, but encoding
  where both ends are the same code gains nothing.
- **FlatBuffers or CBOR.** Both in nuttx-apps; Protocol Buffers has the
  widest support in app languages.
- **JSON.** Readable, but the largest and slowest.
- **A registry service.** Introspection and a place for capabilities, but
  another service; well-known paths and the manager are enough.

## Costs and risks

- **Every message between programs is encoded and decoded:** about as
  much as the socket trip itself.
- **Long lists are expensive** unless paged.
- **nanopb drops fields it does not know** when a message is decoded and
  encoded again, so a program that forwards messages must be built with
  their current definition.
- **nanopb needs about 1 KB of stack** while encoding, by its own
  documentation; not measured here.
- **uORB topics are open to all native code.**
- **Streams to WebAssembly apps are copied** by the runtime.
