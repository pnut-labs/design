# RFC 0006: The manager

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-03
- **Last changed:** 2026-10-03
- **Supersedes / superseded by:** —

## Summary

The manager is NuttX's `nxinit`, extended in our nuttx-apps fork with what
it lacks: readiness, restart limits with a growing delay, and a control
socket through which programs ask it to start and stop services, report
that they are ready, and find out which task is which program. One pnut-os
module speaks to it and offers the rest of the system a normal interface.

## Problem

[RFC 0004](0004-layers.md) gives one manager these duties:

- start programs in order, with their dependencies;
- restart a program that died, with a limit and a growing delay;
- know when each service is ready, and tell those who wait;
- stop programs;
- tell which task is which program or app, for caller identity.

`nxinit`, in nuttx-apps, is a small init (about 3,300 lines, with unit
tests) that reads `/etc/init.d/init.rc` in Android's init language. At
13.1.0-RC0 it covers part of the list:

| Duty | `nxinit` today | Missing |
|---|---|---|
| Start in order, with dependencies | In the order of `start` commands; `exec` runs a command and waits for it | Starting a service once another is ready |
| Restart, with a limit and a growing delay | After a fixed period, forever; or a reboot on failure | A limit, a growing delay |
| Readiness | — | All of it: its properties can only be set from `init.rc` itself |
| Stop | From `init.rc` actions | A way for other programs to ask |
| Which task is which program | Known inside `nxinit`, which started them | A way for others to ask |

Nor can a service be added while the system runs, which an installed
native app ([RFC 0002](0002-build-mode.md)) needs: `import` reads extra
files only at start-up.

The previous system ran sixteen services under `nxinit` with no trouble,
but built readiness itself: a registry service and a state topic that
every service published.

## Proposal

### `nxinit` is the manager

`nxinit` is the first task. The system's programs are declared in
`/etc/init.d/init.rc`, which is part of the firmware. What `nxinit`
already does stays as it is: starting services and classes of services,
collecting programs that exit, restarting them, stopping them (with
`SIGTERM` first where asked), running commands at start-up, and reacting
to properties.

### What we add

Each addition is a general feature, kept as its own commit in the
nuttx-apps fork ([RFC 0003](0003-repositories.md)).

1. **Readiness.** A service declared with a new option, `notify`, counts
   as starting until it reports that it is ready; one without it counts as
   ready once started. `nxinit` keeps each service's state in a property,
   `svc.<name>.state`: `starting`, `ready`, `restarting`, `stopped` or
   `failed`. Dependencies are then written as actions:

   ```
   on property:svc.cfgd.state=ready
      start modemd
   ```

   Property triggers fire each time their condition becomes true, so this
   action runs again after `cfgd` restarts; starting a service that is
   already running does nothing.

2. **Restart limits and a growing delay.** New service options set the
   first delay and the longest one. The delay counts from the moment the
   service exited (today's `restart_period` counts from when it started),
   doubles with each failure in a row, and goes back to the first once the
   service has stayed up long enough. After a set number of failures within
   a set time, the service is `failed` and is not restarted again;
   `init.rc` can react to that like any other state:

   ```
   on property:svc.modemd.state=failed
      start modem-recovery
   ```

   `reboot_on_failure` keeps its present meaning: the device restarts at
   the service's first failure.

3. **A control socket.** A local stream socket at a fixed path, speaking a
   small text protocol: one command per line, one answer per command, and
   a version line when a connection opens.

   | Command | Does |
   |---|---|
   | `start <name>`, `stop <name>` | starts or stops a service |
   | `state [<name>]` | one service's state, or all of them, with their tasks |
   | `watch` | keeps the connection open and sends a line whenever a state changes |
   | `who <task>` | which service a task is |
   | `ready` | reports that the caller is ready; accepted only from that service's own task, checked with `SO_PEERCRED` |
   | `add`, `remove <name>` | adds a service, described as an `init.rc` fragment, or removes one, while the system runs (for native apps) |

   A text protocol keeps `nxinit` general, with no dependency on pnut-os,
   and lets a developer drive it by hand: a small `svc` command in NSH
   (`svc state`, `svc stop lorad`) speaks it.

### pnut-os's side: one module speaks to `nxinit`

Apart from `ready`, no pnut-os program speaks the text protocol. One
pnut-os module does, and offers the rest of the system a normal interface
([RFC 0005](0005-communication.md)):

- it keeps a `watch` connection open and publishes every state change on a
  uORB topic, so services wait for each other in their event loops without
  each asking the manager;
- it keeps the table of which task is which service, from those same
  changes;
- its interface (a `.proto` file, like any other) offers the states,
  starting and stopping services, and which program a task is.

A service reports `ready` to `nxinit` directly, because the check that the
report comes from the service's own task works only on a direct
connection; the service library does it in one call.

### Caller identity

A service learns the caller's task from the socket (`SO_PEERCRED`) and asks
the module which program it is, once per connection: a connection's caller
does not change while it is open. For WebAssembly apps, the runtime marks
each call with the app's identity (RFC 0004).

### What the manager does not do

- **WebAssembly apps** run inside the runtime, which looks after them
  itself; the manager sees only the runtime program.
- **Nothing outside the list above:** no logging, timers, mounts or
  networking (RFC 0004).

### If this outgrows `nxinit`

If the additions grow beyond this list, or upstream takes `nxinit` somewhere
they cannot follow, the fallback is a manager of our own in pnut-os,
started as the first task, decided in a new RFC.

## Alternatives

- **`nxinit` as it is.** Services publish their own readiness and identity
  comes from task names. The least work, but the manager's duties end up
  spread over every service, and any task can change its own name.
- **A manager of our own in pnut-os, as the first task.** Fits RFC 0005
  exactly and needs no patches, but rewrites what `nxinit` does well, makes
  us sole owner of the most critical program, and needs a configuration
  format of its own.
- **nanopb on the control socket.** One protocol for the whole system, but
  `nxinit` would depend on nanopb and on a `.proto` file of ours, so the
  additions would no longer be general, and it could not be driven by hand.
- **Android's property names** (`init.svc.<name>`). Familiar, but the
  states here differ from Android's, so the names would be only half
  compatible.
- **`nxinit` starting a manager of ours, which supervises the rest.** Two
  supervisors; and on NuttX a restarted manager cannot collect the exit of
  programs its previous instance started.

## Costs and risks

- **Patches on a busy component:** `nxinit` changes often upstream, so
  every release move may mean reworking our additions.
- **`nxinit` is still marked experimental** in NuttX.
- **The manager is the most critical program:** a fault in it stops the
  system, so its additions stay small.
- **A second, small protocol** beside RFC 0005's, spoken by `nxinit`, the
  module, the service library (for `ready`) and the `svc` command.
- **Dependencies are limited by `nxinit`'s action events:** an action
  waits on at most `SYSTEM_NXINIT_ACTION_EVENTS_MAX` conditions (2 by
  default; up to 64).
