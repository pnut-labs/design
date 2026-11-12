# RFC 0032: Audio

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-11-12
- **Last changed:** 2026-11-12
- **Supersedes / superseded by:** —

## Summary

Audio owns the codec, the amplifier and the switch in front of it, the
speaker and the microphone. It plays sounds and streams mixed together,
the lower ones quieter under the higher, decodes WAV, Opus and MP3,
records from the microphone, gives the speaker to the modem for a call,
and keeps three volumes. It runs in the **media** program
([RFC 0007](0007-services.md)).

## Problem

RFC 0007 lists Audio among the services, but no RFC
says what it does, and others wait for it:

- Notifications asks Audio for an alert's sound
  ([RFC 0020](0020-notifications.md)).
- A call needs the speaker switched to the modem, and an incoming call
  rings ([RFC 0030](0030-telephony.md)): calls wait for Audio.
- Apps play sounds and record ([RFC 0008](0008-runtime.md)), and a music
  or podcast app plays files from apps' storage
  ([RFC 0011](0011-storage.md)).

What the first device has, and what the port to it has shown:

- **One codec, mono:** its digital-to-analog side feeds the amplifier and
  the speaker, its analog-to-digital side the microphone. Both share one
  I2S bus and one clock, so playing and recording run at one rate.
- **The modem has its own microphone,** and its speaker output reaches
  the amplifier through a switch, the codec's or the modem's: the two
  cannot be heard together.
- **The amplifier has a power rail** of its own, on the I/O expander.
- **NuttX's audio driver** offers the codec as an output and an input
  device. A player killed while playing leaves the codec reserved until a
  restart, and the I2S driver has open follow-ups from its review; the
  speaker has not been heard on this port yet.

## Proposal

### What others see

| Call | Does |
|---|---|
| **Play** | a sound: a built-in one by name, or a file handed over as an open file (RFC 0011), with its use (below), once or looping; returns a handle |
| **Stop** | a sound, by its handle |
| **Output** | a stream of samples the caller writes: its rate, channels and use; returns a ring in shared memory ([RFC 0005](0005-communication.md)) |
| **Input** | a stream from the microphone, at a rate and channels the caller asks; returns a ring |
| **Route** | the speaker to the modem for a call, and back (Telephony only) |

A topic, **`audio`**, says what plays (each use, and whether the call
route is on) and whether the microphone is recording, for the status bar.
The volumes are settings (below).

### Uses, and what plays over what

Every sound and stream has a **use**, which sets its volume and how it
meets the others:

