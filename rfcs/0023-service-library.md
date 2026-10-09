# RFC 0023: The service library

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-08
- **Last changed:** 2026-10-09
- **Supersedes / superseded by:** —

## Summary

The code every program shares, `libpnut`: an event loop on `epoll`, a
fixed pool of workers, the message header and status codes, clients that
time out and reconnect, caller identity, readiness, and memory from fixed
pools. Its client and service code is generated from the `.proto` files.
Its NuttX-specific parts sit behind small interfaces, so it builds on
Linux for tests. This RFC fixes the library's shape; its details (the
header's fields, the status codes, the values) are a first proposal,
expected to move as the code is written.

## Problem

The RFCs so far fix how programs behave, not the code they do it with:

- one event loop and a small pool of workers per program; handlers return
  quickly; modules need no locks ([RFC 0004](0004-layers.md), sizes in
  [RFC 0007](0007-services.md));
- local stream sockets with a small fixed header, nanopb between
  programs, plain structs inside one, uORB for events, everything
  asynchronous and bounded ([RFC 0005](0005-communication.md));
- `ready` reported to nxinit directly, states mirrored by the Service
  states module, caller identity from `SO_PEERCRED` and `who`
  ([RFC 0006](0006-manager.md));
- sockets at well-known paths under `/var/run`
  ([RFC 0010](0010-file-layout.md)).

Every service would otherwise write these again. NuttX always builds
`epoll`; `eventfd`, `timerfd` and `signalfd` are options; nuttx-apps has
a package for nanopb, which downloads it, unchecked; uORB topics are
file descriptors.

## Proposal

### What is fixed, and what will move

This is the first RFC written for code about to be started, and the code
will teach. Two kinds of things are in it:

- **The shape,** which this RFC decides: one loop on `epoll` and a fixed
  pool of workers; client and service code generated from the `.proto`
  files; a fixed header; bounded everything, with memory from fixed
  pools; a library that builds on Linux.
- **The details,** which are a first proposal: the header's exact fields,
  the status codes, the starting values. They are expected to change
  while the library is written. The library's reference documentation,
  next to the code ([RFC 0003](0003-repositories.md)), is where they are
  kept once implemented; this RFC is then corrected to match, as the
  process allows for changes that do not touch a decision.

### The library

**`libpnut`** is one static library, in `lib/`, linked into every program.

- **A program's `main`** creates the loop, adds its modules, and runs it.
- **A module** declares the interfaces it serves, the topics it
  subscribes to, and its timers; the library calls its handlers on the
  loop.
- **Readiness:** the program reports `ready` to nxinit (RFC 0006) once
  every module has said it is ready; a module that depends on another
  service waits for its state on the Service states topic.
- **Stopping:** nxinit's `SIGTERM` arrives on the loop as an event
  (`signalfd`); the library tells every module, then exits.

### The loop

The loop waits with **`epoll`** on everything a program listens to:

| Source | Through |
|---|---|
| sockets: the services' listening sockets, connections, clients | the socket |
| uORB topics | the topic's file descriptor |
| workers' results, and shared-memory rings (RFC 0005) | `eventfd` |
| timers | `timerfd` |
| signals | `signalfd` |

NuttX's `EVENT_FD`, `TIMER_FD` and `SIGNAL_FD` options go into the
configuration fragment ([RFC 0022](0022-build.md)). A handler returns
within its budget (RFC 0007); the library measures each handler and logs
the ones that run over.

### Code from the `.proto` files

| Generator | Writes |
|---|---|
| **nanopb's,** as packaged | the message structs, with a maximum size for every string and list |
| **the project's,** in `tools/`, in Python | the client and service code for each interface, the runtime's bindings ([RFC 0008](0008-runtime.md)), the uORB topic definitions, and the reference documentation |

Both read the same `.proto` files. The project's generator reads **custom
options:** a method's number, version and permission; a topic's message
and queue depth. Generated code is made at build time and never committed;
`protoc` and nanopb's plugin are in the container image (RFC 0022).

- **nanopb,** at a release the build pins and checks against its SHA-256
  (RFC 0022): the version lives there, and moves on purpose. nuttx-apps'
  package and the unit tests build from that copy. Its generator writes
  names in C style (`settings_get_request_t`, not `settings_GetRequest`),
  as NuttX's style wants.
