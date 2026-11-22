# RFC 0037: Apps and the system UI

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-11-16
- **Last changed:** 2026-11-22
- **Supersedes / superseded by:** —

## Summary

How an app, a program of its own, reaches the screen: one connection to
the system UI, who the app is, when it runs and stops, what it may put on
the screen, how keys and back reach it, how it opens other apps, how it
works while it is not open, and what happens when it fails. Apps draw
only on a canvas; everything else they describe as layouts, and the
system UI draws them. Native apps come first; WebAssembly apps use the
same interface through the runtime.

## Problem

- [RFC 0019](0019-system-ui.md) has apps describe their screens from
  building blocks, and leaves "the exact catalogue of building blocks,
  and how screens are encoded" for later.
- [RFC 0008](0008-runtime.md) gives WebAssembly apps a life cycle, a
  memory budget, intents and actions, all run by the runtime. Native apps
  have no runtime to do it for them.
- The first apps are native programs apart from the system UI, built
  before the runtime: Messages, Phone and Contacts, Calendar, Reminders,
  Notes, Weather, Clock, Calculator and Files, in that order, in English,
  Polish and French from the start.
- The visual design's catalogue of templates exists (the design canvas's
  Components page): what each template shows and which events it sends.
  What is missing is the plumbing between an app and the system UI.
- The manager ([RFC 0006](0006-manager.md)) restarts a service that exits
  on its own, so an app could never be closed; it caps service names at
  31 bytes; adding and removing services is planned, not built.

## Proposal

### What an app draws, and what the system UI owns

- **An app draws only on a canvas** (RFC 0019). Every other screen it
  sends as a layout: a template filled with blocks, which the system UI
  draws for the kind of screen it has.
- **What must work outside an app is the system UI's:** incoming calls
  and calls in progress, an alarm ringing, the lock and the SIM PIN,
  banners and the notification centre, prompts. A call can be answered,
  muted and ended while the device is locked, and after the Phone app
  has failed.

### The connection

Each app holds **one connection to the system UI**, opened by the app and
kept for its life, in the system UI's public interface
([RFC 0004](0004-layers.md), RFC 0019), over the local sockets of
[RFC 0005](0005-communication.md). The app sends requests; the system UI
sends events back on the same connection. Events have no replies, so the
system UI keeps its own timers for answers an app owes it (below,
Failures).

| From the app | Does |
|---|---|
| **Open** | the first message: the app's id, the catalogue level it was built for, what it can do (a canvas, images) |
| **Show** | a whole screen: an id the app chooses, a template, its content, the focused item; or a sheet over the screen |
| **Update** | changes to the screen on top: a row's title, detail, value or mark; a field's text; a switch; the rows of a range; the middle softkey's label, or whether it is enabled; the title; progress |
| **Rows** | a page of rows the system UI asked for, naming the screen and the first row |
| **Text** | a long text block in chunks, written or read back |
| **Read** | the next chunk of the launch's data |
| **Close** | leave: the app is done, or back was pressed on its first screen |
| **Resume** | the app's resume token (below), at most 512 bytes |
| **Intent** | another app, explicitly or by kind (RFC 0008) |

| To the app | When |
|---|---|
| **Welcome** | the answer to Open: the kind of screen, rows per page, text limits, the languages in order, the region, the 12- or 24-hour clock, the theme; the catalogue level and, for WebAssembly apps, the host API level; the launch's reason (opened, action, intent, result, resumed, scheduled run, background start), its data's first chunk and total size, and the resume token |
| **The template's events** | below, Templates |
| **Back** | the owner pressed back |
| **Shown** | the app is on top again, after another app or a system screen; it answers with Show of its current screen, inside the key's window when it is still running |
| **Page needed** | rows from a given one, for a list |
| **Result** | the answer to an intent the app sent |
| **Stop** | save and exit |

