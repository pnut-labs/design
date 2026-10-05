# RFC 0009: Packages

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-04
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

How apps are packaged, signed, installed, updated and removed. A package
is a zip archive holding the manifest, the code and a signed list of its
files. Packages, a service, checks and installs every package, whether it
is built into the firmware, brought over USB or downloaded from a
repository. Installed apps live on a read-only partition that only
Packages opens for writing. Each app has data and cache storage of its
own; external storage takes a permission, and permissions are asked for
when an app first needs them.

## Problem

The RFCs so far leave these to a Packages RFC:

- [RFC 0007](0007-services.md) makes Packages the service that owns the
  installed apps, native and WebAssembly, and the permissions each has
  been granted; the runtime keeps the grants of running apps and follows
  Packages' changes.
- [RFC 0008](0008-runtime.md) defines the manifest, and leaves the package
  file, installing, the store and signing to this RFC, with the checks an
  install makes: the host API level, the WebAssembly features, the
  hardware an app needs, and memory beyond the device's budget.
- [RFC 0002](0002-build-mode.md) lets the owner install native apps: over
  USB only, with a switch in Developer options, each install confirmed on
  the screen, and tied to the firmware version. A native app joins the
  manager through its `add` and `remove` commands
  ([RFC 0006](0006-manager.md)).

This RFC says what the places for installed apps and their data are, and
how they are protected; where they lie in flash, and their sizes, are set
by the file-layout RFC.

## Proposal

### Packages installs

Packages is the only part of the system that installs, updates and
removes apps. Whatever brings a package (the store, a computer over USB)
hands Packages the file; that is what RFC 0007 calls the installer.

- **Checking and writing** run on the data program's workers, which are
  there for flash (RFC 0007).
- **Checking a WebAssembly module** uses WAMR's loader. In the flat build
  every program is linked into one image, so code used by both Packages
  and the runtime is there once.
- **Asking the owner** follows RFC 0004: Packages announces that an
  install waits for confirmation, the system UI asks the owner, and sends
  the answer back to Packages.

### The package file

A package is a **zip archive**, made by the SDK's packaging tool, which
also compiles the manifest (RFC 0008). zlib, with zip support, is in
nuttx-apps.

Its extension tells the two kinds apart: **`.wpk`** for a WebAssembly app
and **`.npk`** for a native one, with the MIME types
`application/vnd.pnut.wasm-application` and
`application/vnd.pnut.native-application`. The extension is a label for
people and tools: Packages goes by what is inside (the manifest, and
whether the code is a WebAssembly module or an ELF file), and refuses a
package whose name does not match its contents.

| Inside | What |
|---|---|
| The manifest | compiled, as RFC 0008 defines it |
| The code | a WebAssembly module, or for a native app an ELF file |
| The icon and the translations | as the manifest refers to them |
| The file list | every other file's path, size and SHA-256 hash |
| The developer's signature | of the file list |
| The project's signature | of the file list, for packages in the project's repositories |

Signing the file list signs every file, since each file's hash is in it.

### Signing

- **The developer's key.** The SDK makes an Ed25519 key for each
  developer (libsodium, in nuttx-apps, provides Ed25519), and every
  package is signed with it. An app's id is tied to the key it was first
  installed with: an update must carry the same key.
- **The project's key.** The project signs the packages in its own
  repositories; its public key is part of the firmware. Precompiled apps
  (RFC 0008) will need this signature.

A signature says who made or distributed a package, not that its code is
safe: WebAssembly apps are confined no matter who made them, and native
apps run with full rights no matter who made them (RFC 0002).

### Where packages come from

| Source | What | Needs |
|---|---|---|
| **The firmware** | native and WebAssembly apps built in | — |
| **A computer, over USB** | native or WebAssembly apps | Developer options turned on, and each install confirmed on the screen (RFC 0002) |
| **Repositories** | WebAssembly apps only | — |

How a computer talks to the device over USB is designed in the
developer-access RFC.

**Repositories** work like Homebrew's taps or F-Droid's repositories: a
signed index (each app, its versions, sizes and hashes, and where its
package file is) and the package files, on any HTTPS server.

- **The project runs the main repository;** its key is in the firmware.
- **The owner can add others,** confirming each one's key when adding it.
  Their packages carry their developer's signature; the index vouches for
  each file's hash.
- **Packages downloads** indexes and packages itself, on its workers. It
  is native code, so the lack of network access for apps (RFC 0008) does
  not hold it back.
