# RFC 0008: The app runtime

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-04
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

WebAssembly apps run in one program, the runtime, on WAMR's interpreter.
An app is event-driven: it exports handlers and actions, and its calls to
services are asynchronous. It runs while it is open or called, and is
unloaded when it is closed, unless the device's owner lets it work in the
background. Apps call each other through intents. Each app is described by
a manifest, written as text and compiled to Protocol Buffers.

## Problem

[RFC 0002](0002-build-mode.md) makes every app distributed by the pnut-os
project a WebAssembly app, confined by its runtime, and leaves the choice
of runtime to its own RFC. The RFCs since have given the runtime its
duties:

- it is one program, and the manager sees only that program, not the apps
  inside it ([RFC 0006](0006-manager.md), [RFC 0007](0007-services.md));
- it marks each call with the app's identity, and services accept it
  ([RFC 0004](0004-layers.md));
- apps see no devices and open no topics: the runtime subscribes for them
  and passes on what their permissions allow, and copies streams into them
  ([RFC 0005](0005-communication.md));
- it enforces permissions, with the grants it keeps from Packages
  (RFC 0007);
- WebAssembly apps keep working across firmware updates (RFC 0002).

This RFC chooses the runtime and says how apps run, what they reach, when
they start and stop, how they call each other, and what describes them.

## Proposal

### The engine: WAMR, interpreted

The runtime is built on **WAMR** (WebAssembly Micro Runtime, by the
Bytecode Alliance), which nuttx-apps packages. WAMR fixes security bugs
regularly, so the runtime follows its releases: 2.4.5 at the time of
writing, where nuttx-apps defaults to 2.1.0; the version is one
configuration value in the fork.

- **Apps are interpreted.** WAMR can also run apps compiled ahead of time
  to machine code, which is faster. But such code is native: on a chip
  without memory protection, the device cannot check that it stays inside
  its own memory, so it is only as trustworthy as the compiler that made
  it. Precompiled apps therefore wait for the project to compile and sign
  them, in a later RFC.
- **A handler can be cut off.** WAMR can stop a function after a set
  number of instructions (`wasm_runtime_set_instruction_count_limit`). It
  is a build option that nuttx-apps does not offer yet: one commit in the
  fork ([RFC 0003](0003-repositories.md)).
- **A fixed list of WebAssembly features** is accepted, set in the
  runtime's reference documentation and checked when an app is installed.
  A feature joins the list when the interpreter supports it and an app
  needs it.

The runtime does not care which language an app is written in; the SDK is
designed separately.

### How apps run

The runtime is built like any other program (RFC 0004): one event loop
and a small pool of workers. Running an app's code is work that can take
long, which RFC 0004 gives to the workers; as in every program, what an
app asks for goes back through the loop, and only the loop touches the
runtime's own state.

- **The loop** holds the connections to services, the topics and the
  timers. Everything addressed to an app goes into that app's queue.
- **The workers run the apps' handlers.** Each running app is given to one
  worker when it starts. An app's handlers run one at a time, so an app
  needs no locks; different apps on different workers run at once.
- **Each handler has a time limit,** enforced as an instruction limit. An
  app that runs over it is stopped, as if it had crashed. Long work is
  split by the app into steps, as in a browser.
- **Calls are asynchronous** (RFC 0005): a call returns at once, and its
  answer comes later as an event.

### What an app exports

An app is one WebAssembly module. It exports:

- **`init`,** called once when the app is loaded, with the reason: opened,
  an action, an intent, background work, start-up;
- **an event handler:** answers to its calls, events from services and
  topics, timers, the system UI's events for its screens, `stop`;
- **its actions:** one export for each action in the manifest (below),
  called with a small argument, such as which message a notification is
  about.

### What apps reach

- **Services' public interfaces** (RFC 0005). The app encodes a request
  with nanopb or any Protocol Buffers library and hands it to the runtime,
  with the interface, method and version. The runtime checks the method's
  permission against the app's grants and passes the request on.
  - The runtime keeps one connection to each service it uses, shared by
    all apps, and puts the calling app's identity in each message's
    header, in a field added to RFC 0005's header for it. Services accept
    that field only on connections from the runtime, and native callers
    leave it empty.
  - Answers and events come back to the app as events.