**Who the app is.** An app's id is its service's name in the manager:
the line that declares Messages names it `org.pnut.messages`, so Service
states' `Who` gives the system UI the app's id from the connection's
`SO_PEERCRED`. The manager's names grow to 63 bytes (below, the
manager). A connection from a task the manager does not know is refused.
One from a program started at the developer's shell is accepted only
while Developer options are on, under the id it gives in Open, and is
logged as such, as [RFC 0030](0030-telephony.md) lets the developer's
shell do what the system UI may. That id may not be an app's: one that
has a manifest, or one that is open, is refused. A shell's program has
no resume token kept and no crashes counted, so it never changes an
app's.

**Versions.** Methods carry versions as every interface does (RFC 0005).
The templates and blocks are numbered together as one **catalogue
level**: a new field is free, a new template or block raises the level.
The system UI gives its level in Welcome, and answers `invalid` to a
template it does not know, so an app built for a newer level can fall
back.

### The life cycle

The states of RFC 0008, for native apps too, plus one:

| State | What it means |
|---|---|
| **Not running** | no task |
| **Open** | its screen is on top |
| **Behind** | another app was opened from it (below, Moving between apps), or the owner went Home; still running, or stopped with its resume token kept |
| **Called** | started for an action, an intent or a scheduled run, without a screen |
| **Background** | allowed to stay (RFC 0008's permission) |
| **Stopping** | asked to stop, finishing |

The system UI decides every change, since it alone knows what is on
screen; Service states and the manager carry them out for native apps,
and the runtime for WebAssembly apps.

- **Opening:** the system UI asks Service states to start the app, waits
  for Open, sends Welcome and draws the app's first screen. Nothing new
  is drawn for the first 0.7 s, which is one redraw anyway; after it, the
  Busy template with the app's name. An app whose last task has not
  exited yet, or one tapped while no other app may run, shows the Busy
  at once and starts as soon as it can; Home, its Cancel or another app
  forgets it. A task that opens after a newer start of the same app is
  told to stop, and its exit is no crash.
- **One app kept warm:** the app that was open before stays running, so
  going back to it is immediate. A third app, or memory running short
  under Board's budget ([RFC 0024](0024-board.md)), stops it first. The
  budget's values for native apps are measured on the device before they
  are set.
- **Stopping:** the system UI sends Stop; the app saves and exits within
  RFC 0008's deadline of 3 s. Past it, Service states stops the task
  (terminate, then kill). An app that closes, or whose connection
  closes, has 3 s for its task to exit too, and is then stopped the same
  way. The system UI follows Service states' services topic to see apps'
  tasks start and exit.
- **The resume token:** an opaque message of at most 512 bytes an app
  gives with Resume whenever its place changes: which screen, which item,
  a draft's id. The system UI keeps it in memory, and gives it back in
  each Welcome with the reason *resumed*, until the app gives another or
  an empty one, closes, or crashes, or the owner closes the app while
  waiting for it or when it isn't responding (below, Failures). A Close
  that answers Stop keeps it: the app saved its place as it stopped. So
  does an app taken off for not keeping up, which is no crash (below,
  Failures). The tokens of a few apps are kept
  (below, Starting values); a new app's replaces the oldest. An app does
  not write flash each time it is left; anything larger it saves itself.

### The screen

Layers, bottom to top:

| Layer | What | Who |
|---|---|---|
| 1 | Home, Menu, Settings | the system UI |
| 2 | **the open app's screen** | the app's layout |
| 3 | **a sheet** over it: Confirm, Choice, the Date, Time and Number pickers, Busy, Options | the app asks; the system UI draws |
| 4 | the status bar, banners, the Find bar | the system UI |
| 5 | incoming calls and calls in progress, an alarm ringing, the sleep and lock screens, the SIM PIN, prompts (permissions, installs, the USB port, the choice of app for an intent), a low battery | the system UI |

- **An app keeps its own stack of screens.** The system UI keeps the
  stack of apps (the chain, below) and, for the open app, only the
  screen on top: its description, the rows loaded so far, the focus.
  Opening a deeper screen is a Show with a new screen id; going back is
  the app's own Show of the screen before (below, Back).
- **At most one sheet** is open over an app's screen; the system UI
  closes it when the owner answers, and sends the answer. Back on a sheet
  cancels it. An app closes its own sheet (Busy, once the work is done)
  with Show of the screen beneath.
- **Options** belong to the screen: the app gives the list, with a letter
  for each and submenus; the system UI opens and closes it, and sends the
  option chosen.
- **The header** of every app screen carries the app's icon, from the
  system's icon set and named in the app's manifest, never chosen per
  screen; the title is the screen's, set by the app ("New message"), and
  changed with Update.
- **No template lacks a header,** and locked screens have none, so no app
  screen can pass for the lock, the PIN or a call. The screens of layers
  4 and 5 are the system UI's own code, not templates.
- **System screens are the system's alone:** banners, the
  notification centre, incoming calls and calls in progress, an alarm
  ringing. No app can show them. An app raises them only through the
  service that owns the event: a notification it posts
  ([RFC 0020](0020-notifications.md)) makes the banner; a call
  (Telephony) makes the call screens; an alarm (Time) rings full screen.

### Templates

The design canvas's templates, at catalogue level 1, with the events each
sends. Focus moves, paging through rows already sent, Find over loaded
rows, and opening and closing Options are the system UI's, never sent.

| Template | For | Events to the app |
|---|---|---|
| **List** (with views) | inboxes, contacts, recents, notes, files | row opened, option chosen, page needed, find (past 200 rows), back |
| **Detail** (with options and submenus) | a contact, an event, a reminder | row opened, option chosen, back |
| **Form** | a new event, a timer, an alarm | value changed, row opened, back |
| **Conversation** | a thread of messages, a USSD session | text submitted, older page needed, option chosen, back |
| **Editor** | a message, a note | text submitted, cancelled |
| **Choice** (sheet or screen) | one of several | choice made; back, or cancelled as a sheet |
| **Confirm** (sheet) | deleting, leaving with changes | confirmed, cancelled |
| **Empty** | nothing yet | middle action, option chosen, back |
| **Busy** (sheet or screen) | waiting, copying | cancelled |
| **Date; Time** (sheets) | a day, a time | date chosen; time set; cancelled |
| **Number** (sheet) | a value in a range | value set, cancelled |
| **Readout** | the dialler, the calculator, a timer, a stopwatch, an alarm's face | value changed, middle action, option chosen, back |
| **Month; Day** | Calendar | as Date, with marks on days; as List, with a time column |

- **Readout** is new to the catalogue: large monospaced lines, a hint, an
  optional row of keys for touch, letter keys typing digits. A running
  time is a typed value the system UI counts itself from its start, and
  redraws as the visual design's rules for e-paper allow; the app sends
  no update every second.
- **Waiting:** Calendar's week, images in apps (the image block, detail
  with an image), and the canvas. A picture is larger than a message
  (below), so they come with shared memory (RFC 0005).

