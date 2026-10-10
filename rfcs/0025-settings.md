# RFC 0025: Settings

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-08
- **Last changed:** 2026-10-09
- **Supersedes / superseded by:** —

## Summary

Settings holds every setting. A setting is named by its owner and its
key; services register their schemas at start-up and apps at install;
the values live in one file per owner, written whole; a change is
announced on one topic by name, and readers fetch the value. Secrets are
encrypted at rest. An owner reaches only its own settings, the system UI
all of them, and anyone the public ones.

## Problem

- [RFC 0007](0007-services.md) makes Settings the owner of every
  setting, its schema and its change events.
- [RFC 0008](0008-runtime.md) puts an app's schema in its manifest (keys,
  types, defaults, ranges or choices, labels, groups); Settings stores the
  values, and the system UI draws the screens from the schemas
  ([RFC 0019](0019-system-ui.md)).
- [RFC 0009](0009-packages.md) has Settings take an app's schema at
  install and drop its settings at removal.
- [RFC 0010](0010-file-layout.md) puts default settings in `/etc` and
  services' state in `/var/lib`.

The previous system kept one file per key and hit littlefs's 32-character
limit on file names, so it hashed them.

## Proposal

### Naming

A setting is named by **two fields: its owner and its key.**

- **The owner** is a service's name or an app's id. An app never names
  it: the runtime supplies the app's identity (RFC 0008), so an app
  reaches only its own settings. Services and the system UI name the
  owner.
- **The key** is a short name within the owner, lowercase, with dots for
  grouping (`display.timeout`). Nothing is parsed out of it: grouping for
  the screens comes from the schema.

### The schema

Every setting is declared before it is used; Settings refuses a key it
does not know.

| Type | With |
|---|---|
| **boolean** | — |
| **integer** | a range, and a unit for the screen |
| **string** | a maximum length |
| **choice** | the list of values |
| **secret** | a maximum length; stored encrypted (below) |

Each setting carries its default, a label and a group, translated
([RFC 0017](0017-i18n.md)), and may be marked **public** (below).

| Owner | Registers | When |
|---|---|---|
| a service | a schema generated with its interface from its `.proto` file ([RFC 0023](0023-service-library.md)) | at start-up |
| an app | the schema in its manifest | at install (RFC 0009) |

When a new version of a service or app changes its schema, added
settings take their defaults, removed ones lose their values, and a
setting whose type changed is treated as removed and added.

### Defaults

A setting's default comes from its schema, **overridden per target** by
the firmware's defaults file: written as `configs/<target>/defaults.toml`
([RFC 0022](0022-build.md)) and compiled into `/etc` with the rest of the
firmware's configuration (RFC 0010). So one service or app can start
with different values on different devices.

### Storage

- **One file per owner,** a Protocol Buffers message holding that owner's
  values, in `/var/lib/settings/`. A change writes the file beside the
  old one and renames it over, so a crash never leaves half a file.
- **Values stay in memory** while the device runs, in fixed pools
  (RFC 0023); a read is answered from memory.
- **Writes are gathered:** a burst of changes makes one write, a moment
  later, on the program's worker, since a littlefs commit is slow.

### Changes

One uORB topic, **`settings`**, carries each change as the owner, the
key and a version number, never the value: topics are open to all native
code ([RFC 0005](0005-communication.md)), and values may be secrets.
Readers fetch the value. An empty key means that several of the owner's
settings may have changed: its schema was registered, or all were reset.
The version counts Settings' changes, all owners' together, one up for
each message, so a reader checks every message's version: a reader that
sees a gap, or a smaller one (Settings restarted), missed changes and
reads its settings again. The runtime passes an app only the
changes to its own settings (RFC 0008).

### Secrets

A secret (a Wi-Fi passphrase, a token) is **encrypted at rest** under a
key Security derives from the device key alone
([RFC 0014](0014-security.md)), not from the unlock key: a passphrase is
needed before the first unlock, so that Wi-Fi connects after a restart. A
copy of the flash away from the device cannot read it; native code on the
device can, as RFC 0014 accepts for the flat build. A secret is never
logged, the system UI shows it as dots, and only its owner reads it.

### Access

| Who | May |
|---|---|
| a service | read and write its own settings, and register its schema |
| an app | read and write its own, through the runtime |
| the system UI | read and write every setting, through the schema |
| anyone | read a setting marked **public**: the language and the region, 12 or 24 hours, and the like |

A service is known by the program that calls: Service states says which
program a task is, and `/etc/pnut/programs` which services it runs
([RFC 0006](0006-manager.md), RFC 0007). The system UI owns settings of its own too, as the
service `ui`. Any other caller is refused with an error that says so;
one whose program cannot be told now is answered "unavailable", to try
again.

### Reset and backup

- **From the system UI, the device's owner can reset** a service's or an
  app's settings to their defaults; a factory reset erases them all
  (RFC 0010).
- **Backup and restore,** to external storage or a computer, is a later
  RFC.

### Settings itself

Settings runs in the system program (RFC 0007) and is among the first
services ready: most others wait for it
(RFC 0006). Its own state is the schemas and the
values.

### Starting values

To be measured:

| Limit | Value |
|---|---|
| A key | 64 characters |
| A string | 256 characters |
| A secret | 1 KB |
| Settings per owner | 128 |
| The wait before a write | 1 s |

## Alternatives

- **One string, `<owner>.<key>`,** with registered owners and the longest
  prefix winning. One field, but string rules to get right, and an app
  could name another's owner.
- **One file per key.** Each write small, but many tiny files, and names
  too long for littlefs.
- **SQLite.** Queries are not needed, and a commit on littlefs took
  seconds in the previous system.
- **Values on the topic.** No fetch after a change, but secrets would
  travel on a topic any native code can read, and messages have a fixed
  size.
- **Secrets under the unlock key** (RFC 0014). Safer at rest, but Wi-Fi
  could not connect until the owner's first unlock after a restart.
- **Keys unregistered,** any owner writing any key. Nothing to declare,
  but no screens could be drawn, and typos would make new settings.

## Costs and risks

- **Every value in memory,** bounded per owner; the pools' sizes need
  measuring with real schemas.
- **A write per burst of changes:** flash wear in proportion to how often
  settings change.
- **Secrets are readable by native code** on the device, with the flat
  build.
- **A schema registered at every start** costs time at boot, in
  proportion to the number of settings.