- **Topics:** the app subscribes through the runtime, to the topics its
  permissions allow (RFC 0005).
- **The runtime's own functions:**
  - a log, marked with the app's identity;
  - clocks;
  - timers: ordinary ones don't wake the device; wake timers go through
    Time, at a minimum interval;
  - random numbers;
  - cryptography, done natively for speed: hashes, message authentication,
    ciphers, signatures and key agreement (keys kept by the system belong
    to Security);
  - which hardware the device has, from Board's description (RFC 0007);
  - the app's own settings, through Settings;
  - intents (below).
- **Part of WASI,** the system interface WebAssembly toolchains expect,
  version "preview 1": clocks, random numbers, standard output and error
  (into the log), and exit. Nothing else: files and sockets answer "not
  supported".

Each of these is designed with its owner:

| What | Where |
|---|---|
| Screens | the system UI's RFC |
| The package file, installing, the store, signing | the Packages RFC |
| Keys kept by the system | the Security RFC |
| Storage for apps (files, databases) | an RFC of its own |
| Network access for apps | no owner yet: until one is given, apps have no network |

### Life cycle

| State | What it means |
|---|---|
| **Not running** | nothing loaded, no memory used |
| **Open** | the owner is using it |
| **Called** | started for an action, an intent or a scheduled run, without being opened |
| **Background** | closed, but allowed to stay (below) |
| **Stopping** | asked to stop, finishing |

**An app starts** when the owner opens it (its `open` action), when one
of its actions or intents is called, when its scheduled run is due, or at
start-up if it may run in the background. Something meant for an app that
is not running waits in the service that holds it: a message to send
through the app's transport stays queued in Messages.

**An app stops** when the owner closes it, when it was called and says it
is done (or its time runs out), when room is needed for another app
(below), or when the owner stops it from Settings:

1. The runtime sends it `stop`. It saves its state and says it is done;
   until then, it still gets answers to the calls it made.
2. If it has not finished by the deadline, the runtime stops delivering to
   it.
3. It is unloaded at once and its memory returned.

**Running apps share a memory budget,** set for each device in Board's
description: what the system leaves free. Each app counts for the memory
its manifest asks for, plus what the runtime itself needs for it, from the
moment it starts; it can grow up to that but not beyond. How many apps can
run at once follows from it.

To start an app that does not fit, the runtime first makes room: it stops
other apps, the least recently used first, until the new one fits. They
are stopped in the usual way (above), with `stop` and the deadline,
because every app still loaded is doing something (in the background, or
called) and may not have saved its state; the new app waits for them. The
app the owner has open is never stopped for this, and a background app
stopped this way starts again when there is room.

An app that does not fit even then is not started, and the system UI tells
the owner. An app that asks for more than the device's whole budget is not
installed.

### Background work

By default an app runs only while it is open or called. Two kinds of
background work are declared in the manifest, each a permission the owner
grants or refuses, and can withdraw:

- **Staying in the background:** after it is closed, the app stays loaded
  and is woken only by the events it subscribed to and its own timers. It
  also starts when the device does. For example, a mesh app that listens
  to its radio.
- **Scheduled runs:** the app is not kept loaded, but run every so often,
  for a few seconds each time. The runtime times these runs with the
  device's other wake-ups, so they are not exact. For example, syncing a
  calendar.

### Failures

- **A crash stops only that app:** a trap, running past its time limit, or
  asking for more memory than it may have. The others go on. The crash is
  logged with the app's identity and the reason.
- **An app that crashes several times in a row is disabled** until the
  owner turns it on again.
- **If the runtime itself crashes,** every app stops; the manager restarts
  the runtime (RFC 0006), and it starts the background apps again.

### Actions

An action is something the system can ask an app to do. It is declared in
the manifest with an id and a label, and is one of the app's exports.

- **Who calls them:** the system UI: a notification's buttons, widgets,
  the app list (opening an app is its `open` action) and Settings. Other
  apps call them through intents, if the action allows it.
