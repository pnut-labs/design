# RFC 0019: The system UI

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-09
- **Supersedes / superseded by:** —

## Summary

How the system UI works: one program, built on LVGL, owns the screen and
every input. Apps describe their screens from building blocks, with a
canvas for what must be drawn, and the system UI draws them, adapting to
the kind of screen. Every screen works with keys alone and with touch
alone. How it looks is designed apart from the RFCs.

## Problem

- [RFC 0004](0004-layers.md) and [RFC 0007](0007-services.md) make the
  system UI one program that owns the screen and all input: keys,
  keyboards, touch and the rest.
- [RFC 0008](0008-runtime.md) gives apps screens only through the system
  UI's public interface, and lets manifests declare widgets, notification
  channels and a settings schema that Settings draws.
- Later RFCs give the system UI its prompts and confirmations: permissions
  and installs ([RFC 0009](0009-packages.md)), the USB port
  ([RFC 0012](0012-developer-access.md)), the lock screen
  ([RFC 0014](0014-security.md)), each use of the security key
  ([RFC 0018](0018-authenticator.md)), and the scripts, fonts and
  keyboard layouts of [RFC 0017](0017-i18n.md).
- Screens differ widely between devices: an e-paper screen is 1-bit and
  takes most of a second to refresh; an LCD has colour and redraws at
  once.

nuttx-apps has LVGL 9, a widely used toolkit that handles 1-bit screens
and input devices, and NuttX's own NX windowing, which is older and
little used.

## Proposal

### How it works, not how it looks

This RFC defines how the system UI behaves. How it looks (layouts,
spacing, icons, the arrangement of each screen) is designed apart from
the RFCs, in the project's visual design, which is still in progress.
Themes (below) carry that design to the device.

### One program, on LVGL

The system UI is the `ui` program (RFC 0007), built on **LVGL**:

- **LVGL runs on the program's event loop** (RFC 0004): its timers are
  the loop's timers, and drawing happens there.
