# RFC 0015: Network access for apps

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

A Network service owns connectivity and carries apps' traffic: HTTP(S)
requests, TCP and UDP connections with TLS done for them, and downloads
written straight into a file. Apps reach only the hosts their manifest
names, each connection is logged, and the owner chooses for each app
which links it may use.

## Problem

- [RFC 0008](0008-runtime.md) gives apps no network until it has an
  owner: WASI's sockets answer "not supported".
- [RFC 0007](0007-services.md) gives each radio an owner (Wi-Fi the Wi-Fi
  radio, Telephony the modem and its mobile data, carried by NuttX's
  `pppd`), but no service owns connectivity itself: which link is in use,
  name lookups, what a link costs.
- Native code uses the kernel's sockets directly
  ([RFC 0002](0002-build-mode.md)).

nuttx-apps has mbedtls (3.6.2) for TLS, an HTTP/1.1 client that can use
TLS, and MQTT and WebSocket clients.

## Proposal

### The Network service

**Network** is a service in the radios program, beside Wi-Fi and
Bluetooth (RFC 0007). It owns:

- **connectivity:** which links are up (Wi-Fi, mobile data, others later)
  and which one carries traffic, and whether a link costs money;
- **name lookups** (DNS);
- **apps' connections,** with TLS and the trusted certificates;
- **what each app uses,** for Settings.

The runtime checks an app's permissions and passes its calls on, marked
with the app ([RFC 0005](0005-communication.md)). Every call is
asynchronous; data arriving on a connection comes to the app as events,
with credit ([RFC 0004](0004-layers.md)).

RFC 0007's list of services gains Network, in the radios program.

### What apps get

| What | For |
|---|---|
| **HTTP(S) requests** | most apps: a method, a URL, headers and a body; the answer comes back as events, the body in pieces |
| **TCP and UDP connections,** with TLS done by Network when asked | other protocols: mail, chat, MQTT |
| **Downloads into a file** | large files: Network writes the body straight into a file in the app's places, handed over by Storage ([RFC 0011](0011-storage.md)), so a podcast never passes through the app |
| **Connections from the local network** | an app serving devices on the same network, with the local-network permission (below); nothing from the internet |

Names are looked up through Network, and apps are told when the link in
use changes, or costs money, so they can wait for Wi-Fi.

### Permissions and hosts

- **Internet** is a permission, asked for when first needed, like every
  other ([RFC 0009](0009-packages.md)).
- **The hosts an app talks to** are named in its manifest, as domains
  (a domain covers the names under it), or "any". The prompt shows them,
  and Network refuses a connection anywhere else.
- **Every connection is logged:** which app, which host, when. The owner
  sees it in Settings.
- **The local network** (other devices on the same Wi-Fi) is a separate
  permission, so an app cannot look around the owner's network unasked.

### Which links an app may use

The internet permission covers every link. In Settings, the owner can
choose for each app:

- **which links:** Wi-Fi, mobile data, both, or any other link added
  later;
- **whether it may use the network in the background,** beside the
  background permissions of RFC 0008, so an app may run in the
  background yet stay offline there.

Network keeps each app's connections to the links it may use; NuttX can
tie a socket to one network interface (`SO_BINDTODEVICE`), which Network
relies on. Native apps use sockets directly, and are not limited by
this.

### Certificates

- **The trusted root certificates** ship in the firmware, in `/etc/ssl`,
  and are updated with it ([RFC 0013](0013-firmware-updates.md)).
- **An app can pin** its own server's certificate, for its connections.
- **The owner can add certificates** in Developer options, for a home
  server for example.

### Limits

Set for each device, in Board's description; starting values, to be
measured:

| Limit | Value |
|---|---|
| Connections open per app | 8 |
| Requests waiting per app | 8; another fails at once with "busy" |
| TLS handshakes at once | 2 |
| Connection log kept | 7 days |

## Alternatives

- **The runtime carrying apps' traffic** in its own event loop. One step
  fewer, but TLS handshakes are heavy, and the runtime has one worker,
  shared with every app.
- **Sockets and TLS inside each app.** Nothing to design, but TLS
  compiled into every app is large, and slow when interpreted; and
  nothing could check where apps connect.
- **HTTP(S) only.** The simplest, but no mail, chat or MQTT.
- **Sockets only.** General, but every app would carry its own HTTP.
- **No list of hosts.** One permission fewer, but the owner could not see
  or limit where an app connects.

## Costs and risks

- **Every byte crosses a program** on its way to an app: from the kernel
  to Network, then to the runtime.
- **A list of domains is coarse:** "any" gives an app the whole
  internet, and a domain covers all the names under it.
- **The connection log** is private data about the owner, kept on the
  device.
- **Tying sockets to one link** relies on NuttX's `SO_BINDTODEVICE`,
  which NuttX describes mostly for UDP; that it holds for every protocol
  is to be checked.
- **TLS takes memory** for every open connection, and a handshake takes
  time; to be measured.