- **An app that is not running** is started for the action, and stopped
  again when it says it is done or its deadline passes, unless it is open
  or in the background. The
  owner asked for it, so no permission is needed.

### Intents

An intent asks another app to do something. It names its target in one of
two ways:

- **Explicitly:** an app's id and one of its actions.
- **By kind:** a verb and a type: view an `https:` URL, view a
  `text/html` file, share `text/plain`, pick a contact. Types are MIME
  types, and URL schemes written as XDG does on desktop Linux
  (`x-scheme-handler/https`). The project defines the
  system's verbs; apps can add their own, with reverse-DNS names. Each app
  lists in its manifest the kinds it handles, each with one of its
  actions.

**Which app handles a kind** is chosen as XDG's `xdg-open` does on Linux:

1. the owner's choice, kept in Settings;
2. otherwise the firmware's list of defaults, `/etc/apps/defaults`, if
   the app it names is installed;
3. otherwise the only installed app that handles it;
4. otherwise the system UI asks the owner, with "always use this", which
   becomes the owner's choice.

When a default app is removed, the next rule applies. The list of
defaults is a text file like XDG's `mimeapps.list`, with one kind per
line:

```
view  x-scheme-handler/https  org.example.browser
view  text/html               org.example.browser
share text/plain              org.example.messages
```

- **Answers:** an intent can ask for an answer, such as the contact that
  was picked. It arrives as an event.
- **Data** travels inside the intent, up to a size limit. Sharing a file
  is left to the storage RFC.
- **Who may send:** the system UI, and the app the owner is using. An app
  in the background or called for something cannot, so apps cannot start
  each other unseen; the owner tapping a notification is the system UI
  sending.
- **The target** sees which app sent the intent and can refuse. Only
  actions marked for other apps can be reached.

Services do not send intents: calls go down, and apps are above them
(RFC 0004).

### Limits

Starting values, to be measured on each device:

| Limit | Value |
|---|---|
| Memory for all running apps together | set for each device in Board's description |
| Workers | 1, as RFC 0007 sets for every program |
| A handler | the instructions of about 100 ms |
| `init` and `stop` | about 1 s |
| Deadline for stopping | 3 s |
| An app's memory | what its manifest asks for, within the budget |
| Calls an app has waiting for answers | 8; another fails at once with "busy" |
| Timers per app | 8 |
| Events waiting for an app | 32; topic events are merged to the latest value, and the app is told it missed some |
| Wake timers | at most once a minute |
| Scheduled runs | at most every 15 minutes, 10 s each |
| Crashes in a row before an app is disabled | 3 |
| An intent's or action's data | 4 KB |

The previous system measured WAMR 2.1.0's interpreter at 89 KB of code and
about 160 KB of memory for each running app.

### Compatibility

- **Host API levels.** The runtime's functions are numbered in levels.
  Each app states the level it was built for, and the runtime keeps every
  earlier level working; a new level only adds.
- **Services' interfaces** have versions (RFC 0004); the runtime passes the
  version the app asked for.

### The manifest

Each app has a manifest. The developer writes it as text, in TOML, and the
SDK's packaging tool compiles it into a Protocol Buffers message, defined
in a `.proto` file like every interface (RFC 0005). The device reads it
with nanopb, so it needs no text parser, and fields are added by number
without breaking older readers.

| Part | What it holds |
|---|---|
| **Identity** | the id, in reverse-DNS form (`org.example.mesh`); the name and description; the version shown, and a version number that only goes up; the publisher; the icon |
| **Code** | the module; the host API level; the memory it asks for: the most it will use, like Java's `-Xmx` (the SDK's tool writes the same maximum into the module); for a native app, the firmware build it was made for ([RFC 0009](0009-packages.md)) |
| **Permissions** | each with the reason the app needs it, shown when the owner is asked |
| **Hardware** | what it **needs**: without it the app is not installed and does not start (a mesh app without a LoRa radio); what it **can use**: without it the app runs with that part off, and asks the runtime what is present (a music player without a network). Names come from Board's description |
| **Actions** | each with its id, label and export, and whether other apps may call it |
| **Intents it handles** | each verb and type, with the action that handles it |
| **Background work** | staying in the background, scheduled runs and their interval, each with its reason |
| **What it offers the system** | a messaging transport (Messages), widgets, notification channels; what they show is designed in the system UI's and Notifications' RFCs |
| **Settings** | the schema: keys, types, defaults, ranges or choices, labels, groups. Settings stores the values; the screens that show them are designed with the system UI |

