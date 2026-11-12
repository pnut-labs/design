# RFC 0033: Haptics

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-11-12
- **Last changed:** 2026-11-12
- **Supersedes / superseded by:** —

## Summary

Haptics owns the vibration motors. It plays named effects, which the
system maps to each device's motor so they feel the same everywhere, and
short patterns of its callers' own, capped in length. It runs in the
**media** program, beside Audio ([RFC 0007](0007-services.md)).

## Problem

[RFC 0007](0007-services.md) lists Haptics among the services, and
Notifications asks it for an alert's vibration
([RFC 0020](0020-notifications.md)); an incoming call vibrates while it
rings ([RFC 0030](0030-telephony.md)); apps vibrate too
([RFC 0008](0008-runtime.md)). No RFC says what Haptics offers.

The first device has one motor, a rotating one (ERM), driven by a
DRV2605L over I2C. NuttX's force-feedback driver for it, in the fork,
plays two kinds of effect: a level held for a time (constant or rumble),
and short shaped effects from the chip's own library (clicks, bumps,
buzzes).

## Proposal

### What others see

| Call | Does |
|---|---|
| **Effect** | a named effect (below), with its use |
| **Pattern** | the caller's own: steps of a duration and a strength (0 to 100), up to 5 s in all, played once |
| **Stop** | what the caller started |

A topic, **`haptics`**, says what plays, for the system UI and tests.

### Named effects

| Effect | For |
|---|---|
| **tick** | a key or a step in a list |
| **click** | a press that did something |
| **double click** | a choice confirmed |
| **notification** | an alert ([RFC 0020](0020-notifications.md)) |
| **ring** | an incoming call: repeats until stopped |
| **alarm** | an alarm: repeats until stopped |
| **success**, **error** | an action's outcome |

- **Each device maps them** to its motor: on the first device, the short
  ones to effects of the chip's library, the longer ones to held levels
  with pauses. The map is Haptics' table for the motor's kind, which
  Board's description names ([RFC 0024](0024-board.md)).
- **Only ring and alarm repeat.** A caller's pattern plays once; a
  caller wanting it again asks again.

### What plays over what

One vibration at a time, by use, as Audio's uses
([RFC 0032](0032-audio.md)): **ring** and **alarm** stop anything lower
and are not stopped by it; **notification** stops the interface's and
apps' effects; a lower one asked while a higher plays is dropped, not
queued, since a late vibration means nothing.

### Settings

([RFC 0025](0025-settings.md))

| Setting | Starting value |
|---|---|
| `haptics.strength`: weak, medium or strong, scaling every effect | medium |
| `haptics.interface`: ticks and clicks of the system UI | off on a device with a keyboard, on otherwise |

Whether an alert vibrates is Notifications' decision, with the focus
modes (RFC 0020), as for sound.

### Who may vibrate

- **Apps** need no permission while on screen; in the background, the
  permission to stay running ([RFC 0008](0008-runtime.md)). A pattern is
  capped at 5 s, and an app may start at most one a second.
- **Services** (Notifications, Telephony's ring through the system UI,
  Time's alarms) call it like any caller, with their uses.

### Where it runs

The media program (RFC 0007), on its event loop: the driver's timing runs
in the kernel, and the motor's I2C writes are short. The motor's rail is
Board's, kept on so that the I2C bus it shares answers.

## Rules it follows

- **One owner per device** ([RFC 0004](0004-layers.md)): Haptics alone
  opens the motor's driver.
- **Policy where it belongs:** whether to vibrate for an alert is
  Notifications'; the effects' feel is the device's map; how strong is
  the owner's.

## Alternatives

- **Named effects only:** every vibration feels the same, and no app can
  make an annoying one, but no app can make its own (a game's rumble, a
  signal in Morse).
- **Any pattern, uncapped:** an app could keep the motor running, with
  the battery and the owner's patience.
- **Haptics inside Audio:** one service fewer, but one would own two
  unrelated devices, against RFC 0004's rule.

## Costs and risks

- **An ERM is slow:** it takes tens of milliseconds to spin up and down,
  so ticks feel soft; the chip's library effects overdrive and brake it,
  which is why the short effects use them.
- **The I2C bus is shared** with the codec's control, the IMU and others:
  a long pattern's steps are a few writes each, not a stream.
- **The map's feel** is a matter of taste, tuned on the device.
