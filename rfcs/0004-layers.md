# RFC 0004: Layers and rules

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-09-29
- **Last changed:** 2026-09-29
- **Supersedes / superseded by:** —

## Summary

The layers of pnut-os, what each may do, and the rules every service
follows: who owns a device, which way calls go, when something may be a
service, how parts talk, how a program runs its modules, and how the
system copes with callers, versions, start-up, failure, power and full
queues.

## Problem

Without shared rules, a system like this grows unevenly: services multiply,
depend on each other in circles, compete for devices, and each solves
start-up, failure and overload its own way. These rules have to hold for
every service from the start, because changing them later means changing
every one.

## Proposal

### Terms

- **Program:** a NuttX task started from its own entry point, with its own
  open files; it can exit and be started again on its own. In the flat
  build ([RFC 0002](0002-build-mode.md)) programs share memory.
- **Module:** a part of a program with an interface of its own.
- **Service:** a module that owns a device or shared state and offers an
  interface to others.
- **Interface:** what a service offers: requests, events and streams, with
  a name and a version.

### Layers

From the top:

| Layer | What is in it | Reaches the layers below through |
|---|---|---|
| **Apps** | WebAssembly apps distributed by the pnut-os project; native apps installed by the device's owner | public interfaces only |
| **System UI** | the shell: screens, input, notifications on screen | public and system interfaces |
| **Services** | modules that own devices and shared state, and offer interfaces | system calls and device files only ([RFC 0002](0002-build-mode.md)) |
| **Kernel** | NuttX: scheduler, drivers, file systems, network stack | registers: the only layer that touches hardware |
| **Hardware** | the device ([T-Deck Max](../hardware/t-deck-max.md)) | — |

Each service offers a **public** interface, for apps, and may offer a
**system** interface, for the system UI and other services. The system UI
is a client of services and owns the screen and input.

### Rules

1. **Services are modules, grouped into a few programs.** Services are
   grouped into programs by domain. A service's interface is the same
   whether its caller is in the same program or another, so the grouping
   can change without touching callers. The grouping is defined with the
   list of services.
2. **Every device has exactly one owner.** One service owns each device;
   everyone else goes through its interface. The owner opens the device at
   start-up, and drivers answer a second open with "busy". For WebAssembly
   apps this is enforced: they see no devices. For native code it guards
   against accidents, not intent ([RFC 0002](0002-build-mode.md)).
3. **Devices can be lent.** A program that needs a device directly, such as
   a native app using the raw LoRa radio, asks its owner. The owner passes
   it the open device, tells its own clients the device is unavailable,
   stops using it, and takes it back when it is returned or the borrower
   exits.
4. **Calls go down; information comes up as events.** Services never
   depend on the system UI, apps never reach a system interface, and
   nothing reaches around a layer.
5. **A service exists only for a device or shared state** that several
   clients need, or that must outlive a client. Anything else is a library,
   or part of the app that needs it.
6. **Services are optional where the hardware is.** Without a LoRa radio or
   a modem, the system runs with less rather than failing.
7. **Three kinds of communication:** **requests** (do this, answer me),
   **events** (something changed; anyone may listen) and **streams**
   (continuous data to one party). The mechanism is decided in its own RFC.

### What every service does

- **Knows its caller.** Every request is identified: the kernel tells a
  service which task is on the other end of a local socket, and the
  manager (below) maps tasks to programs and apps. WebAssembly apps all
  run inside the runtime, so the runtime marks each call with the app's
  identity; services accept it because the runtime is trusted firmware.
- **Versions its interfaces.** Each interface has a version, and a service
  states which versions it serves.
- **Announces readiness,** and callers handle a service that is absent
  (rule 6).
- **Survives restarts.** Clients expect a service to disappear and come
  back: they reconnect and read its state again. A restarted service
  rebuilds its state.
- **Lets the device sleep.** No polling: everything is driven by events.
  A service that needs the device awake says so explicitly and briefly.
  Timers that must wake the device are kept by one service, not by each
  module.
- **Keeps its queues bounded.** Nothing grows without limit:
  - **events** work as in uORB: each topic has a fixed queue depth, the
    newest entry overwrites the oldest, and a reader can tell it missed
    some; a topic that carries state keeps only its latest value;
  - **requests** are bounded: when full, the answer is "busy" at once, and
    every request has a timeout;
  - **streams** use credit: the receiver says how much it can take, and
    the sender never sends more;
  - a client that stays behind is disconnected, reconnects and starts from
    the current state.

### Threads inside a program

Each program runs **one event loop**, and its modules run on it. Work that
can block or take long (flash, storage, heavy computation) goes to a
**small pool of worker threads**, and its result comes back to the loop as
an event.

- **A handler on the loop returns quickly;** anything that can block goes
  to a worker.
- **Workers never touch a module's state directly;** they hand results back
  to the loop, so modules need no locks.
- **The pool is small and fixed per program;** its size, and what
  "quickly" means, are set with the grouping of programs.

### The manager

One manager starts and watches the programs. Its duties:

- start programs in order, with their dependencies;
- restart a program that died, with a limit and a growing delay;
- know when each service is ready, and tell those who wait;
- stop programs;
- tell which task is which program or app, for caller identity.

Which manager is decided in its own RFC; NuttX's `nxinit` (in nuttx-apps)
is the candidate.

## Alternatives

- **One program per service.** Clear ownership and separate restarts, but
  every service costs a task, a stack and buffers from tight internal RAM,
  and the flat build does not isolate a crash anyway.
- **One program for all services.** Cheapest, but one crash stops every
  service, and services drift into each other's internals.
- **One event loop per program, without workers.** One stack per program,
  but one blocking call stalls every module in it (a storage commit on
  flash has taken seconds on this device).
- **One thread per module.** Plain sequential code, but a stack per
  module from tight RAM, and locks and ordering between modules.
- **Devices shared through their drivers.** No owner to go through, but
  every user has to agree on the device's state, and none can be told it
  has changed.
- **Hiding a device's file after opening it.** NuttX can remove the entry
  while the device stays open, but the driver then releases the device at
  its last close, so a restarted owner could not get it back; Linux does
  not work this way either.

## Costs and risks

- **Modules sharing a program can still break each other:** a fault in one
  stops the whole program, and in the flat build any native code can
  corrupt any other.
- **For native code the rules hold by convention,** checked in review, not
  enforced.
- **Events can be lost under load:** readers must notice missed entries
  and read the state again.