### Whole screens, then changes

- **Show** sends a whole screen when it is new or replaced; **Update**
  sends only what changed. The system UI draws only what differs.
- **A message stays at most 4 KB** ([RFC 0023](0023-service-library.md)). A row is about 150 bytes
  within the limits the visual design sets (title, detail, value, icon,
  time), so twenty rows fit one message.
- **Rows are pulled:** the system UI asks for a page of 20 (`page
  needed`) and the app answers with Rows, naming the screen and the first
  row, so an answer for a screen that is gone is dropped. The system UI
  keeps up to 200 loaded rows per screen for paging and Find, fetches the
  next page ahead, and past 200 sends `find` to the app.
- **Anything over one message goes in chunks:** a long text block (a
  note) is written and read back in chunks of up to 3 KB, up to 32 KB per
  block, and an Editor's `text submitted` carries the first chunk and the
  total size, the app reading the rest with Text; an intent's or action's data larger than Welcome holds comes as
  its first chunk and total size, and the app reads the rest with Read.
  RFC 0008's 4 KB for an intent's data stays, and can grow later.

### Typing, keys and back

- **Typed text** reaches the app when it is committed (Enter, leaving the
  field). A field the app marks **live** also sends `value changed` as
  the text is echoed, every 0.5 s, for the dialler's matching contacts or
  Find in notes; never at every key.
- **Back is asked of the app first.** The system UI sends Back; the app
  answers with Show (its screen before), a Confirm sheet (a note with
  changes, a message half written), or Close (its first screen). An
  answer inside the key window (below, E-paper) is drawn in the same
  redraw; a later one costs a second; none is a hang (below, Failures).
