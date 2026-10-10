# RFC 0007: Services and programs

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-03
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

The services of pnut-os, what each owns, and the nine programs they are
grouped into. The list covers the parts a device pnut-os runs on may
have, not only those of a device that exists today; a service whose
hardware a device lacks is simply not started. What is specific to one
device is kept in one service, Board.

## Problem

[RFC 0004](0004-layers.md) says when something may be a service (it owns a
device, or shared state that several clients need or that must outlive a
client), that every device has one owner, that services are optional where
their hardware is, and that services are modules grouped into a few
programs by domain. This RFC applies those rules, so that the same list of
services works on every device pnut-os runs on, and a new part finds its
place without reopening the list. Each device's hardware is described
under [`hardware/`](../hardware/).

## Proposal

### Services that own a device

| Service | Owns | Optional |
|---|---|---|
| **Telephony** | the cellular modem and its control lines: calls, SMS, network state, mobile data, one or more SIMs or eSIMs, and satellite messaging where the modem offers it. Data runs through NuttX's `pppd` on a channel of the modem's link, which Telephony lends it | yes |
| **Radio** | low-power packet radios: LoRa and other sub-GHz radios, 802.15.4 radios (Thread, Zigbee), ultra-wideband (UWB). It sets a radio up and lends it to whatever runs the protocol, such as a mesh app (RFC 0004) | yes |
| **Wi-Fi** | the Wi-Fi radio: scanning, known networks, connecting | yes |
| **Bluetooth** | the Bluetooth radio, including links to audio devices such as headphones; their sound goes through Audio | yes |
| **NFC** | the NFC radio | yes |
| **Location** | the satellite positioning (GNSS) receiver; it may also position from Wi-Fi networks and cell towers, through those services' interfaces | yes |
| **Motion** | the motion sensors: orientation, movement and stillness, steps, wake gestures. It sets up a sensor hub where a device has one, and keeps the step history across restarts | yes |
| **Sensors** | the other sensors: ambient light, proximity, compass, barometer, temperature, humidity, air quality, UV, hinge or slide position (below) | yes |
| **Health** | heart rate and other health sensors, and the history they build up; their data needs permission everywhere | yes |
| **Audio** | the audio codec and amplifier, speakers and earpiece, microphones, a headphone socket, and which source drives the output, including an FM tuner where a device has one. Its streams use shared memory ([RFC 0005](0005-communication.md)) | yes |
| **Camera** | cameras; frames go through shared memory (RFC 0005) | yes |
| **Haptics** | vibration motors, rotating or linear: vibration patterns | yes |
| **Power** | chargers, wired and wireless, USB power delivery, batteries and their gauges, and the decision to sleep: battery state, requests to stay awake (RFC 0004), external power, restart and power-off | no |
| **Security** | a secure element and a fingerprint reader where a device has them, and the secrets and checks they serve (below) | no |
| **Board** | what is specific to the device (below) | no |

### Services that own shared state

| Service | Owns |
|---|---|
| **Service states** | which services are running and ready, and which task is which program: the module of [RFC 0006](0006-manager.md) |
| **Settings** | every setting, its schema, and change events |
| **Time** | the wall clock, the time zone, network time, and the timers that wake the device; RFC 0004 asks for exactly one keeper of them, so alarms live here |
| **Notifications** | the notifications currently posted: services post them, the system UI shows them |
| **Log** | logs kept across restarts; NuttX's own log is a ring in RAM |
| **Messages** | one store for SMS and mesh messages; Telephony and mesh apps deliver into it |
| **Contacts** | the address book and the call history |
| **Packages** | installed apps, native and WebAssembly, and the permissions each has been granted |
| **Storage** | apps' files and databases, and external storage: mounting it and preparing the apps' storage ([RFC 0011](0011-storage.md)) |
| **Network** | connectivity (which link carries traffic, what it costs), name lookups, TLS and the trusted certificates, and apps' connections ([RFC 0015](0015-network.md)) |

### What is not a service

| What | Why not | Lives in |
|---|---|---|
| Screens, touch, keys, keyboards, and other input: trackpads, trackballs, knobs, navigation keys, styluses | the system UI owns the screen and input (RFC 0004) | the system UI |
| I/O expanders and shared buses | the kernel splits them into per-function devices (one per power rail, for example), each with its owner | the kernel |
| Mesh protocols and other protocols over Radio | protocols, not devices | apps, or stacks decided in their own RFCs |
| Calendars, notes and the like | app data | apps, with Time for alarms |
| `pppd`, `adbd`, NSH | NuttX programs | started by the manager |
| The WebAssembly runtime | a program that runs apps | its own program |

### Sensors

NuttX's sensor framework publishes each sensor as a topic, and the kernel
shares it between readers: when several ask for different rates, the
sensor runs at the fastest and every reader gets the data. The Sensors
service builds on that rather than repeating it:

- **the data flows on the topics,** straight from the kernel to readers,
  with no copying through the service; the runtime decides which apps may
  see which topic (RFC 0005);
- **the service owns what the framework does not:** calibration and
  reference values (a compass's calibration, a barometer's reference
  pressure), the catalogue of sensors present with their units and ranges,
  switching a sensor off when nobody reads it, and publishing sensors whose
  drivers are not on the framework as topics like the rest.

Location, Motion and Health stay services of their own: each has logic of
its own, and the data of Location and Health needs permission, so it does
not go on open topics (RFC 0005).

### Security

Security proves who is using the device and keeps secrets:

- **keys and passkeys,** in a secure element where the device has one, so
  they never leave the chip; otherwise encrypted in flash, with the
  processor's key hardware where it has some;