| Use | Volume | Over a lower use |
|---|---|---|
| **call** | call | the speaker goes to the modem: nothing else is heard |
| **alarm** | ring, never silent | lower uses are muted while it plays |
| **ring** | ring | lower uses are muted while it plays |
| **notification** | ring | lower uses are ducked: quieter, by a set amount |
| **interface** (key clicks, sounds of the system UI) | ring | mixed in as they are |
| **media** (music, podcasts, games, an app's sounds) | media | — |

- **A software mixer** adds every sound and stream playing into one
  output, at one rate: each is decoded, resampled to that rate, scaled by
  its volume and by any ducking or muting, and summed with clipping.
- **A stream that is muted** goes on being read, so its owner keeps its
  pace; the `audio` topic says it is muted, so a player can pause
  instead.
- **During a call** the switch gives the speaker to the modem, and the
  codec's output is not heard: alerts that come meanwhile are left to
  Haptics and the screen, as Notifications decides (RFC 0020). The
  microphone stays free, since the modem has its own.

### Volumes

Three settings ([RFC 0025](0025-settings.md)), 0 to 10:
`audio.volume.ring` (ring, alarms, notifications, the interface),
`audio.volume.media` and `audio.volume.call`. An alarm plays at the ring
volume but at least at 1. Whether an alert makes a sound at all is
Notifications' decision, with the focus modes (RFC 0020); Audio plays
what it is asked to.

### Formats

| Format | Decoder | License | Why |
|---|---|---|---|
| WAV (PCM, 8, 16 or 24 bits) | Audio's own | — | built-in sounds, recordings |
| Opus, in Ogg | libopus, opusfile, libogg | BSD | small files of good quality: ringtones, voice |
| MP3 | minimp3 | public domain (CC0) | the format most owners' own sounds and music come in |

- **Built-in sounds** are short files in the firmware
  (`/usr/share/sounds`, [RFC 0010](0010-file-layout.md)): a ringtone, an
  alarm, a notification, a key click, an error.
- **libmad,** the MP3 decoder nuttx-apps carries, is GPL, and is not
  used: the firmware is Apache-2.0.
- Decoding runs on the media program's worker, ahead of the mixer, so a
  slow decode never stops the output.

### The output

- **One rate for the device,** a starting value: 48 kHz, 16 bits, mono
  on the first device, as Opus decodes at 48 kHz and the codec is mono.
  MP3 at 44.1 kHz and WAV at any rate are resampled.
- **The mixer runs on its own thread,** at a priority above the event
  loop's, and fills the codec's buffers a period at a time (20 ms to
  start with): the loop takes requests, the thread only mixes.
- **The amplifier is powered only while something plays,** and for a
  moment after, so that a run of key clicks does not switch it on and off
  each time; it is switched with the output silent, against clicks in
  the speaker.
- **Recording** shares the clock: the microphone is read at the output's
  rate and resampled to what each caller asked.
- **The codec is opened once** and kept by Audio, so no player of its own
  can leave it reserved; NuttX's driver fix for a killed user stays
  wanted, for the developer's shell.

### Who may play and record

- **Playing** needs no permission while the app is on screen; in the
  background, the app needs the permission to stay running
  (RFC 0008).
- **Recording** needs the microphone permission, asked when first needed
  (RFC 0008). While anything records, the status bar shows it, from the
  `audio` topic.
- **Route** is Telephony's alone (RFC 0005's caller identity).
- Services (Notifications, Time's alarms, the system UI) play sounds as
  callers like any other, with their uses.

### Where it runs

The media program (RFC 0007): its event loop, a worker for decoding, and
the mixer's thread. Board's description ([RFC 0024](0024-board.md)) lists
the codec, the amplifier's rail and the switch; a device without them
has no Audio.

## Rules it follows

- **One owner per device** ([RFC 0004](0004-layers.md)): Audio alone
  opens the codec and switches the amplifier and the speaker's source;
  Telephony asks for the speaker rather than switching it.
- **Streams through shared memory** (RFC 0005): an output stream drops
  its oldest block when its ring is full (live audio), a recording makes
  the producer wait.
- **Policy where it belongs:** whether to sound is Notifications'; how
  loud is the owner's, in the volumes; what plays over what is Audio's.

## Alternatives

- **One sound at a time, by priority, without mixing.** Simpler, no
  resampling, little processor time; a notification would pause music
  instead of playing over it.
- **WAV only to start,** with a compressed format added when an app
  needs one: less code now, but the owner's own ringtones and music
  could not be played.
- **Two volumes** (everything but calls, and calls), or **four**, like
  Android (ring, notifications, media and alarm apart).
- **NuttX's player** (`nxplayer`) as the service: it plays one file at a
  time, with no mixing, no streams and no uses.

## Costs and risks

- **Flash:** libopus with opusfile and libogg about 150-200 KB, minimp3
  about 20 KB; to be measured.
- **Memory and processor:** each decoder's state (Opus about 20-30 KB,
  MP3 about 10 KB), the rings and the mixer's buffers, in external RAM
  where they can be; resampling and mixing at 48 kHz keep the processor
  busier while something plays, and the power with it.
- **The I2S driver's open follow-ups** (stops that leak buffers in
  flight, playback resetting a recording's transmitter on the shared
  clock) are fixed in the NuttX fork before Audio relies on both
  directions at once.
- **The speaker has not been heard on this port:** the first slice
  starts by playing a built-in sound and listening.
- **A switch, not a mixer, in front of the amplifier:** nothing but the
  call is heard during one, on the first device.