- **Home is always the system's:** the key map's way Home (on a keyboard,
  Alt and Backspace held) never asks the app; the app becomes *behind*.
- **The primary action is the app's** (the middle softkey; with two
  function keys, the steering's E and Enter, RFC 0019): it gives the
  label, enables or disables it ("Send" with no text), and handles it.
  The apps' guidelines give its usual role in each template (open, send,
  set, done).
- **Command letters are the app's:** its developer gives each option its
  letter while designing the app, and the system UI handles the key. A
  letter the device's key map steers with (W, A, S, D, Q and E in the
  suggested map, RFC 0019), or one already used in the same menu, is
  shown without its letter and logged, so the option is still reached by
  moving to it; the SDK's simulator flags such a letter while the app is
  designed ([RFC 0021](0021-sdk.md)).
- **A canvas** gets the keys and touches on it, as RFC 0019 and
  [RFC 0027](0027-input.md) say.

### Moving between apps

- **The system UI dispatches intents** (RFC 0008): it alone can start a
  native app, show the choice of app and place its screen. Packages gives
  it the apps' handlers, the owner's defaults are settings, and the
  runtime hands it WebAssembly apps' intents.
- **The chain:** an intent from app A puts app B's screen on top; when B
  closes, A is on top again, started with its resume token if it was
  stopped. A chain holds 3 apps; a fourth drops the oldest, and back from
  the first then goes Home.
- **Results** come to the caller as an event, or, if it had to be started
  again, in its Welcome: the reason *result*, the result as the launch's
  data, and its resume token.
- **Opening an app from Menu or Home** starts a new chain; the app open
  before is kept warm or stopped (above, The life cycle).
- **Recent apps** lists recent chains, each brought back by starting its
  top app with its resume token; the apps beneath start when back reaches
  them.
- **Kinds the first apps handle:** `tel:` (Phone), `sms:` and sharing
  `text/plain` (Messages), picking a contact (Contacts), viewing a time
  (Calendar), viewing a file by its type (Files, then whoever handles
  it); `geo:` with Maps, later.

### When an app is not open

- **The service that owns an event posts its notification** on the app's
  behalf, since it knows which app handles it: Messages for a new message
  ([RFC 0029](0029-messages.md) already posts one per conversation);
  Time for alarms, reminders and a calendar event's alerts, from wake
  timers an app registers with their channel, title and action (Time's
  timers come in a later RFC; [RFC 0028](0028-time.md) leaves them out); Telephony for a missed call. The
  notification is the app's (its icon, its channel, the owner's answer
  for it, RFC 0020), and its actions start the app with a reason. So none
  of the first apps needs to stay in the background.
- **Staying in the background** (RFC 0008) is for later apps that must
  listen, such as a mesh app.
- **Scheduled runs** (RFC 0008) apply to native apps too: the system UI
  starts the app *called* with the reason *scheduled run*, timed with the
  device's other wake-ups; it does its work (Weather fetches a forecast
  and updates its widget), and says it is done or is stopped at the
  deadline.
- **Widgets, glances and status icons** (RFC 0019) take their content
  from a common store, `/var/lib/surfaces/<app id>/<name>`: one record
  for each surface the app declared, in the same blocks as a screen, an
  encoded message without its header that fits one message (4 KB, RFC
  0023), written beside the old one and renamed over it, as
  [RFC 0011](0011-storage.md)'s whole files are. The app writes it, in
  any state, or a service does on its behalf (Messages, the count of
  unread conversations); then the writer publishes an update event on
  the `surfaces` topic ([RFC 0005](0005-communication.md)), naming the
  app and the record. The event is only a nudge: the topic has many
  writers, so a missed event cannot be told, and on every event, and at
  its own start, the system UI reads again each record it shows (a few,
  small). It redraws them by the e-paper rules (below). The store is the
  content: an app need not run for its widget to show, and Home is whole
  after a restart. A declared surface with no record yet shows the app's
  icon and name, and the app is not started to fill it. Writers write
  only when what is shown changes: a time is a typed value the system UI
  formats, so "in 5 minutes" or a countdown is never rewritten. A
  WebAssembly app gives its records through a host function, and the
  runtime writes and publishes them, for the app's own id only.