- **Updates:** Packages looks for them when the store asks, and by itself
  every night, when a network is up. By default updates install by
  themselves at night; the owner can make them manual in Settings, and is
  then told what is new. An update waits until the app is not open.

The store's screens are designed with the system UI.

### The package store

Installed apps live in **the package store, a partition of their own,
mounted read-only.** Only Packages changes it, and only to install, update
or remove an app:

1. it switches the partition to read-write;
2. it writes the new files beside the old ones;
3. it checks every file it wrote against the file list;
4. it switches the new files in by renaming them;
5. it switches the partition back to read-only.

A removal only deletes, between the two switches.

- **The switch happens in place,** without unmounting. NuttX's littlefs
  can be mounted read-only, and then refuses every write, down to the
  flash; that state is a single flag. NuttX defines `MS_REMOUNT`, which
  changes a mounted file system's flags, but does not implement it. One
  general commit in the NuttX fork lets littlefs switch in place
  ([RFC 0003](0003-repositories.md)).
- **Nothing else is affected:** other apps keep running and starting, and
  read the partition as usual. littlefs renames atomically, so a reader
  sees the old version or the new one, never part of each.
- **Changes that are waiting are made together,** in one read-write spell.
- **Built-in apps** are in the firmware, which is read-only anyway.

Files are checked when they are written, not each time they are read: the
partition is read-only, WebAssembly apps cannot reach it at all, and native
code is trusted (RFC 0002) and could change anything anyway. Changing an
installed app's files by hand, with developer tools, is the developer's own
business.

### Installing

1. **The file list.** Packages checks its signatures and every file's
   hash.
2. **The manifest and the code:**
   - the host API level is one the runtime serves (RFC 0008);
   - the module loads in WAMR, using only the accepted WebAssembly
     features (RFC 0008);
   - the hardware the app needs is present (Board's description,
     RFC 0007);
   - the memory it asks for is within the device's budget (RFC 0008);
   - a native app was built for this firmware (RFC 0002): its manifest
     names the firmware build it was made for, a field added to
     RFC 0008's manifest for this;
   - the package store has room.
3. **An update** has the same id, the same developer key and a higher
   version number. Anything else is refused, downgrades included.
4. **Confirmation:** an install over USB needs Developer options turned
   on, and is confirmed on the screen (RFC 0002); an install from the
   store is the owner's own choice there.
5. **Unpacking** into the package store, in a read-write spell (above);
   what was written is checked against the file list, and the archive is
   deleted. Packages keeps the developer key the app was installed with,
   for its updates. The new files are written beside the old ones and
   take over only when complete, so a failed install leaves the previous
   state.
6. **Announcing.** Packages announces the change, and each service takes
   what concerns it: Settings the settings schema, Notifications the
   channels, Messages the transport, the runtime the actions and intents
   (RFC 0008). A native app is added to the manager (RFC 0006).

The firmware's `init.rc` does not list installed apps, so at every start of
the device Packages adds each installed native app to the manager again,
through RFC 0006's `add`.

### Apps' storage

Each app has two places of its own, on a partition that can be written,
apart from the package store:

| Storage | For | Cleared |
|---|---|---|
| **Data** | what the app keeps: its state, its databases | when the app is removed, or by the owner in Settings |
| **Cache** | what can be made again: downloads, thumbnails | also by the system when space runs short, apps used longest ago first |

Both are kept across updates.

**External storage** is any mount point that is not the system's: a
memory card, a built-in eMMC, a USB drive. Two permissions reach it, each
asked for when first needed like any other:

| Permission | Gives |
|---|---|
| **A folder of its own** | a folder for the app on each external storage, for large data such as maps or recordings; deleted with the app |
| **Shared files** | the whole of each external storage, shared with the owner's own files and other apps, such as music for a player |

How apps read and write these, and how much each may use, is designed in
the storage RFC.

### Permissions

Packages keeps every app's grants. A permission is asked for **when it is
first needed**:

1. The runtime holds the call that needs it (calls are asynchronous,
   RFC 0005) and asks Packages.
2. Packages announces the question; the system UI asks the owner and
   sends the answer back.
3. Packages records it and announces the change; the runtime passes the
   call on, or answers it with "denied".

- **An app can ask ahead,** at any time while it is open: for example
  for background work at its first run, after explaining why on its own
  screen.
- **An app that is not open cannot be asked for:** a call that needs a
  permission not yet decided is refused, and the owner gets a
  notification that the app wants it.
- **A refused permission is not asked for again:** calls that need it,
  and the app asking ahead, are answered at once with "denied". Only the
  owner can change it, in Settings.
- **Built-in apps ask like any other app;** the firmware grants them
  nothing in advance.
