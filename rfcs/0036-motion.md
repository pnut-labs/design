# RFC 0036: Motion

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-11-13
- **Last changed:** 2026-11-13
- **Supersedes / superseded by:** —

## Summary

Motion owns the motion sensors and tells what they mean: whether the
device is still or moving, which way it faces, the steps taken today,
what its owner is doing (walking, running, in a vehicle), and the
gestures that wake it. Where a device has a sensor hub, the hub works it
all out while the processor sleeps. It runs in the **sensors** program
([RFC 0007](0007-services.md)).

## Problem

RFC 0007 lists Motion among the services. The system UI wants to wake
when the device is picked up, and to stop a ring when the device is
turned face down; Location could rest its receiver while the device
lies still, and apps count steps. No RFC
says what Motion offers or who may know it.

The first device has a Bosch BHI260AP: an accelerometer, a gyroscope and
a small processor of its own, which runs Bosch's firmware image
(BSD-3-Clause), uploaded each time the hub is powered. Besides the raw
readings, the image works out "virtual sensors": steps, significant
motion, still or moving, orientation, gestures, depending on the image.
Its NuttX driver offers only the accelerometer and the gyroscope today.

## Proposal

### What others see

| Call | Does |
|---|---|
| **Steps** | today's count, since local midnight, and when it last changed |
| **Follow steps** | a stream of the count as it changes, until the caller ends it |
| **Activity** | what the owner is doing: still, walking, running, on a bicycle, in a vehicle, or unknown; and a stream of its changes |
| **Abilities** | what Motion can tell on this device: which `motion` fields it fills, and whether it counts steps, tells the activity and sees gestures |

| Topic | Says | For |
|---|---|---|
| **`motion`** | still or moving; which way the device faces (up, down, upright, on its side) | anyone allowed (below) |
| **`gestures`** | picked up; lifted to look at | the system UI alone |

What a device's hub cannot work out, Motion says it lacks, rather than
guessing: a caller asks Abilities, and a topic field it cannot fill says
"unknown".

### In the sensor hub

Motion sets the hub up and turns its virtual sensors on and off as they
are asked for, at the slowest rate that serves every caller:

- **The hub does the work:** steps, stillness, orientation, gestures and
  activity are computed on the hub, which wakes the processor only when
  something changes.
- **Which virtual sensors a hub has** depends on its firmware image;
  Motion reads the list from the hub at start and offers what it finds.
- **A device without a hub** has less: Motion offers what the raw
  accelerometer can tell cheaply (still or moving, facing), and nothing
  else; working out steps on the processor would keep it awake.
- **The raw readings** (accelerometer, gyroscope) are the Sensors
  service's, for games and tools. This changes RFC 0007, which gave
  Sensors the other sensors only.

### Steps

- **Today's count** is Motion's: the hub counts, and Motion keeps the
  count across restarts of the hub or the device, so a restart at noon
  does not lose the morning.
- **The history,** day by day, is Health's, once Health exists
  (RFC 0007); until then there is none. This changes RFC 0007, which
  gave the history to Motion.

### Settings

([RFC 0025](0025-settings.md))

| Setting | Starting value |
|---|---|
| `motion.steps`: count steps | on |
| `motion.wake`: wake the screen when the device is picked up | on |
| `motion.face_down`: turning the device face down silences a ring | on |

The actions themselves are the system UI's, which rings for a call
([RFC 0030](0030-telephony.md)); the settings say whether Motion's
gestures drive them.

### Who may know what

| What | Needs |
|---|---|
| Still or moving, facing (`motion`) | nothing, for the app on screen; the permission to stay in the background, for one that is not |
| Steps, activity | the **physical activity** permission ([RFC 0008](0008-runtime.md)), asked when first needed |
| Gestures (`gestures`) | the system UI alone; apps never see them |
| Abilities | nothing: it describes the device, not its owner |

### Where it runs

The sensors program (RFC 0007), beside Location and Sensors, on its
event loop: the hub sends only changes.

## Rules it follows

- **One owner per device** ([RFC 0004](0004-layers.md)): Motion sets the
  hub up and turns its virtual sensors on; the raw readings go to Sensors
  through the kernel's sensor framework, not past Motion's setup.
- **Power by default:** the processor sleeps while the hub works; what
  nobody asks for is off.
- **Permissions where the data is personal:** steps and activity tell
  how someone lives, and need permission.

## Alternatives

- **On the processor,** from the raw readings: the same on every device,
  but the processor wakes at the sensor's rate, many times a second, for
  as long as steps are counted.
- **The hub first, the processor for what it lacks:** every device would
  offer everything, at the cost of keeping two ways of working each thing
  out; worth it only once a device without a hub comes.
- **Step history in Motion:** one service fewer to ask, but health data
  would sit in two places with two rules.
- **Everything behind one permission:** simpler to ask, but a level in a
  game would need the same grant as a record of someone's walks.

## Costs and risks

- **Bosch's image decides** what the first device can do: its list of
  virtual sensors is to be read from the hub before the first version is
  promised. A different image, built with Bosch's tools, could add what
  is missing.
- **Gestures misfire:** a device picked up in a pocket wakes its screen;
  the setting turns it off, and the system UI may ignore wakes with the
  proximity sensor where there is one.
- **The upload** takes time and current each time the hub is powered:
  once per start of the device while its rail stays on, before the first
  sensor works.