### Languages and formats

- **The system UI formats** typed values in the owner's language and
  region: a time (as a clock, a short date, relative, a date and time), a
  number (with decimals and a unit from a small list), a duration, a
  size. An app sends text only where no type fits.
- **An app translates its own strings** ([RFC 0017](0017-i18n.md)), with
  libc's gettext in its own domain, catalogues at
  `/usr/share/locale/<language>/LC_MESSAGES/<app id>.mo` for built-in
  apps; plurals with `ngettext`. Strings with substitutions ("3 of 4")
  must be translated before they are filled in, which only the app can
  do. The system UI translates its own: softkeys, pickers, its screens.
  For WebAssembly apps the runtime translates, as RFC 0017 says.
- **A change of language or region restarts the device:** Settings asks
  to confirm, then saves the change and restarts at once; cancelled,
  nothing changes. The device never runs with a change saved but not in
  effect, and no program, the system UI included, watches for it.

### What the system UI enforces

Whatever the caller, the system UI checks:

- **Only the open app** may Show or Update. An app behind, called or in
  the background may post notifications and update its widgets, nothing
  on screen.
- **The templates and intents an app declares:** the manifest (RFC 0008)
  lists the templates an app shows, the kinds of intent it sends, and its
  widgets, glances and status icons; Packages gives the lists to the
  system UI, which refuses the others and draws no record the app did
  not declare.
- **The header's icon** is the app's; the screens of layers 4 and 5 are
  never an app's.
- **An app's screens are never drawn while the device is locked.**
- **A password field's text** goes only to the app the screen belongs
  to, is shown as dots, is wiped in the system UI once submitted, and is
  marked sensitive, so it is redacted wherever a message is logged
  ([RFC 0026](0026-log.md)).
- **Identity** is `Who`'s, or the runtime's `app` field on the runtime's
  own connection only.

For native apps these checks guard against mistakes, not intent: native
apps are trusted ([RFC 0002](0002-build-mode.md)). WebAssembly apps are
held to them by the runtime too.

### E-paper

The system UI gathers the keys of one action into one redraw (RFC 0019),
in a key's window of 0.5 s; updates an app sends on its own are gathered
too:

- an Update that arrives while a key's window is open joins its redraw:
  it is the answer to the owner's key;
- an Update nobody's key caused opens a window of 1 s, and the app's
  screen is redrawn at most every 2 s; one marked **now** is drawn at
  once (a timer reaching zero), one marked **quiet** waits for the next redraw
  anything causes (a counter);
- **an app answers within 200 ms** an event that needs a new screen, so
  it is drawn in the key's own redraw; this is the apps' contract;
- switching apps and returning Home take the full refresh the visual
  design waits for;
- **a changed widget, glance or status icon** is drawn only where it is
  on screen, as an Update nobody's key caused: gathered for 1 s, at most
  every 2 s; a status icon is quiet.

### Failures

| Failure | What the system UI does |
|---|---|
| **An answer owed is late** (a first screen, a page of rows, back) | after the key's window, it draws what it has; after 3 s, a screen of the system UI's own: "Waiting for *app*…" with Cancel, which closes the app; after 10 s, in its place: "*app* isn't responding", with Wait (another 10 s) or Close. Full screens, not sheets: the app's screen may not exist yet, and a hung app's screen is not the owner's to act on |
| **Events cannot be sent** for 3 s (the app stopped reading), or Open never comes after a start (10 s) | closes the connection, asks Service states to stop the app, and takes its screen off (RFC 0004: a client that stays behind is disconnected) |
| **The connection closes** (the app exited or crashed) | the app leaves the chain at once; the app before it, or Home, shows; logged with the app's id; no dialog. An app that crashes 3 times in a row is disabled, as RFC 0008 says, until the owner turns it on again |
| **The app exits before it opens** (its task gone on the services topic before Open) | the Busy goes at once; counted as a crash, as for a connection that closes |
| **The system UI itself restarts** (the manager restarts it) | Home; an app that reconnects has its Open answered with Stop |

