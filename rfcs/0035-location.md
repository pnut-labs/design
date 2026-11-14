# RFC 0035: Location

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-11-13
- **Last changed:** 2026-11-13
- **Supersedes / superseded by:** —

## Summary

Location owns the satellite positioning (GNSS) receiver. It answers
where the device is, once or as it moves, to callers allowed to know,
approximately or precisely; it keeps the receiver off while nobody asks,
and gets a fix fast without any network by keeping the receiver's orbit
data across switch-offs. It runs in the **sensors** program
([RFC 0007](0007-services.md)).

## Problem

RFC 0007 lists Location among the services. Apps want positions (maps, a
tracker, a mesh app sharing its place), Time takes the satellites' time
([RFC 0028](0028-time.md)), and focus modes may follow places later
([RFC 0020](0020-notifications.md)). No RFC says what Location offers,
who may ask, or how it keeps the receiver's cost down.

The first device has a u-blox MIA-M10Q (GPS, GLONASS, Galileo, BeiDou)
on a UART, on a rail that is switched off when unused. Its NuttX driver
parses the receiver's NMEA sentences into topics and passes the raw data
through a device file. Switched off, the receiver forgets the satellites'
orbits, so each start is a cold start, the slowest kind.

## Proposal

### What others see

| Call | Does |
|---|---|
| **Current** | where the device is: a position no older than the caller allows and at least as accurate as it asks, within a time limit; answered with the position, or "not yet" with the best known |
| **Follow** | a stream of positions at an interval the caller asks, each as accurate as it asks, until it ends the stream |

A position has latitude, longitude, altitude, their accuracy (metres, as
the receiver estimates it), speed, heading, the time it was taken, and
the satellites used.

A topic, **`location`**, says whether the receiver is on, whether it has
a fix and with how many satellites, for the status bar. It never carries
the position: topics are public, positions need permission.

A topic, **`gnss_time`**, carries the satellites' time, its uncertainty
and when it was taken, while the receiver has a fix and something else
uses it, for Time ([RFC 0028](0028-time.md)); no position.

### Satellites only, for now

Positions come from the GNSS receiver alone. Positioning from Wi-Fi
networks and cell towers needs a database of where they are: an online
lookup, which tells a server where the device is, or a large regional
download. Either can come later, through Wi-Fi's and Telephony's
interfaces, as its own change.

### The receiver

Location speaks each receiver's own protocol, through the raw device:

- **A module per chip family,** chosen by the chip Board's description
  names ([RFC 0024](0024-board.md)): u-blox's UBX first. Through it,
  Location sets the constellations, the power mode and the update rate,
  reads the receiver's own accuracy estimate, and saves and restores its
  orbit data.
- **NMEA for any other receiver:** positions and satellites, nothing
  more.

The NuttX driver's NMEA topics stay for tools and tests; Location does
not read them.

### Fast fixes, without a network

- **The receiver's own prediction** of the satellites' orbits
  (AssistNow Autonomous on u-blox), worked out from what it has heard,
  good for a few days.
- **Its orbit data saved** to `/var/lib/location` before the receiver is
  switched off, and given back when it is switched on, with the time
  (from Time) and the last position: a start within a few hours of the
  last becomes a hot start, a few seconds.
- **Nothing is fetched or sent:** no assistance server, no account.

### Power

- **The receiver is on only while someone asks.** A few seconds after the
  last caller, Location saves the orbit data and lets the driver switch
  the rail off.
- **A long interval** (a tracker asking every few minutes) switches the
  receiver off between fixes, each then a hot start; a short one keeps it
  on in its power-saving mode where the chip has one.
- **A Current with a recent enough position** is answered from it,
  without switching the receiver on.

### Who may know where the device is

Three permissions ([RFC 0008](0008-runtime.md)), each asked when first
needed:

| Permission | Gives |
|---|---|
| **Approximate location** | positions rounded to a grid of a few kilometres, their accuracy given as the grid's size, without altitude, speed or heading, in whatever answer they come; while the app is on screen |
| **Precise location** | positions as the receiver gives them, while the app is on screen |
| **Location in the background** | either of the above while the app is not on screen; asked only after one of them |

Services (the system UI) are callers like any other, without the
rounding.

### Where it runs

The sensors program (RFC 0007), beside Motion and Sensors: its event loop
reads the receiver's few hundred bytes a second.

## Rules it follows

- **One owner per device** ([RFC 0004](0004-layers.md)): Location alone
  opens the receiver's raw device.
- **Permissions where the data is personal:** positions only to callers
  allowed, rounded unless precise is granted; the public topics say only
  that the receiver is on, and the satellites' time; never a position.
- **Power by default:** the receiver is off unless someone asks.

## Alternatives

- **NuttX's NMEA topics only:** nothing to write per chip, but no
  accuracy estimate, no power modes and no saved orbits: every start a
  cold one.
- **Online assistance** (u-blox's AssistNow service): the fastest first
  fix, but an account token, the network, and a server that learns when
  the device asks.
- **A coarse position from the cell:** quick and cheap in power, but a
  database lookup that tells a server where the device is, or a large
  download; later, if wanted.
- **One location permission:** simpler to ask, but every app that wants
  a city gets a street.

## Costs and risks

- **No fix measured yet on the first device:** only tried indoors during
  bring-up. The first outdoor cold start, and a hot start from saved
  data, are to be measured before the hot start's few seconds are trusted.
- **Saved orbit data ages:** after a few days it no longer helps, and the
  start is cold again.
- **A module per chip family** is code to write and test for each; NMEA
  covers the rest, poorly.
- **Writes to `/var/lib/location`** at each switch-off: a few kilobytes,
  once per use.
