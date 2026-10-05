# RFC 0011: Storage for apps

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

A Storage service, in the data program, owns apps' storage. Apps reach it
through the runtime, asynchronously: whole files for small state, large
files in fragments, and SQLite databases with their rows streamed back. A
large file, such as a podcast, can be handed straight to the service that
plays it, and any file lent to another app through an intent. Installed
apps have no quota; built-in apps have a fixed one.

## Problem

The RFCs so far give apps places to keep things, but no way to use them:

- [RFC 0008](0008-runtime.md): apps' calls are asynchronous, and WASI's
  file calls answer "not supported" because they block; sharing a file
  through an intent is left to this RFC;
- [RFC 0009](0009-packages.md): each app has data and cache; external
  storage takes one of two permissions, an app's own folder or the shared
  files; the system clears caches when space runs short;
- [RFC 0010](0010-file-layout.md): installed apps' data and cache are on
  the apps' storage, at `/var/opt/<id>`; built-in apps' are on internal
  flash, at `/var/lib/apps/<id>`; external storage is mounted under
  `/media`.

## Proposal

### The Storage service

**Storage** is a service in the data program, whose workers are there for
flash ([RFC 0007](0007-services.md)). It owns apps' storage (one owner,
[RFC 0004](0004-layers.md)):

- **apps' files and databases,** in every place below;
- **external storage:** it mounts each one under `/media` when it appears,
  prepares the apps' storage when the owner confirms, mounts the apps'
  partition at `/var/opt`, and announces when storage comes and goes
  (RFC 0010).

The runtime checks an app's identity and permissions, then passes its call
on, marked with the app (RFC 0005). Storage keeps each app inside its own
places, since FAT, on external storage, has no permissions of its own.

RFC 0007's list of services gains Storage, in the data program.

### An app's places

An app names a place and a path inside it, never a full path; a path that
would leave the place (`..`) is refused.

| Place | Where | Needs |
|---|---|---|
| **Data** | `/var/opt/<id>/data`, or `/var/lib/apps/<id>/data` for a built-in app | — |
| **Cache** | `/var/opt/<id>/cache`, or `/var/lib/apps/<id>/cache` | — |
| **Its own folder** on an external storage | `/media/<kind><n>/apps/<id>` | the permission for its own folder (RFC 0009) |
| **The shared files** on an external storage | `/media/<kind><n>`, except other apps' own folders | the permission for shared files (RFC 0009) |

Large content an app keeps for the owner, such as recordings, maps or
downloaded podcasts, always lives on external storage, in its own folder,
never in its data on internal flash.

### Files

Every call is asynchronous: it returns at once, and its answer comes as an
event (RFC 0005).

- **Whole files,** for the usual small state:
  - *read:* the contents come back in the answer;
  - *write:* the file is replaced in one piece, written beside the old one
    and renamed over it, so a crash never leaves half a file;
  - *list* (paged), *delete*, *rename*, *make a folder*, *information*
    (size, time).
- **In fragments,** for large files such as an ebook, a map or a
  podcast, which are never read whole into an app's memory:
  - *at an offset:* the app opens the file, reads or writes a fragment
    at any offset, and closes it; for jumping around, like a chapter of an
    ebook;
  - *as a stream:* the app opens a stream from an offset, and Storage
    sends the fragments in turn whenever the app has room for them, with
    credit (RFC 0004), with no request for each; for reading straight
    through. Writing works the same way the other way round, for a
    download, with Storage giving the credit.

  Open files and streams close when the app stops.

### Databases

Storage runs **SQLite**, which nuttx-apps packages, for apps' databases:

- an app opens a database by name, in its data;
- it sends SQL with parameters; the rows come back as a stream, with
  credit (RFC 0004), and changes are counted in the answer;
- transactions work as in SQLite, on one open database;
- a statement that runs too long is interrupted;
- an app reaches only its own databases: attaching another database file
  and loading extensions are refused.

Services such as Messages and Contacts link SQLite directly; in the flat
build its code is in the image once.

### Space

- **Installed apps** have no limit each. Settings shows what each app
  uses (data, cache, its own folders). When the apps' partition runs
  short, Storage clears caches, of the apps used longest ago first
  (RFC 0009); when it is full, writes fail with "no space".
- **Built-in apps** each have a fixed limit, set in the firmware, since
  the system partition is small; beyond it, writes fail with "no space".

### Handing a file to a service

Media do not go through the app. Playing a podcast by reading it into an
interpreted app and passing it on to Audio would copy every byte twice,
through the slowest part of the system. Instead, the app hands the file
to the service that uses it:

1. the app asks the service, for example Audio, to play a file from one
   of its places;
2. the service asks Storage for it; Storage checks that the file is that
   app's, and passes the service the open file (`SCM_RIGHTS`, RFC 0005);
3. the service, which is native, reads it directly. The app controls what
   it does, such as play, pause and seek, and gets events such as the
   position.

A service can be handed a file to write in the same way: a recording, or a
photo, into the app's place. Each service's interface says what it can be
handed; this RFC defines only the handing over.

### Lending a file through an intent

An app sending an intent can lend one of its files to the app that
handles it (RFC 0008):

- the runtime puts a grant for that file in the intent;
- the receiving app reads the file through Storage with the grant: that
  file only, read-only, with no copy;
- the grant ends when the intent does: when the receiving app answers it,
  or stops.

### Native apps

Native apps are trusted (RFC 0002): they use POSIX files directly, in
their own places, and may use Storage too, for databases for example.

### Limits

Set for each device, in Board's description (RFC 0007); starting values,
to be measured:

| Limit | Value |
|---|---|
| A whole file read or written in one call | 64 KB |
| A fragment: one read or write at an offset, or one piece of a stream | 16 KB |
| Open files and streams per app | 8 |
| Open databases per app | 4 |
| A statement's run time before it is interrupted | 1 s |
| Free space on the apps' partition below which caches are cleared | 10% |

## Alternatives

- **The runtime doing apps' file work** on its own workers. One crossing
  fewer, but with one worker a slow write to a memory card would hold up
  every app.
- **Only a POSIX-like API.** Familiar, but every small state file would
  take an open, a read and a close, each answered by an event.
- **Only whole files.** The simplest, but large files would have to be
  read whole into an app's memory.
- **SQLite compiled into each app.** Roughly 1 MB of WebAssembly per app,
  and slow when interpreted.
- **A key-value store** instead of SQL. Smaller, but no queries.
- **WASI's file calls.** Standard, but they block a worker for every
  read and write.
- **A limit for every installed app.** Fair, but apps' needs differ too
  much to set one, and the owner already sees and clears what each uses.
- **Media passed through the app,** in fragments, to the service that
  plays them. No handing over to design, but every byte copied twice
  through the interpreter.
- **Copying a file into the receiving app's cache** for an intent. Simple,
  but a large file takes double the space.

## Costs and risks

- **Every read and write crosses programs and is copied** into the app's
  memory, so large files are slow.
- **SQLite's speed on a memory card** has not been measured. On internal
  flash, the previous system measured 2–4.5 s for each commit on littlefs.
- **A service handed a file uses it beyond the runtime's checks;** it is
  native, and trusted (RFC 0002).
- **A fault in Storage stops apps' storage** until the manager restarts it;
  apps get errors meanwhile, and try again.
- **Built-in apps share about 2 MB** of internal flash with the services
  (RFC 0010).
- **A memory card pulled out during a write** loses what was not yet
  written; littlefs keeps the last complete state.