### WebAssembly apps

- **One interface:** the system UI's interface is one `.proto`; the
  generator writes the C client for native apps and the runtime's
  bindings ([RFC 0023](0023-service-library.md)).
- **The runtime forwards an app's messages as bytes,** with the app's id
  in the header's `app` field, never decoding them, so fields its build
  does not know are not lost; events for an app wait in the runtime's
  queue for that app (RFC 0008).
- **The system UI starts and stops WebAssembly apps** by asking the
  runtime, as it asks Service states for native apps: one policy for
  both.
- **Welcome reports both levels:** the catalogue level and the runtime's
  host API level.

### Testing

- **The interface:** a fake app sends Show and Update through libpnut's
  Linux side, every template is drawn to an image kept with the tests,
  and the events are read back.
- **An app's logic:** against a fake system UI and fake services, without
  drawing.
- **An app's screens:** the fake system UI records what the app sends,
  the real renderer draws it, and the images go in the app's pull
  requests.
- **End to end:** in the simulator with the real manager
  ([RFC 0022](0022-build.md)); on the device, the same scripts.

### The manager

Two general additions to nxinit in the fork (RFC 0006):

- **A service option for no restart on exit,** which keeps the service
  startable: an app that exits is closed, not restarted. (Today a service
  that exits on its own is restarted, and a `oneshot` one is removed.)
- **Service names up to 63 bytes,** so reverse-DNS ids fit; Service
  states' `Who` answers grow with them.

Adding and removing services (`add`, `remove`) stay as RFC 0006 plans
them, for installed native apps.

### Starting values

| What | Value |
|---|---|
| A message | 4 KB (RFC 0023) |
| A resume token | 512 bytes |
| Apps whose resume token is kept | 4 |
| Rows per page; rows kept per screen | 20; 200 |
| A text block; a chunk | 32 KB; 3 KB |
| Apps in a chain | 3 |
| Apps kept warm | 1 |
| A key's window; a live field's echo | 0.5 s; 0.5 s |
| Nothing drawn after opening an app | 0.7 s |
| An app's answer inside a key's window | 200 ms |
| Updates nobody's key caused | gathered for 1 s, drawn at most every 2 s |
| Waiting; not responding | 3 s; 10 s |
| Open after a start | 10 s |
| Events that cannot be sent, before an app is closed | 3 s |
| Stopping | 3 s (RFC 0008) |
| Crashes in a row before an app is disabled | 3 |
| Connections to the system UI | 8 (RFC 0023): the running apps and the runtime |

### The order of the work

1. **The interface and the first templates:** the `.proto`, the system
   UI serving List, Detail, Confirm, Empty and Busy, the identity rule,
   the fake app drawing templates to images; the manager's two additions
   in the fork.
2. **The life cycle, and Messages:** Menu starts an app, Welcome, Stop,
   resume tokens, the e-paper gathering, the failure rules; Messages'
   inbox and conversations on the device.
3. **Form, Choice, Date, Time, Number;** Phone and Contacts (the Contacts
   service first).
4. **Intents and the chain;** Recent apps.
5. **Notifications on apps' behalf,** widgets from the common store;
   Calendar, Reminders, Clock.
6. **Readout, long text, Month and Day;** Notes, Weather (with
   placeholder data until it has a source), Calculator, Files.
7. **The runtime's path,** when the runtime is built; images and the
   canvas after it.

### Changes to other RFCs

Once this RFC is accepted, a follow-up changes:

- **RFC 0019:** the system UI keeps the stack of apps, and each app its
  own screens; apps give their command letters; a change of language or
  region restarts the device; system screens only through their
  services; the Home chord in the keys table; widgets, glances and
  status icons take their content from the common store, not through
  the interface.
- **RFC 0008:** an intent's or action's data may come in chunks; the
  manifest lists the templates an app shows, the kinds of intent it
  sends, and its glances and status icons beside its widgets; a host
  function gives a widget's, glance's or status icon's record; native
  apps follow this RFC's life cycle.
- **RFC 0006:** the no-restart option and names up to 63 bytes.
- **RFC 0010:** `/var/lib/surfaces`, the common store of widgets,
  glances and status icons, on internal flash.