- **stored passwords,** the vault behind a password manager; the manager
  itself is an app;
- **authentication:** the screen lock, the fingerprint reader, and the
  unlock state other services ask about;
- later, perhaps, **acting as a passkey authenticator** for other devices,
  over Bluetooth, USB or NFC.

It is designed in its own RFC.

### Board

Board is where a device's specifics live, so that every other service
stays the same on every device:

- **the device description:** the model and hardware revision, which
  hardware is present, and its parameters (the LoRa band, the screen's
  size and depth, the modem's bands). The system UI, the runtime ("this
  app needs LoRa") and diagnostics read it there, and services ask Board
  instead of assuming a device;
- **parts no generic service owns,** such as the screen and keyboard
  backlights (brightness, on or off, and requests from others, such as
  lighting the keyboard for a notification; when to dim stays the system
  UI's policy), status and notification lights, and hardware privacy
  switches, whose state Board publishes and the services concerned obey.

A part that belongs to a generic service stays with it on every device: an
antenna switch with Radio, the speaker's source with Audio, the charger's
settings with Power, each power rail with the service whose part it feeds,
keys and touch with the system UI.

### Packages

Packages is shared by the runtime (loading apps, checking their
permissions), the system UI (the app list, permission prompts, Settings),
the installer and, for native apps, the manager. It lives in the data
program, apart from the runtime: native apps are packages too, apps stay
manageable when the runtime fails, and the runtime stays focused on running
apps. The runtime keeps each running app's grants in memory and updates
them when Packages announces a change, so crossing programs costs only when
an app starts, not on every call.

### Programs

| Program | Services |
|---|---|
| **system** | Service states, Settings, Power, Time, Notifications, Log, Board |
| **telephony** | Telephony |
| **radios** | Radio, Wi-Fi, Bluetooth, NFC, Network |
| **sensors** | Location, Motion, Sensors, Health |
| **data** | Messages, Contacts, Packages, Storage |
| **media** | Audio, Camera, Haptics |
| **security** | Security |
| **ui** | the system UI |
| **runtime** | WebAssembly apps |

Besides these, the manager starts NuttX's own programs: `pppd`, `adbd` and
NSH. A service whose hardware a device lacks is not started, and its
program runs without it.

The table is in the firmware, as `/etc/pnut/programs`: one line per
program, its name as the manager knows it, then the services it runs,
named as their settings' owner ([RFC 0025](0025-settings.md)). A service
is one program's.
Settings lets a program reach only its services' settings, so moving a
service to another program is a change of this table alone.

```
# program     services
system        states settings power time notifications log board
telephony     telephony
```

- **Telephony runs alone.** It is the most complex service and the most
  exposed to the outside world (network events, the contents of SMS), and
  calls must keep working when something else fails.
- **The system program stays small and stable:** every other program waits
  on its settings and service states.
- **Sensing is apart from the radios and from audio,** so a fault in a
  sensor's setup stops neither communication nor a call's sound.
- **The data program writes to flash,** which is slow, so its worker pool
  (RFC 0004) is where that waiting happens.
- **Security runs alone,** so its event loop is never held up by others
  and its code stays small enough to review closely.
- **The grouping can change** without touching callers (RFC 0004), once the
  programs' costs are measured.

Each program runs one event loop and a small pool of workers (RFC 0004).
Starting values, to be confirmed by measurement on each device:

- a handler on the loop returns within **10 ms**;
- **one worker** per program, **two** in the data program;
- every task and worker has a stack of **at least 4 KB**: the previous
  system found that 2 KB stacks overflowed silently once a task logged or
  called a service.

## Alternatives

- **One program per service.** About twenty programs: clear, but each
  costs a task, stacks and buffers, and the flat build does not isolate a
  crash anyway ([RFC 0002](0002-build-mode.md)).
- **Three programs:** the system UI, the runtime, and one for every
  service. The cheapest, but one fault stops every service.
- **Grouping by risk:** every service that parses outside input alone.
  More programs at the edges for little gain beyond Telephony.
- **Location and Motion in the radios or media program.** One program
  fewer, but a fault in a sensor's setup would stop the radios or a call's
  sound.
- **A service for each kind of radio** (LoRa, Thread, UWB). Each new radio
  would need a new service, though all of them are set up and lent the
  same way.
- **No service for simple sensors,** only the kernel's topics. Nothing
  extra to build, but calibration, the catalogue and sensors outside the
  framework would have no home.
- **A service for every sensor.** Uniform, but each would repeat what the
  kernel already shares.
- **A key store and a fingerprint service apart.** Two services guarding
  the same secrets, and the unlock state split between them.
- **Security in the system program.** One program fewer, but its event loop
  would be shared with everything the system program does.
- **The backlights in the system UI or in Power.** The UI would make other
  services go through it, against RFC 0004; Power would mix lights into
  battery and sleep.
- **A Lights service** instead of Board. Simpler, but the device
  description and other parts specific to a device would still need a
  home.
- **Packages inside the runtime.** Free permission checks, but native apps'
  packages would live in the WebAssembly runtime, and a runtime failure
  would take package management with it.

## Costs and risks

- **Board could become a junk drawer:** only its rule keeps parts that
  belong to a generic service out of it.
- **Nine programs** cost nine tasks, their stacks and their workers;
  where those stacks can live (internal RAM or PSRAM) is still to be
  measured.
- **Modules in one program share its fate:** a fault in one stops the
  others in the same program, so each program groups services that can
  sensibly fail together.
- **A program of its own does not protect Security's memory** in the flat
  build: any native code can read it (RFC 0002). Keys are truly protected
  only where a secure element holds them.