- **It is a client of services, and nothing below calls it** (RFC 0004).
  When a service needs the owner (a permission, an install, a security
  key's use, preparing a memory card), it announces it; the system UI asks
  the owner and sends the answer back.
- **It owns every input device** and every screen, and lends neither.

### The kind of screen

The system UI adapts to the screen that Board's description names
(RFC 0007):

| | E-paper | LCD |
|---|---|---|
| Colour | 1-bit, or a few greys | colour |
| Redrawing | slow: each action costs one redraw, changes are gathered into one, nothing moves | fast: transitions may move |
| Long lists | page | scroll |
| Ghosting | a full refresh after every few partial ones; the owner sets how many | — |

The same building blocks work on both; apps do not know which screen they
are on.

### Input

**Every screen works with keys alone, and with touch alone,** since a
device may lack either.

- **Building blocks carry meaning,** not keys: the primary action, the
  options, going back, moving between items. The system UI maps them to
  what the device has: softkeys, navigation keys, a keyboard, touch keys,
  a touch screen.
- **Options can suggest a letter,** so an important action is one key
  away on a keyboard; the system UI gives the final letters, so they never
  clash.
- **Text is entered through the system UI,** which owns keyboard layouts,
  letters with diacritics, and an on-screen keyboard on devices without
  keys; apps receive the text.

### Keys and chords

A keyboard driver delivers characters as printed on the keys, and tells
which modifiers are in effect ([RFC 0024](0024-board.md)); what a key or a
chord means on screen is the system UI's, the same in every app. Input
([RFC 0027](0027-input.md)) brings the keys to it, repeated when held.

| Keys | Meaning |
|---|---|
| **Function keys,** three | the options, the primary action, back, left to right |
| **Function keys,** two | the options, back; the primary action is the steering's (below) |
| **Alt and Enter** | the options, in a text field too |
| **Space; Shift and Space** | outside a text field: the next page; the previous one |

- **A chord counts the modifier in effect,** however it came to be: held,
  tapped before the key, or locked.
- **Find is a role,** given to a key by the keyboard's description: on a
  keyboard with a Sym key, Sym.

### Steering with a keyboard

A keyboard is often a device's only way to move between items. The system
UI suggests this map; a device's description may name its own instead
(RFC 0024), and the system UI follows it.

| Keys | Outside a text field | While typing in one |
|---|---|---|
| **W, S** | up, down | type |
| **A, D** | left, right, where the screen has a row (tabs, choices, a slider); the previous and the next page where it has not | type |
| **Q, Backspace** | back | Q types; Backspace deletes |
| **E, Enter** | the primary action | E types; Enter confirms a one-line field and moves to the next item, or starts a new line |
| **Find held, with W, A, S, D** | as W, A, S, D | arrows: the cursor, and past the text's edge the neighbouring item |
| **Find held, with Q; with E** | back; the primary action | leaving the field, its text kept; the screen's primary action |
| **Find tapped** | find, in the screen's list | — |

- **Arrow keys and a navigation's centre,** where the device has them, do
  what W, A, S, D and E do, while typing too.
- **The letters a map steers with** (W, A, S, D, Q and E in this one) are
  never given to commands, so an option's letter never moves the screen.
- **Backspace only deletes** while typing, even in an empty field: held,
  it would otherwise delete the text and then leave the screen.
- **How the hints look,** a letter beside an option for one, is the visual
  design's.

### Apps' screens

Apps never draw pixels on their own. They describe screens from
**building blocks**, through the system UI's public interface, and the
system UI draws them:

| Building blocks | For |
|---|---|
| **Lists** | rows with a title, details, an icon, a value, a mark, a switch; long lists are fetched a page at a time ([RFC 0005](0005-communication.md)) |
| **Forms** | text, numbers, choices, dates and times, switches |
| **Text and images** | text with simple styling, pictures, progress |
| **Actions** | the primary action, options, confirmations |
| **A canvas** | what must be drawn: maps, charts, simple games |

- **A screen is a tree of blocks,** each with an id. An app changes a
  screen by sending only the blocks that changed.
- **Events come back as events** (RFC 0005): a row opened, a value
  changed, text entered, an option chosen, back.
- **A canvas** takes drawing commands or a picture from the app, and is
  redrawn only when the app asks; on e-paper that matters most. Keys and
  touches on it go to the app.
- **The system UI keeps the stack of screens,** the apps' and its own,
  and decides when an app's screen is shown: only while the app is open
  (RFC 0008).
- **The exact catalogue** of building blocks, and how screens are
  encoded, are set in the system UI's reference documentation, written
  later.

### Surfaces apps contribute to

Apps can contribute to the system's own surfaces, declared in their
manifest (RFC 0008) and built from the same blocks:

- **widgets** on the home screen;
- **glances** on the lock screen, such as a count;
- **icons** in the status bar.

The app updates their content through the public interface, even while it
runs in the background; the system UI decides where they go and when they
are redrawn, so an e-paper screen is not refreshed for every change. The
owner chooses which of them are shown.

### The system's own screens

The system UI draws, with the same building blocks:

- the **lock screen** and unlocking (RFC 0014);
- the **home screen** and the **list of apps**, which opens an app through
  its `open` action (RFC 0008);
- the **status bar** and the **notifications** on screen;
- **Settings,** including every app's settings, drawn from their schemas
  (RFC 0008);
- the **store** (RFC 0009);
- **prompts and confirmations:** permissions, installs, the USB port,
  intents' choice of app (RFC 0008), the security key, preparing a
  memory card ([RFC 0010](0010-file-layout.md)).

### Themes and fonts

- **A theme** carries the visual design to the device: colours or greys,
  fonts, sizes, spacing, icons. It is made for a kind of screen, and
  changes how things look, never how they work.
- **Themes are packages,** installed in the package store like language
  packs (RFC 0017); a default theme comes with the firmware.
- **Fonts:** one default family, covering Latin, Cyrillic and Greek
  (RFC 0017), chosen with the visual design.
- **Changing the theme, the language or the region** takes effect at once:
  the system UI and open apps' screens are drawn again.

## Alternatives

- **NuttX's NX widgets.** Already in NuttX, but older and little used.
- **A small renderer of the project's own.** Less code, but every widget,
  font and input to write.
- **Only building blocks, no canvas.** Every app consistent, but maps,
  charts and games impossible.
- **Apps drawing everything themselves.** Freedom, but every app would
  have to handle e-paper, keys and touch on its own, and look different.
- **Designing the look in the RFCs.** One place for everything, but the
  look changes far more often than how the system works.

## Costs and risks

- **Every app screen crosses programs** as a description, and is drawn by
  the system UI; to be measured on e-paper and LCD.
- **The building blocks limit apps** to what they can express; the canvas
  is the way out, and costs a redraw each time.
- **Fonts take flash** for every size and script; whether to render them
  from outlines at run time instead is to be measured.
- **A fault in the system UI** takes every screen with it, until the
  manager restarts it ([RFC 0006](0006-manager.md)).