- **A program keeps a server object** for each interface it serves, and a
  **client object** for each connection it makes, statically or in a
  module's state. They hold the buffers a request or an answer is decoded
  into and encoded from, sized from nanopb's maximum sizes, so nothing is
  allocated and nothing large goes on the stack.
- **Handlers and calls are typed:** a server's handler receives the
  decoded request; a call's handler, the decoded answer. A decoded body
  is valid while its handler runs; a request answered later is kept as a
  copy. A method a server leaves without a handler is answered
  `not found`.
- **Every message has bounds:** a string or a list without a maximum size
  does not build, so every message has a largest encoding, which the
  answers' room (below) relies on.

**Inside a program,** a call to another module is delivered as the struct,
with no socket and no encoding (RFC 0005): the generator knows the
grouping of services into programs (RFC 0007), and writes the direct path
where both ends share a program.

### The header

Every message on a socket starts with a **16-byte header, little-endian.**
A first layout:

| Bytes | Field |
|---|---|
| 1 | header version |
| 1 | kind: request, reply, event, stream data |
| 2 | interface |
| 2 | method |
| 1 | the method's version (RFC 0005) |
| 1 | status, in replies (below) |
| 2 | sequence number: a reply carries its request's |
| 2 | payload length |
| 2 | app, on the runtime's connections (RFC 0005, RFC 0008) |
| 2 | reserved |

- **Interface and method numbers** are assigned in the `.proto` files, as
  field numbers are, and never reused; one file in `proto/` lists the
  interfaces and their numbers.
- **A message is at most 4 KB,** header included; anything larger is
  paged or streamed (RFC 0005).

### Status codes

A first set:

| Code | Means |
|---|---|
| ok | done |
| busy | the queue is full; try later (RFC 0004) |
| denied | the caller lacks the permission |
| not found | no such interface, method, or thing |
| invalid | the request does not parse, or asks for the impossible |
| unavailable | the service is not ready, or its hardware is absent |
| timeout | no reply in time |
| internal | the service failed |

**An error's detail.** The body of an error answer may be one common
message, `pnut.Error` (`proto/pnut/error.proto`). Its `code` is one of the
interface's own numbers, listed as an enum in its `.proto`; 0 means none.
Its `text` is short, at most 63 bytes; a longer one is cut at a
character's start. The code tells programs which error of the status's
kind it was; the text is for the log and developers, never shown on a
screen, where text is translated ([RFC 0017](0017-i18n.md)), and never
carries the user's data ([RFC 0026](0026-log.md)). A server gives it with
the generated `*_fail()`; a caller's handler receives it beside the
status, when the server gave one. A detail that does not decode is
dropped, and the status stands. Every method's room for its answer
(below) counts it.

### Services

- **Room for every answer.** Each method declares the largest answer it
  gives (its answer's largest encoding, or an error's detail's, whichever
  is larger, from the `.proto`). A connection
  takes a request only with that much room kept in its send buffer, until
  the answer is given, now or later; so an answer never finds the buffer
  full, however slowly the caller reads. The requests after one with no
  room yet wait in the socket.
- **An answer larger than its method declared** is refused
  (`-EMSGSIZE`), so it cannot take room kept for another. An event takes
  only room that is not kept.
- **Requests in flight** per connection are bounded; one more is answered
  `busy` at once. Every request is answered, and once: one never answered
  keeps its room until the connection closes.
- **Connections per caller** are bounded: a program, known by
  `SO_PEERCRED`, holds at most a few of a service's connections; one more
  is closed as soon as it is accepted, and logged. So one program cannot
  take every connection and lock the others out. A caller whose pid
  cannot be read is not counted, since it cannot be told from the others.
  Apps reach services through the runtime's one connection each
  (RFC 0008), so this is about native programs.

### Clients

- **Every request has a timeout,** 5 s unless the call says otherwise;
  a reply that comes later is dropped, and the caller got `timeout`.
