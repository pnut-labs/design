# RFC 0027: Input

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-09
- **Last changed:** 2026-10-09
- **Supersedes / superseded by:** —

## Summary

How input reaches the screen. A part of the system UI, Input, opens every
input device the device's description names, and turns keyboards,
function keys, navigation, buttons and touch into one stream for the
screen in front: it repeats held keys, adds the letters of the owner's
languages, and leaves what keys mean to the system UI. The kinds of input
are generic, so the same code serves every device.

## Problem

The RFCs so far each hold a piece of input:

- the system UI **owns every input device** ([RFC 0004](0004-layers.md),
  [RFC 0007](0007-services.md));
- it **maps meanings to keys:** every screen works with keys alone and
  with touch alone, and chords and function keys have their meanings
  ([RFC 0019](0019-system-ui.md));
- **keyboard layouts and letters with diacritics** are left to be designed
  with the system UI ([RFC 0017](0017-i18n.md));
- the device's description **names its inputs,** and every keyboard driver
  delivers characters as printed on the keys, with the modifiers in effect
  ([RFC 0024](0024-board.md)).

What no RFC says yet: how input moves from the drivers to a screen, what
happens when a key is held, how a letter with a diacritic is typed, what
function keys are on a device that has them and on one that does not,
what buttons do, what reaches apps, what works while the device is
locked, and how input is tested without hands.

NuttX offers a keyboard upper half (`/dev/kbdN`: characters, special keys,
modifiers), a touchscreen upper half, buttons, and the ability to inject
events into a keyboard. LVGL takes keypads, pointers, encoders and
buttons as input devices.

## Proposal

### One part of the system UI

**Input** is a module of the system UI's program (RFC 0007), on its event
loop ([RFC 0023](0023-service-library.md)):

- it opens **every input device Board's description names** (RFC 0024),
  and nothing else opens them;
- it feeds them to **LVGL's input devices,** so the building blocks of
  RFC 0019 receive keys and touches as LVGL does;
- it adds **what depends on the owner,** never on the hardware: repeat,
  languages, long presses. What a key or a chord means stays the system
  UI's (RFC 0019); what is printed on the keys stays the driver's
  (RFC 0024).

### The kinds of input

| Kind | The driver gives | Input makes of it |
|---|---|---|
| **Keyboard** | characters as printed, special keys, the modifiers in effect | text and keys; repeat; the letters of the owner's languages; chords, for the system UI |
| **Function keys** | presses and releases, each key numbered from the left | the screen's soft keys (below) |
| **Navigation** | arrows, the centre, a wheel's steps | moving between items; the primary action |
| **Buttons** | presses and releases, each with its role | by role (below) |
| **Touch** | points and their movement | taps, long presses, swipes, through LVGL |

A device may have any of them; a touch-only device and a keys-only device
are both whole (RFC 0019).

### Function keys

Function keys are **soft keys,** as on a classic phone: the screen shows,
above or beside each key, what it does now, and RFC 0019 gives them their
meanings (the options, the primary action, back).

- **Physical keys or touch keys on the glass** are function keys alike,
  numbered from the left.
- **A device without them** shows the same actions as touchable buttons,
  or reaches them with the keyboard's chords (RFC 0019).
- **A long press** of a function key may be given a place to open by the
  owner, in Settings; none by default.

### Holding a key

- **Backspace, Space, the arrows, the navigation and the keys steering**
  (W, A, S and D outside a text field, or with Find held, in the system
  UI's suggested map, RFC 0019) repeat while held: after half a second,
  then 20 times a second, to start with. The owner sets the delay and the
  rate, or turns repeating off.
- **A letter being typed does not repeat:** held, it offers its variants
  (below).
- **Modifiers, Enter and the function keys** never repeat.
- **Input repeats, not the drivers,** which report a press and a release
  only. On an e-paper screen, repeated keys are gathered into one redraw
  (RFC 0019).

### Letters of the owner's languages

The keys give what is printed on them; letters with diacritics come from
the **owner's languages** (RFC 0017), not from the keyboard:

- **Holding a letter** offers its variants in the owner's languages: `ą`
  for `a` in Polish, `ä` and `à` in German and French; the key again, or
  the navigation, moves between them, and letting go of the letter types
  the one chosen.
- **What variants a letter has** comes with each language: English's in
  the firmware, the others' in their language packs (RFC 0017).
- **Text reaches apps as text,** whatever keys made it (RFC 0019).

### Buttons

| Role | Does |
|---|---|
| **Power** | a press locks the screen, or wakes it; a long press offers to switch off or restart |
| **Volume** | steps the volume of what plays, through Audio |
| **Others** | what the owner gives them in Settings, from the system UI's actions |

### What reaches apps

- **Apps never see input devices.** Their screens' building blocks give
  them events with meanings: a row opened, an option chosen, text entered
  (RFC 0019).
- **A canvas** receives keys and touches in the system UI's terms:
  characters, special keys, the modifiers in effect, touch points.

### While the device is locked

- **The power button** wakes the screen to unlock it; the owner may let
  any key do so instead.
- **Touch and function keys** do nothing until it is unlocked, so a
  device carried in a pocket stays still.
- **Unlocking itself,** a PIN or a password, is typed as everywhere else
  ([RFC 0014](0014-security.md)).

### Testing

Every kind of input can be **injected** in place of a device: NuttX's
keyboard upper half takes events written to it, and the simulator has a
keyboard and a pointer. Input reads injected events as it reads a
driver's, so the system tests of [RFC 0022](0022-build.md) drive the
system UI without hands.

## Rules it follows

- **One owner per device** (RFC 0004): Input, in the system UI.
- **Facts, choices and meanings apart:** what is printed on the keys is
  the driver's and the description's (RFC 0024); repeat, languages and
  long presses are the owner's, in Settings; what keys mean is the system
  UI's (RFC 0019).
- **Calls down, events up** (RFC 0004): Input asks Audio for the volume;
  it does not reach into it.

## Alternatives

- **Repeat in the drivers.** Each driver would do it again, and the owner
  could not set it.
- **Letters with diacritics as keyboard layers,** in the driver's keymap.
  But the owner's languages are not a fact of the hardware, and a keymap
  per language per device would follow.
- **Input as a service of its own,** in another program. Every keystroke
  would cross programs, and the system UI owns input anyway.
- **Function keys as plain keys** (F1 to F3) with meanings fixed per
  device. Simpler, but it ties the meanings to one device.
- **Letters that repeat when held,** as on a computer. But holding a
  letter is the quickest way to its variants, and a repeated letter is
  rarely wanted on a phone.

## Costs and risks

- **Holding a letter** no longer repeats it; someone used to that loses
  it.
- **Variants per language** have to be made and kept with the language
  packs.
- **On e-paper,** every key is a redraw; repeat and fast typing depend on
  gathering them.

## Open questions

- **Which variants each language offers,** and in what order: from
  CLDR's tables, or chosen by hand.
- **Alt with a letter** for a language's commonest variant, on keyboards
  whose Alt layer leaves letters free.
