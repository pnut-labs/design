# RFC 0020: Notifications

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-06
- **Last changed:** 2026-10-06
- **Supersedes / superseded by:** —

## Summary

How notifications work: what one carries (little, since screens are
small), channels with an importance the owner can change, asking an app
whether it may post, focus modes such as do not disturb, alerts, and a
history. The Notifications service keeps them; the system UI shows them.

## Problem

- [RFC 0007](0007-services.md) gives the notifications currently posted
  to the Notifications service: services and apps post them, the system
  UI shows them ([RFC 0019](0019-system-ui.md)).
- [RFC 0008](0008-runtime.md) lets manifests declare notification
  channels, and a notification's buttons call the app's actions.
- [RFC 0009](0009-packages.md) asks for permissions when they are first
  needed.
- Haptics owns vibration, Audio sound, and Board the notification lights
  (RFC 0007).

## Proposal

### What a notification carries

Little, since screens are small:

- **who posted it,** an app or a service, whose icon comes with it, and
  **its channel;**
- **a title and a short text,** and the time;
- **a group,** such as one conversation, so several become one entry;
- **a main action,** called when the notification is opened, and **one
  more action,** shown as a button; each calls one of the app's actions
  (RFC 0008);
- **whether it is ongoing,** such as a call in progress, and cannot be
  dismissed.

What of it is shown, and where, is the visual design's choice.

### Channels and importance

Each channel is declared in the manifest with a default importance, which
the owner can change for every channel:

| Importance | Does |
|---|---|
| **Urgent** | sound or vibration, and wakes the screen |
| **Normal** | sound or vibration |
| **Quiet** | in the list and the status bar, without sound |
| **Silent** | in the list only |

### Asking whether an app may post

Posting is a permission, asked for the first time an app posts
(RFC 0009). The owner answers:

| Answer | Means |
|---|---|
| **Allow** | as each channel says |
| **Allow quietly** | posted, but every channel quiet: no sound, no vibration, no waking |
| **Don't allow** | not posted |

The answer can be changed in Settings, for the app or each channel.

### Focus modes

A focus mode lets through only what is on its **list of people and apps**;
everything else goes at once, quietly, to the list of notifications.

- **Do not disturb** is the first mode: its list starts with alarms, and
  the owner adds people and apps.
- **The owner can make other modes,** such as work or sleep, each with a
  list of its own.
- **A mode can switch on by itself,** on a schedule (from 23:00 to 7:00)
  or at a place, through Location (RFC 0007) and with its permission:
  from satellites, or more cheaply from cell towers. Later, perhaps, by
  touching an NFC tag. The system offers these; the owner chooses whether
  to use them.

### Alerts

Notifications decides how to alert, as the channel and the focus mode
say, and asks the services that own the hardware: Audio for a sound,
Haptics for a vibration pattern, Board for a notification light.

- **On e-paper,** new notifications are gathered into one redraw of the
  status bar, at most once every few seconds; only urgent ones wake the
  screen at once.
- **The lock screen,** part of the system UI, reads the notifications
  like the rest of it, and takes from them only what the visual design
  shows there (RFC 0019).

### History

Dismissed notifications are kept for a few days, in a history the owner
can open from the list.

### Later

**Push from the internet,** where one connection carries messages for
every app instead of each app keeping its own, comes in an RFC of its own;
UnifiedPush, an open standard for it, is the starting point. Until then,
apps allowed to run in the background keep their own connections
([RFC 0015](0015-network.md)).

### Starting values

Set for each device, in Board's description; to be measured:

| What | Value |
|---|---|
| A notification's title | 100 characters |
| A notification's text | 1,000 characters; the system UI shortens it to what the screen fits |
| The shortest time between redraws for new notifications, on e-paper | 5 s |
| How long the history is kept | 7 days |

## Alternatives

- **Rich notifications,** with pictures, progress, several buttons and
  replies typed in place. More in each one, but no room for it on a small
  screen.
- **Allowing every app to post by default.** One question fewer, but the
  owner would first learn of a noisy app from its noise.
- **Do not disturb as a single switch** with fixed exceptions. Simpler,
  but one list cannot suit both a night and a meeting.
- **Showing notifications' content on the lock screen,** as a choice. More
  at a glance, but anyone holding the device could read it.

## Costs and risks

- **Place-based focus modes** cost power while Location watches where the
  device is; less from cell towers than from satellites. It is the owner's
  choice.
- **Every app keeping its own connection** for its notifications costs
  power until the push RFC.
- **The history** is private data about the owner, kept on the device.