- **When a service dies,** its connections drop (RFC 0005). The library
  reconnects with a growing delay, watching the service's state on the
  Service states topic, and tells the module when the connection is back,
  so it reads the service's state again (RFC 0004). **The delay starts
  again from the shortest only after a connection that lasted:** a
  service that closes a connection as soon as it is accepted, because it
  has no room for it, sees the delay grow, not a try every 100 ms.
- **A client keeps one timer,** for its next try while it is not
  connected and for its calls' deadlines while it is; so the timers' pool
  holds one per client, whatever its calls.

### Caller identity

A service learns the task behind a connection from `SO_PEERCRED` when it
accepts it, and asks the Service states module once which program it is
(RFC 0006). On a connection from the runtime, the header's app field
names the app (RFC 0008); the library accepts the field only there.
Both reach a handler with every request.

### Workers

- **A fixed pool** per program, sized in RFC 0007, with a **bounded job
  queue;** a full queue answers `busy`.
- **A job gets copies of its input,** never a module's state, and posts
  its result to the loop through an `eventfd`; the loop calls the module
  with it (RFC 0004).
- **Stacks of at least 4 KB** (RFC 0007).

### Memory

Everything a program needs comes from **fixed pools, sized at build time**
in the configuration fragment: connections, requests in flight, jobs,
buffers, timers. `malloc` is used only while a program starts, never in a
handler. When a pool is empty, the answer is `busy` or `unavailable`, not
a failed allocation. Internal RAM is scarce and fragments; this is what
keeps a program up for weeks.

### Building on the computer

The library's NuttX-specific parts (uORB, nxinit's socket, device files)
sit behind small interfaces, with a Linux side for each. `epoll`,
`eventfd`, `timerfd` and `signalfd` exist on Linux as they are. So the
library and the services' logic build on Linux, for the unit tests and
the light runner ([RFC 0021](0021-sdk.md), RFC 0022).

### Starting values

Set for each program in the fragment; to be measured:

| What | Value |
|---|---|
| A handler's budget | 10 ms (RFC 0007) |
| A request's timeout | 5 s |
| Requests in flight per connection | 8 |
| Reconnecting | after 100 ms, doubling to 5 s |
| A message | 4 KB at most |
| Connections per service | 8 |
| Connections per caller | 2 |
| A connection's send buffer | 8 KB |

## Alternatives

- **libuv,** which nuttx-apps has and `adbd` uses. A proven loop, but its
  handles are allocated on the heap, against fixed pools, and it knows
  nothing of uORB.
- **Dispatch written by hand,** with only nanopb generated. Less tooling,
  but every interface would repeat the same code, and the bindings and
  documentation would drift.
- **Committing the generated code.** No `protoc` at build time, but every
  `.proto` change would land twice.
- **Interface numbers hashed from names.** Nothing to assign, but a
  collision would be found only when two interfaces met.
- **`malloc` in handlers.** Simpler code, but fragmentation of scarce RAM
  over weeks, and failures nowhere to answer `busy`.
- **A NuttX-only library.** One platform to write for, but every test
  would need the simulator.
- **A whole message's room kept for every request,** instead of each
  method's declared answer. Nothing to declare, but 4 KB per request: with
  8 KB buffers, a service that answers later would stop reading after
  two.
- **Closing a connection whose answer does not fit.** Simple, but a slow
  caller is not a dead one, and every call it had in flight would fail.
- **An error message for each method,** declared with an option. Typed
  details, but more to declare and to generate, for no interface that
  needs it yet; it can come later, beside the common one.
- **No error detail at all,** with results in the answer's own fields.
  Nothing to add, but every interface would invent its own way to say
  which error it was.
- **Idle connections closed after a while,** against one program holding
  many. But a connection that only waits for events is idle, and would be
  closed for doing its job.

## Costs and risks

- **A generator to keep,** and `protoc` in every build.
- **Fixed pools hold memory while idle,** and refuse work when full; their
  sizes need measuring on each device.
- **Two paths to test** for every call: inside a program, and across
  programs.
- **16 bytes on every message,** about a tenth of a typical one
  (RFC 0005).
- **The Linux side** of each interface is one more thing to keep in step
  with the NuttX side.
- **Every message needs bounds,** which the generator enforces; an answer
  larger than declared is refused at run time, a bug found by its caller's
  `timeout`.