**Text is translated.** Every name, label and reason in the manifest is a
message id, looked up in the translation catalogues inside the package, in
the format the i18n RFC chooses (gettext is the likely one). What a
permission allows ("can send SMS") comes from the system's own
catalogue, so an app cannot change how it reads; only the app's reason
comes from the app.

An example, as written by the developer (the keys are illustrative; the
`.proto` file sets them):

```toml
id = "org.example.mesh"
name = "app-name"
version = "1.2.0"
version_code = 12
api = 1
module = "app.wasm"
memory = "256K"

[[permission]]
name = "radio.lora"
reason = "reason-lora"

[hardware]
needs = ["lora"]
can_use = ["gnss"]

[[action]]
id = "reply"
label = "action-reply"
export = "act_reply"

[[intent]]
verb = "share"
type = "text/plain"
action = "share"

[background]
stay = "reason-background"
```

The package file that carries the manifest, the module, the icon and the
catalogues is designed in the Packages RFC.

## Alternatives

- **wasm3 0.9.** Quiet from 2021 until August 2026, now active again; it
  checks modules fully, meters instructions and allows memory pages smaller
  than 64 KB. But it is only an interpreter, with no path to precompiled
  apps, and nuttx-apps packages an older version.
- **toywasm.** Small and light on memory, but the slowest, with a single
  maintainer.
- **wasmi.** Written in Rust, so it would bring a Rust toolchain into the
  build.
- **Precompiled apps now.** Faster, but native code that the device
  cannot confine; it needs the project's compiler and signing first.
- **A thread for each app** (the previous system). A stuck app holds up no
  one, but every running app costs a thread and its stack, against
  RFC 0004's one loop and small pool.
- **A program for each app.** The strongest separation, but the manager
  would see apps (against RFC 0006), and each would carry a whole runtime.
- **A blocking `main()`.** Familiar, and existing code ports easily, but
  it needs a thread per app and goes against RFC 0005's asynchronous rule.
- **All of WASI,** files and sockets included. Standard, but blocking,
  and it would reach around services and permissions.
- **Cryptography in Security.** One place for all of it, but a crossing to
  another program for every block of data.
- **A connection to each service for each app.** Services would learn the
  app once per connection, but every connection takes kernel memory
  (buffers of 1 KB each way by default), the scarcest kind.
- **A fixed number of running apps.** Simpler, but apps differ widely in
  memory: three small apps can use less than one large one.
- **Killing apps at once to make room,** without `stop`. The new app
  would start sooner, but background and called apps would lose what
  they had not saved, or would have to save all the time, wearing the
  flash.
- **Keeping closed apps suspended in memory,** as iOS does. They would
  reopen at once, but memory is too short to keep them.
- **Intents only by app id,** or **only by kind.** The first makes apps
  know each other; the second cannot reach one particular app.
- **A manifest read as text on the device,** in TOML, JSON or INI. No
  packaging step, but the device needs a parser (none for TOML in
  nuttx-apps), JSON allows no comments, and INI is awkward for lists.
- **Apps drawing their own settings screens.** More freedom, but every app
  would look different, and settings could not be changed without starting
  the app.

## Costs and risks

- **Interpreted apps are slow:** much slower than native code. Heavy work
  that is common, such as cryptography, is done natively by the runtime.
- **The instruction limit costs a little on every instruction;** how much
  is to be measured.
- **With one worker, apps take turns:** a slow handler holds up the others
  until its limit.
- **A bug in WAMR can break the confinement of every app,** so its
  security fixes have to be followed.
- **A crash of the runtime stops every app.**
- **Few apps run at once,** and background apps take memory from the
  others; opening an app can stop others, and wait up to their deadline.
- **Every host API level is kept,** so the runtime only grows.
- **Apps cannot be packaged without the SDK's tool,** since the manifest is
  compiled.
