# RFC 0024: Board

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-08
- **Last changed:** 2026-10-08
- **Supersedes / superseded by:** —

## Summary

Board is the service that knows the device. It serves the device
description: what the firmware was built for, from a file per target, and
what differs per unit, from the device data partition, which Board alone
reads and serves to the other services. It also owns the parts no generic
service owns: backlights, lights, privacy switches, and the USB port's
mode.

## Problem

[RFC 0007](0007-services.md) makes Board the service where a device's
specifics live, so that every other service stays the same on every
device, and gives it two things: the device description, and the parts
no generic service owns. The RFCs since lean on it:

- apps' manifests name the hardware they **need** or **can use**, checked
  against the description when they are installed and when they run
  ([RFC 0008](0008-runtime.md), [RFC 0009](0009-packages.md));
- **limits set for each device** live in the description: the apps'
  memory budget (RFC 0008), Storage's and Network's limits
  ([RFC 0011](0011-storage.md), [RFC 0015](0015-network.md)), the
  notifications' ([RFC 0020](0020-notifications.md));
- the system UI adapts to **the kind of screen** the description names
  ([RFC 0019](0019-system-ui.md));
- Board **switches the USB port** between its modes
  ([RFC 0012](0012-developer-access.md));
- the **device data partition** (serial number, calibration, device keys)
  is read by Board and Security ([RFC 0010](0010-file-layout.md));
- Notifications asks Board for a **notification light** (RFC 0020).

NuttX can report a board's unique id (`BOARDIOC_UNIQUEID`); the rest is
the project's.

## Proposal

### The device description

The description comes from two sources, which Board merges and serves:

| Source | Holds | Comes from |
|---|---|---|
| **The target's file** | what the firmware was built for: the model, the hardware and its parameters, the limits | `configs/<target>/board.toml` ([RFC 0022](0022-build.md)), compiled to Protocol Buffers by the build, as a manifest is (RFC 0008), and placed in `/etc` with the firmware's other configuration (RFC 0010) |
| **Device data** | what differs per unit: the serial number, the hardware revision, calibration | the device data partition, read at start-up (below) |

- **The schema** is a `.proto` file in `proto/`
  ([RFC 0005](0005-communication.md)), and it **enumerates the hardware
  kinds** (`lora`, `gnss`, `modem`, `nfc`, `camera`, …), so apps'
  manifests and devices' descriptions use the same names. A kind, once
  numbered, is never renumbered.
- **Several of a kind** are allowed: two SIM slots, two screens.
- **The description does not change while the device runs;** readers ask
  once and keep it.
- **The simulator and QEMU targets** have descriptions of their own
  (RFC 0022): a window for a screen, no radios, and so on.

### What it holds

| Part | For example |
|---|---|
| **Identity** | the model, the hardware revision, the serial number |
| **Hardware present,** each with its parameters | radios and their bands; the screen's kind, size and depth; inputs (keyboard, touch, keys); sensors; audio parts; cameras; the modem's bands and SIM slots; internal flash and its partitions (RFC 0010); external storage slots; the USB port's possible modes (RFC 0012) |
| **Limits for this device** | the apps' memory budget (RFC 0008); Storage's limits (RFC 0011); Network's (RFC 0015); Notifications' (RFC 0020); and others as RFCs add them |

### Who reads it

| Reader | For |
|---|---|
| the runtime and Packages | an app's "needs" and "can use", the memory budget |
| the system UI | the kind of screen, the inputs; the About screen |
| Storage, Network, Notifications | their limits |
| the services | their parts' parameters: Radio the bands, Telephony the modem's bands, Power the battery's |
| diagnostics and developer tools | all of it |

A service asks Board rather than assuming a device, so the same service
runs on every device.

### Device data

- **Board alone opens the partition.** It reads it at start-up and serves
  its records to any service that asks, Security included (on a chip
  without an HMAC unit, Security's device key is one of them,
  [RFC 0014](0014-security.md)). One reader knows the partition's format.
- **Records are keyed by name,** each a Protocol Buffers message like
  everything else; a service owns the records it understands
  (calibration belongs to the service that uses it).
- **Writing happens when a device is provisioned,** over USB with
  developer access (RFC 0012), by a provisioning command; and once, at
  first set-up, for a record a service must make on the device itself,
  such as Security's device key on a chip without an HMAC unit
  (RFC 0014). Otherwise the partition is read-only to everyone, and the
  owner never writes it. A factory reset keeps it (RFC 0010).
- **A device that was never provisioned runs,** fully, on the target's
  file alone: it must, since provisioning itself needs the running system
  (developer access over USB, RFC 0012), and so does recovering a device
  whose device data was damaged. Board marks the description as
  unprovisioned, which the About screen and developer tools show; what
  needs per-unit data falls back to the target's defaults.

### The parts Board owns

| Part | Board does | Others |
|---|---|---|
| **Backlights** (screen, keyboard) | on, off, brightness | the system UI keeps the dimming policy; services may ask for a moment's light, such as the keyboard for a notification |
| **Lights** (status, notification) | drives them: a colour and on-and-off timings | Notifications asks for a pattern (RFC 0020); Power for charging |
| **Privacy switches** (microphone, camera) | publishes their state on a topic | the services concerned obey: Audio mutes, Camera stops |
| **The USB port** | switches it between its modes, and publishes whether it is connected and in which mode | the system UI asks the owner which mode (RFC 0012) |

**The rule** stays RFC 0007's: a part goes to Board only when no generic
service owns parts of its kind. An antenna switch is Radio's, a speaker's
source Audio's, a charger's settings Power's, each power rail the service
whose part it feeds, keys and touch the system UI's.

### Board itself

Board runs in the system program and is never optional (RFC 0007). Its
interface offers the description and device data's records as requests,
and the backlights, lights and the USB port's mode as requests and
events; the privacy switches and the port's state are topics.

## Alternatives

- **The whole description in device data.** One source, but every unit
  would be provisioned with facts the firmware already knows.
- **The whole description compiled in.** No partition to read, but one
  firmware per hardware revision, and no serial number.
- **Kconfig options as the description.** Already there at build time,
  but not readable by apps and tools while the device runs, and not a
  structure.
- **Every service reading device data itself.** No Board in the way, but
  every service would learn the partition's format.
- **A Lights service** instead of Board. Rejected in RFC 0007.
- **Writing device data from Settings.** Convenient for calibration, but
  the owner could damage what the device needs to work.

## Costs and risks

- **Board can become a junk drawer** (RFC 0007); the rule is what keeps
  parts out of it.
- **The schema grows** with every new hardware kind and every new
  per-device limit, and its numbers can never be reused.
- **Two sources to merge:** a mismatch between the target's file and the
  unit's revision is caught only at start-up, and logged.
- **A device needs provisioning** to have a serial number and
  calibration; until then it runs on the target's defaults, marked
  unprovisioned.