- **RFC 0009:** Packages hands apps' intents, actions and the owner's
  defaults to the system UI, which dispatches them, not to the runtime;
  removing an app deletes its folder in `/var/lib/surfaces`.
- **RFC 0017:** a change of language or region restarts the device;
  built-in native apps' catalogues live in `/usr/share/locale` under the
  app's id.
- **RFC 0020:** a service may post a notification on an app's behalf, as
  the app's.
- **RFC 0024:** the apps' memory budget as two values, internal memory
  for the running native apps together and external memory for each.
- **RFC 0028:** wake timers registered by apps, with the notification
  Time posts for them, when Time's timers are designed.

## Rules it follows

- **Layers** ([RFC 0004](0004-layers.md)): apps are above the system UI,
  so an app calls it and is told by events; the system UI never calls an
  app.
- **One owner:** the system UI owns the screen and every input; apps
  describe, the system UI draws (RFC 0019).
- **Trust** (RFC 0002): native apps are trusted; the system UI's checks
  catch their mistakes, and hold WebAssembly apps.
- **It bends RFC 0019,** and RFC 0017 with it, in three places: the stack of screens becomes a
  stack of apps, each keeping its own screens; command letters are the
  app's, not the system UI's; and a change of language or region
  restarts the device instead of taking effect at once.

## Alternatives

- **The system UI also a client of each app.** Replies from the app, but
  a second connection per app, and the system UI calling up the layers.
- **Shared memory for screens.** Pointless for messages of a few hundred
  bytes; kept for pictures and the canvas.
- **The system UI keeping each app's screens.** Back is immediate,
  without asking the app, but the system UI holds every screen's
  description and the app is not asked about back.
- **One screen per app, no stack.** Simplest, but every back is the app's
  work anyway, and the shape of a screen's history is lost.
- **Back pops at once, unless a screen is marked to ask first.** One
  redraw per back without a round trip, but two rules for back.
- **The system UI assigning command letters** from the app's suggestions.
  Never a clash, but the same command's letter could differ between
  menus.
- **One app at a time,** leaving an app stops it: the least memory, but
  every return pays a start. **Keeping every app** that was started: no
  room for it.
- **Apps posting their own notifications** from the background: the
  first apps would all stay loaded.
- **The whole template at every change; or only changes, from the
  first.** Simpler for apps, or less sent; whole screens then changes
  takes both, and the system UI draws only what differs anyway.
- **The system UI translating apps' strings.** Possible, but strings
  with substitutions must be translated before they are filled in.
- **An event when the language changes, or restarting only the apps.**
  No restart of the device, but every program, or every app, must handle
  the change correctly, for a change made rarely.
- **Bringing the chain back after the system UI restarts.** The owner
  keeps their place, but Home is simpler and a restart is rare.
- **The runtime decoding and re-encoding apps' messages.** Twice the
  work, and fields it does not know are lost.
- **Widgets given through the connection, kept by the system UI.** No
  shared files, but the app or a service must reach the system UI for
  each change, and the system UI must save what it was given.

## Costs and risks

- **Memory:** a native app's task, its descriptors and about 1 KB each
  way per local socket (4 to 8 KB of internal memory) cannot go to
  external memory. On the current e-paper device, about 44 KB of internal
  memory is free in normal use and 16 KB with Wi-Fi and mobile data up
  (measured 2026-09-26 to 2026-09-28): two apps at once is the most to
  plan for. The budget's
  values are measured before they are set.
- **Back crosses programs** every time; a slow app costs a second partial
  refresh, and a hung one shows "Waiting for *app*…".
- **Returning to a stopped app** pays a start: a task, a connection, the
  first screen; to be measured.
- **Native apps are trusted:** the checks do not stop one written to
  misbehave.
- **Widgets write to internal flash:** each change of a record is a
  file replaced; writers keep to changes of what is shown, and a time
  is formatted by the system UI, not rewritten.
- **The manager's two additions** live in the fork, as every other nxinit
  change does ([RFC 0003](0003-repositories.md)).

## Open questions

- **The memory budget's values** for native apps: measured on the device,
  then decided.