- **Native apps have no permissions:** they run with full rights
  (RFC 0002), which the owner is told when installing one.

### Updates

- An app that is open is not updated until it is closed. One that is
  running otherwise is stopped, with `stop` and the deadline (RFC 0008),
  replaced, and started again if it was in the background. Its data and
  cache are kept.
- `init` tells the app it was updated, and from which version; the app
  migrates its own data.
- Permissions the new version adds are asked when needed; those it no
  longer lists are dropped.

### Removal

- The app is stopped. Its files are deleted from the package store, in a
  read-write spell, and its data, its cache and its folders on external
  storage are deleted.
- Packages announces the removal, and each service drops its own part:
  Settings the app's settings, Notifications its channels and
  notifications, Messages the transport (the messages themselves stay),
  Time its timers and scheduled runs, the runtime its actions, intents and
  the owner's choices of it as a default (RFC 0008).
- A native app is removed from the manager (RFC 0006).

### Built-in apps

| | Native | WebAssembly |
|---|---|---|
| **Comes with** | the firmware | the firmware |
| **Updated by** | firmware updates only | firmware updates, or the project's repository |
| **Can be** | disabled, not removed | removed, and installed again |

- **A built-in WebAssembly app updated from the repository:** the update
  goes into the package store and takes over from the firmware's copy. A
  firmware update brings its own copy; the higher version number wins.
- **A removed built-in app** stays removed through firmware updates, until
  the owner installs it again, from Settings or the repository. Its copy
  stays in the firmware, which cannot be written.

## Alternatives

- **An installer in the runtime program.** WAMR would be at hand, but
  native apps would go through the WebAssembly runtime.
- **An installer in the system UI.** The screen would be at hand, but the
  system UI would carry the checking and the flash writes.
- **tar,** or **a format of the project's own.** tar reads files only in
  order; a new format needs its own tools. Zip is read by standard tools,
  and any file in it can be reached directly.
- **The icon and translations inside the WebAssembly module,** as custom
  sections. One file, but the engine keeps the module in memory while the
  app runs, so they would take memory in every running app.
- **Unmounting the package store and mounting it again** for each
  change. No change to NuttX, but apps could not start while it was
  unmounted, everything reading it would have to wait, and a file left
  open would stop the change.
- **The package store on a writable partition,** with the apps' data.
  One partition fewer, but nothing would stop a write that goes astray.
- **A read-only image** (such as ROMFS) rebuilt for every change. Truly
  read-only, but every install rewrites the whole partition.
- **Other extensions:** `.pnut` and `.pnutx`, one name with the native
  kind marked; or one extension for both, leaving the kind to the
  contents, which people cannot see at a glance.
- **Keeping the archive after installing.** Less flash, but every start
  unpacks what it needs.
- **Checking files every time they are loaded.** It would catch flash
  corruption, but costs time at every start, for a partition that is
  read-only.
- **Only a developer's key,** or **only the project's.** The first cannot
  tell the project's packages apart; the second cannot tie an update to
  its author outside the project's repositories.
- **No signing yet.** Precompiled apps (RFC 0008) wait for signing, and
  updates could not be tied to their author.
- **A single store run by the project.** Simpler, but the owner could not
  add other sources.
- **Asking for every permission at install,** or **as a list at first
  open.** Both ask before the owner can see why the app needs it.
- **Built-in WebAssembly apps fixed like native ones,** updated only with
  the firmware. They would wait for firmware releases, and could not be
  removed.

## Costs and risks

- **Flash corruption in the package store goes unseen** until something
  breaks; the app is then installed again.
- **A change to NuttX's littlefs** is kept in the fork (RFC 0003), and
  carried to each new NuttX release.
- **The package store's size is fixed** by the flash layout: room for apps
  cannot be borrowed from data, or the other way round.
- **Checking a module at install** loads it in WAMR inside the data
  program, which takes memory outside the runtime's budget while other
  apps may be running; to be measured.
- **A lost developer key** means the app cannot be updated, only removed
  and installed again, which loses its data.
- **The project's key in the firmware:** if it leaks, a firmware update
  must replace it.
- **Updates take flash twice** for a moment, old and new side by side.
- **Repositories other than the project's** are trusted by their key; an
  app from one is still confined.
- **`.npk` is also MikroTik RouterOS's package format,** so tools that go
  by the extension alone may take one for the other.
- **Native apps cannot be installed** until the developer-access RFC
  defines how a computer talks to the device.
- **NuttX's ELF loader** has not yet been tried with pnut-os.
