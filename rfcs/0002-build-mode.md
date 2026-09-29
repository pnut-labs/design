# RFC 0002: Build mode and trust

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-09-28
- **Last changed:** 2026-09-29
- **Supersedes / superseded by:** —

## Summary

pnut-os uses NuttX's flat build until it runs on hardware that can do
better. Protection goes where the risk is: apps distributed by the
pnut-os project are WebAssembly, confined by their runtime; native code is
trusted. The device's owner can still install native apps, deliberately,
over USB.

## Problem

NuttX ([RFC 0001](0001-nuttx.md)) has three builds, and they separate
different things:

| Build | Separates | On the ESP32-S3 |
|---|---|---|
| Flat | nothing: kernel, drivers and all programs share one address space | works |
| Protected | the kernel from the programs; the programs still share one user space, so they are **not** protected from each other | little used: two sample configurations upstream, both for Espressif's devkit |
| Kernel | every program in its own address space | not possible: needs an MMU with address translation, which the chip lacks |

So no build on this chip separates programs from each other. The system
has to decide where its protection comes from.

A first attempt at the protected build on this device (2026-09-25) found:

- Espressif's QEMU cannot run it (no World Controller), so every test is
  on the device.
- It needs Espressif's second-stage bootloader and partition table: a
  different boot chain and flash layout.
- The link failed: the protected linker scripts have no PSRAM data
  sections, no memory kept across resets and no RTC sections; the
  kernel's instruction RAM overflowed by 4.7 KB; a C library function was
  missing on the kernel side.
- The RAM split is fixed at build time: the kernel keeps a fixed part of
  internal RAM (192 KB by default) for Wi-Fi, Bluetooth, the drivers and
  the screen buffer, and PSRAM becomes the programs' heap only. Internal
  RAM is already the tightest limit
  ([T-Deck Max](../hardware/t-deck-max.md#memory-and-storage)).

## Proposal

### The flat build

pnut-os uses the **flat build** on the T-Deck Max, and on any device
without better memory protection. It is revisited when the system moves
to hardware that has it.

### Who is trusted

- **Trusted, native:** the kernel, the drivers, the system services and
  the shell: the firmware.
- **Not trusted, WebAssembly:** every app distributed by the pnut-os
  project. A WebAssembly module reaches only its own memory and the
  functions its host gives it, so the runtime is the boundary. Which
  runtime is decided in its own RFC.
- **Native apps installed by the device's owner** are trusted too
  (below).

### The rule that keeps the protected build possible

Programs reach the kernel and the hardware **only through system calls and
device files**: no kernel-internal functions, and no hardware registers
outside drivers. Code written this way runs unchanged in the protected
build, so moving to it later is board and linker work, not a redesign.

### Native apps

The device's owner can install native apps. A native app runs with the
system's full rights: it can read and change everything, permissions do
not restrict it, and a bug in it can bring the device down. Installing one
is meant to be a conscious step, not a hard one:

- **Off by default.** A switch in Developer options turns native apps on.
- **Over USB only**, never from the store or the network. Each install is
  confirmed on the phone's screen.
- **Tied to the firmware version.** Native apps call the system's internal
  interfaces, which may change with any update. The loader refuses an app
  built for another version rather than letting it crash; native apps are
  rebuilt after an update. (WebAssembly apps are not affected.)

Self-built firmware can include native code, and needs none of this.

## Alternatives

- **Flat, everything native.** Simplest and fastest, but no protection
  where the risk is: code from others.
- **The protected build.** Shields the kernel from a crashing program, but
  not programs from each other; costs the port work above, a fixed RAM
  split, system-call overhead, and testing only on the device. Revisited
  with better hardware.
- **Native apps only in self-built firmware** (no loader in the image).
  Hardest to misuse, but too closed for an enthusiast device.
- **Guarded native installs:** the owner's PIN for each install, a
  signing key enrolled on the device, a wipe when native apps are first
  turned on, markings and a safe mode. Not chosen: each adds a step for
  everyone who builds on the device, while turning native apps on and
  confirming each install on the screen already make installing one a
  conscious act.

## Costs and risks

- **WebAssembly's cost:** apps run slower and use more memory than native
  code.
- **No protection among native code:** a bug in a service, the shell or a
  native app can corrupt the others or the kernel.
- **A native app has full access** to everything on the device, including
  messages and keys.
- **Native apps break on updates** until they are rebuilt.

## Open questions

- **When to revisit:** which hardware, and what the protected build would
  cost there (RAM, system-call overhead).
- **What native apps may call:** which of the system's interfaces they
  build against.
