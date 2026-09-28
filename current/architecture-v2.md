> **Status:** the current system, as of 2026-09-28. Copied from the
> development workspace, where it was kept next to the code; paths such as
> `pnut-os/docs/`, `BACKLOG.md` and `README.md` refer to that workspace and
> its repositories, which are not on GitHub yet. This is the starting point
> for the rewrite: RFCs change it, and it is not edited to match them.

# T-Deck Max system architecture (v2)

How the software on the T-Deck Max is organised above the drivers.
- **v1** was agreed on 2026-09-22 and built as five daemons and a UI shell.
- **v2** (2026-09-23/24) turns that into a platform: daemons as device APIs, apps as WebAssembly packages, a registry, permissions, notifications, i18n, storage and logging.

This document is the overview and the record of decisions. The reference is in `pnut-os/docs/`:

| Document | Covers |
|---|---|
| `services.md` | The service contract: bus, interfaces, naming, capabilities, permissions, wire format, paths |
| `apps.md` | Packages, manifest, the WASM runtime and its host API, lifecycle, MeshCore walked through |
| `ui.md` | The UI component catalogue (from the design) and how apps describe screens |
| `notifications.md` | notifyd, the four notification levels and their rules |
| `storage.md` | SQLite and dbd |
| `i18n.md` | Strings, languages, formats, fonts |
| `logging.md` | Levels, logd, crash reports, adb |
| `theming.md` | Themes (already built) |

Open hardware items are in `BACKLOG.md`. Driver details are in `README.md` and the board page in `nuttx/Documentation/`.

**Status (2026-09-24):** rounds 1 to 6 are built, and most of round 7. Round 1: the service skeleton, the IDL, svcd, `pnut_log` and live levels, the 64 KB RAM log kept over a reset, logd with crash reports, adb over the network. Round 2: capabilities, registered settings namespaces with schemas and reverse-DNS keys, Settings built from the schemas. Round 3: streams, and msgd routing messages through transports provided by modemd and meshd. Round 4: lorad, the one holder of the LoRa radio. Round 5: notifyd, and the shell in English and Polish from gettext catalogues. Round 6: SQLite, with msgd's messages in `messages.db` and dbd for apps' databases. Round 7: the catalogue's templates in the shell, and `org.pnut.shell`, through which apps show screens and widgets. The rest of v2 is a design; section *Implementation rounds* lists the order and marks what is done.

## Goal

A pocket phone that can be demonstrated and then grown by enthusiasts:
- SMS and calls over 4G, and text messaging over a LoRa mesh.
- The mesh is **MeshCore or Meshtastic**, and later others (a MeshCore fork, Ripple).
- Everything beyond the core is an installable **app**.
- The screen follows the user's design: the Claude Design canvas "E-ink Softkey Phone", its *Prototype 240×320* and *Components* pages.

## Principles

1. **Daemons own the hardware and turn it into APIs.**
   - Each device (modem UART, LoRa radio, GNSS, battery, audio) is opened by exactly one daemon.
   - It offers a typed **public API** (`send an SMS`, `transmit a LoRa frame`) and a **private API** for native system programs only (raw AT commands, radio registers). Nothing above a daemon touches a peripheral.
2. **Apps are sandboxed packages.** MeshCore, Meshtastic and future protocols and tools are WebAssembly apps. They see only public APIs, and only those their **permissions** allow.
3. **The device describes itself.** Daemons register **capabilities** (`radio.lora`, `telephony.sms`…), so an app that needs LoRa can be refused on a device without it.
4. **Daemons check who is asking.** The kernel tells a daemon which task is calling, and appd stamps which app is calling. The daemon acts only within that caller's rules and grants.
5. **The UI is a client.** shelld draws what services publish and sends them commands. A fault in the modem or an app must not take the screen down, and the UI can restart without dropping a call.
6. **Designed for e-paper.**
   - Nothing animates.
   - Each decision costs one partial refresh (about 0.7 s).
   - Lists page, never scroll.
   - Typing is batched.
7. **No hard-coded English.** Every user-visible string comes from a catalogue.
8. **For enthusiasts.** NSH will be fully exposed on the device, adb works, and the system is documented well enough to extend.

## Layers

```
 ┌───────────────────────────── apps (WASM packages) ─────────────────────────────┐
 │ org.meshcore.companion   org.meshtastic.app   … third-party                     │
 │   background service + declarative screens + widgets + settings schema          │
 └──────────▲──────────────────────── public APIs only ─────────────────▲──────────┘
            │                    (through appd, identity stamped)        │ screens,
 ┌──────────┴───────── system UI: shelld (native) ──────────────────────┴─ widgets ─┐
 │ Lock, Home, Menu, Messages (unified inbox), Phone, Settings, notifications       │
 └──────────▲─────────────── public + private APIs ─────────────────────────────────┘
 ┌──────────┴────────────────────── system daemons (native) ───────────────────────┐
 │ svcd (registry)  pkgd (packages, permissions)  appd (WASM runtime)  cfgd        │
 │ modemd (telephony)  lorad (LoRa radio)  msgd (messages)  sysd (power, time)     │
 │ notifyd  dbd (SQLite)  logd  wifid   ·   adbd (NuttX)   ·   later: btd, gnssd   │
 └──────────▲──────────────────────────────────────────────────────────────────────┘
          drivers: /dev/ttyS1, /dev/lora0, /dev/fb0, /dev/kbd0-2, batt0, …
```

### Components

| Component | Service name | Owns / does | State |
|---|---|---|---|
| **svcd** | `org.pnut.svc` | Registry: names, interfaces, capabilities, readiness | built (round 1) |
| **pkgd** | `org.pnut.pkg` | Packages: install, manifests, settings and transport registration, permission grants | new |
| **appd** | `org.pnut.app` | WASM runtime (WAMR): one task per running app, the host API | new |
| **cfgd** | `org.pnut.cfg` | Settings: registered namespaces, schemas | built (round 2) |
| **sysd** | `org.pnut.sys` | Battery, charger, lights, clock | exists |
| **modemd** | `org.pnut.modem` | A7682E: typed telephony API (SMS, calls, status); AT passthrough private; provides the SMS transport | exists; API tightened |
| **msgd** | `org.pnut.msg` | The message store (unified inbox) and routing to transports | exists; transports added; on SQLite (`messages.db`) |
| **lorad** | `org.pnut.lora` | SX1262: one owner at a time, region plan, airtime | split out of today's meshd |
| **notifyd** | `org.pnut.notify` | Notifications: levels, channels, focus modes, bursts | new |
| **dbd** | `org.pnut.db` | SQLite databases for apps | built (round 6) |
| **logd** | `org.pnut.log` | Persistent logs, crash reports | built (round 1) |
| **wifid** | `org.pnut.wifi` | The Wi-Fi station: joins the network in the settings, DHCP with renewal, rejoins; scans | built (2026-09-26) |
| **shelld** | `org.pnut.shell` | The system UI; renders app screens and widgets; permission prompts | today's `shell` |
| **adbd** | (NuttX) | Remote shell, files, logcat | built over the network (round 1), with key checks; USB later |
| `pnut` | — | The command-line tool, in NSH | exists; has `pnut ps`, `caps`, `info`, `log`, `adb`; grows (`pnut app`, `pnut db`) |

Task names stay short (`modemd`); service names are reverse-DNS.

**Rule for what is a daemon and what is an app:**
- **Daemon:** anything that owns a device, or must run for the phone to work (cellular, storage, the radio driver, the UI).
- **App:** anything that is a protocol or a feature on top of a device, which users might add, swap or remove.

## The bus

Details are in `services.md`.

- **Registry plus direct calls,** like Android's Binder, with naming and introspection like D-Bus. svcd keeps who is who; callers look a service up once and talk to it directly over its local socket.
- **Methods** are request and reply over a local stream socket per service (`/var/run/pnut/<service>`). The kernel reports the calling task (`SO_PEERCRED`).
- **Signals** (broadcast state and events) are uORB topics.
- **Streams** are a connection kept open for events meant for one party: radio frames for the app holding the radio, outgoing messages for one transport.
- **Interfaces** are described in a small IDL. Each method is `public` or `private` and names the permission it needs. A generator writes the C client stubs, the server dispatch with the permission check, the WASM host bindings and the reference docs.
- **Wire format:** versioned, fixed-size, little-endian structs.
- **Readiness:** every service publishes its state (starting, ready, degraded, failed) on one topic, `org.pnut.svc`.

## Apps

Details are in `apps.md`.

A **package** is a directory: `manifest`, `app.wasm` (or ahead-of-time compiled `app.aot`), strings, icons.

The manifest declares:
- `requires` and `uses` (capabilities);
- `permissions`;
- components: a background **service**, **screens**, **widgets**, a messaging **transport**, notification **channels**;
- the **settings schema**.

On the device:
- **pkgd** installs it and registers its settings, transport and channels.
- **appd** runs it in WAMR, one task per app, event-driven.
- **shelld** asks for permissions on first run.

**Screens** are declarative. An app sends a screen built from the system's templates (list, detail, form, conversation…); shelld draws it in the theme and handles keys, touch, paging and Find; the app gets events back. See `ui.md`. Apps never draw pixels, except an image block the shell dithers.

### Messaging: the unified inbox

msgd is a **message store with routing**, like Android's SMS provider. It hosts no app code.
- **Incoming:** an app stores what it received under its own transport (`org.meshcore.companion`).
- **Outgoing:** when you reply in Messages, msgd queues the message on that transport's outgoing stream; the app sends it and reports sent, delivered or failed.
- **SMS** is the same shape, with modemd as the SMS transport.

### LoRa

- **lorad** is the radio daemon: it is what makes the capability `radio.lora` exist.
- **Single owner:** the app you choose (Settings › LoRa › Radio used by) gets the radio with its parameters, checked against the region plan and its permission limits. It alone receives frames and may transmit.
- **Why one owner:** MeshCore (869.618 MHz, 62.5 kHz, SF8) and Meshtastic (869.525 MHz, 250 kHz, SF11, another sync word) share no configuration, and the SX1262 listens on one at a time.
- **The protocols are apps:** MeshCore and Meshtastic decrypt, route, keep node lists, provide a messaging transport, and show their own screens and widgets.
- **Built (round 4):** lorad (`org.pnut.lora`, `idl/LoRa1.pidl`): acquire opens a stream of received frames for the one holder; transmit; release, or the holder's stream closing. EU868 sub-bands with power limits and duty cycles (airtime counted over the last hour). Until appd, meshd hosts both protocols natively and holds the radio for the network `org.pnut.mesh.provider` names.

## Settings, capabilities, permissions

- **Settings** keys are reverse-DNS and owned by their service or app: `org.pnut.modem.pin`, `org.meshcore.companion.name`. The schema comes from the IDL or the manifest, and Settings is built from the schemas, so an app's settings appear without shell changes.
- **Capabilities** are registered by the daemons with their parameters (`radio.lora`: EU868, 22 dBm max). Apps are checked against them at install.
- **Permissions** are granular:
  - `lora.receive`
  - `lora.transmit` with power and duty limits
  - `msg.transport`
  - `msg.read`
  - `telephony.sms.send`
  - `contacts.read`
  - `location.fine`
  - `notify`
  - …

  They are granted item by item on first run and enforced by the daemon on every call. Native programs are the system tier, governed by `/etc/pnut/policy`.

## Notifications, storage, i18n, logging

- **notifyd** takes all notifications. The design's four levels are Silent, Passive, Banner and Interrupt (see `notifications.md`), with channels, focus modes, merged bursts and private public-text. Nothing interrupts while you type.
- **SQLite** (already in the NuttX apps tree) is used in-process by native daemons: msgd moves to `messages.db`. Apps get their own databases through **dbd**. Shared data stays behind typed APIs, never raw SQL.
- **i18n:** gettext catalogues (NuttX's libc has gettext), a language setting, localised manifests, locale formats, fonts per script.
- **Logging:** one API with levels and timestamps; a bigger RAM log; **logd** writes `/var/log`; crash reports after a panic.
- **adb** (NuttX's adbd) gives shell, file push and pull, and logcat, over USB or Wi-Fi.

## Paths

Linux-like; details and reasons are in `services.md`.

| Path | What | Medium |
|---|---|---|
| `/bin`, `/usr/bin` | native programs | firmware image |
| `/usr/lib/pnut/apps/<id>/` | built-in app packages | ROMFS |
| `/usr/share/pnut/` | themes, locale catalogues, fonts, icons | ROMFS |
| `/etc/` | `init.d/init.rc`, `pnut/policy`, `pnut/defaults` | ROMFS, read-only |
| `/var/lib/pnut/` | settings, package database, grants, app private data | 1 MB littlefs, mounted at `/var/lib` (at `/data` until 2026-09-25) |
| `/var/log/` | logs, crash reports | soft link to `/var/lib/log` |
| `/var/run/pnut/` | sockets | in memory |
| `/opt/pnut/<id>/` | installed apps, themes, language packs | the 7 MB flash region, its own littlefs |
| `/mnt/sd/pnut/` | the system's bulk data: `messages.db`, verbose logs (`log/`) | microSD |
| `/mnt/sd/pnut/data/<id>/` | an app's bulk data: its databases (dbd), maps | microSD |

## Privileges

NuttX on the ESP32-S3 cannot give Linux-style isolation between native programs, because separate address spaces need an MMU. In steps:

1. **Now (flat build):** daemons are the only openers of their devices, and they check the kernel-reported caller against their rules. Native programs could still ignore the rules.
2. **Apps (v2):** WASM apps can't touch memory or devices outside their sandbox; their only way out is appd's host API, which stamps their identity. So app permissions are real even in the flat build.
3. **Protected build** (`CONFIG_BUILD_PROTECTED`, supported on the ESP32-S3; see `esp32s3-devkit:knsh`): the chip's permission hardware separates the kernel and drivers from native programs.
4. **To investigate:** per-task user and group identity (`SCHED_USER_IDENTITY`) and permission bits on device nodes.

## Storage (flash)

| Range | Size | Use |
|---|---|---|
| `0x000000`–`0x3FFFFF` | 4 MB | Firmware slot A (image 2.10 MiB on 2026-09-25) |
| `0x400000`–`0x7FFFFF` | 4 MB | Firmware slot B, for over-the-air updates |
| `0x800000`–`0xEFFFFF` | 7 MB | `/opt/pnut`: installed apps (the mesh apps are about 20 KB each) |
| `0xF00000`–`0xFFFFFF` | 1 MB | `/var/lib`: settings, packages, grants, app data, logs |

- **The slots stay at 4 MB** (measured in round 8). The image is 2.10 MiB (2,198,704 bytes): SQLite 358 KB, WAMR 89 KB, the ROMFS with the two mesh apps about 60 KB. That doesn't fit a 2 MiB slot without cutting something. 3 MB slots would fit with room to grow and give `/opt` 9 MB, but `/opt` would have to move and be emptied; apps are about 20 KB, so 7 MB already holds hundreds. To revisit if `/opt` fills up.
- **`/var/lib` stays in the last megabyte** whatever boot scheme the slots end up with.
- **Messages, recordings and bulk data** go to the microSD card.

## Boot sequence (v2)

1. Drivers and board bring-up; `/var/lib` and `/opt` mounted; the microSD card mounted if present.
2. `pnut migrate` (init waits for it): moves what an older release left in `/var/lib`.
3. svcd, then logd and cfgd.
4. sysd, modemd, lorad, msgd, notifyd, dbd, pkgd.
5. appd starts the apps in `app.autostart`; meshd has it start the mesh network's app.
6. shelld shows the lock screen.

All are NuttX init services in the board's `init.rc`, restarted if they exit. Order comes from readiness on `org.pnut.svc`, not from sleeps.

## Decisions (2026-09-23/24)

**Architecture:**
- Daemons are device and service abstractions with public and private APIs; no raw passthrough in public APIs.
- MeshCore, Meshtastic and other protocols are standalone **WebAssembly apps**, not plugins inside a daemon.
- The bus is a registry plus direct calls, with D-Bus-style naming and introspection.
- Versioned, fixed, little-endian structs on the wire.

**Settings, capabilities, permissions:**
- Reverse-DNS names for services, apps, interfaces and settings.
- The settings schema lives in the manifest.
- Capabilities are registered per device; permissions are granular and granted on first run (the UX comes later).

**Messages and LoRa:**
- Unified inbox: msgd stores and routes; apps provide transports.
- LoRa has a single owner.

**UI:**
- Apps use declarative screens and widgets from a catalogue designed first. The catalogue is now on the *Components* page of the canvas.
- The shell becomes `shelld`.

**Platform services:**
- Notifications through notifyd.
- SQLite for storage, through dbd for apps.
- i18n from now on: no new hard-coded strings.
- Logging to `/var/log`; adb for remote access.

**Paths and tooling:**
- Linux-like paths; settings in `/var/lib` because `/etc` is the read-only ROMFS; `/var/log` as a soft link.
- `pnut` stays; NSH will be fully exposed.

**Far off:** package signing.

## Implementation rounds

Each round keeps the phone working, and is committed and documented step by step.

1. **Service skeleton.** **Done** (2026-09-24).
   - libpnut service skeleton, versioned header, common methods (`ping`, `introspect`, `info`, `loglevel`), `org.pnut.svc`, `pnut_log`.
   - The IDL and its generator.
   - Move the daemons onto them with no behaviour change.
   - adbd (over the network; USB later) and a bigger RAM log; then logd and crash reports.
   - Found and fixed on the way (NuttX): soft links into a mounted volume lost the rest of the path; a restarted server could not take its socket's address back; the ESP32-S3 tickless timer could lose its alarm and stop every timer of the system.
   - Moved to later rounds: `pnut/paths.h` (with the `/var/lib` move; done in round 8); USB adb (a Developer setting, with the user at hand).
2. **Registry and settings.** **Done** (2026-09-24). svcd with capabilities (registered by meshd, modemd, sysd and the shell); cfgd registration, schemas and reverse-DNS keys, with a one-time migration; Settings built from schemas.
3. **Messaging.** **Done** (2026-09-24). Streams; msgd transports and outgoing streams; modemd as the SMS transport (and meshd as the mesh one until lorad). Found on the way (NuttX): a thread killed inside a blocking call left poll registrations and file references behind; now it leaves the call first (cancellation points).
4. **lorad.** **Done** (2026-09-24). Split out of meshd; the protocols stay native behind the transport and radio APIs until appd exists.
5. **notifyd and i18n.** **Done** (2026-09-24). notifyd, with the shell's own counters moved onto it; gettext catalogues in a ROMFS image at `/usr/share/pnut`, the language setting, every shell string moved into the catalogues, and Polish.
6. **SQLite.** **Done** (2026-09-24). msgd on `messages.db`, with the first release's files imported; dbd for apps' databases (own files only, quotas). SQLite costs 377 KB of image (now 2.0 MB). Found on the way (NuttX): the FAT driver could lose a file whose name was exactly eleven lower-case characters.
7. **The UI catalogue in shelld.** **Mostly done** (2026-09-24). In the shell: the list, detail, form, choice, empty, busy, editor and confirm templates (the confirm with its action on the left softkey), the banner and the grouped notification centre. Apps' interface `org.pnut.shell` (`idl/Ui1.pidl`): screens from those templates, events back, widgets on Home, the lock screen and the status bar; Home's rows in the order you set. The wire form is in `ui.md`. Left: the conversation, image, date, time and incoming-call templates for apps, a screen to arrange Home, and the rename to shelld.
8. **Apps.** **Done** (2026-09-25).
   - Espressif's QEMU (`esp32s3-devkit:qemu_wamr`, `scripts/qemu.py`) and the WASI SDK 34 (`tools/wasi-sdk`) set up first; WAMR brought up in QEMU, then on the phone.
   - appd: WAMR's fast interpreter, a task per app named by its id, host API level 1 (`sdk/include/pnut_app.h`), the SDK (`sdk/build.sh`, `sdk/examples/hello`).
   - Permissions: apps held to public methods and their grants in libpnut's dispatch.
   - pkgd: manifest, install and remove, grants, apps' settings registered with cfgd (Settings › Apps, or where `settings_in` says), apps with a screen in the Menu.
   - `/opt` (7 MB littlefs), `/usr` (ROMFS with the built-in apps), `/var/lib` (the partition moved from `/data`, with `pnut migrate`), `pnut/paths.h`.
   - MeshCore, then Meshtastic, as apps: byte-identical packets to the native code; meshd is now only the network chooser.
   - Measured: WAMR 89 KB, apps 19–22 KB, 160 KB of PSRAM per running app, 1.4 ms to decode a Meshtastic packet. The slots stay at 4 MB (Storage).
   - Found and fixed on the way (NuttX): the ESP32-S3 HAL didn't build with GCC 15 (`ATOMIC_VAR_INIT`); procfs cut task names to 18 characters.
   - Left: AOT (needs Espressif's LLVM for `wamrc`), the permission screen (round 9), files in the host API, the apps on the air.
9. **Hardening and openness.** **Done, but for the protected build** (2026-09-25). The permission UX, NSH fully exposed. Package signing later.
   - **The protected build: parked.** QEMU can't run it (no World Controller), it needs the ESP-IDF bootloader and new linker-script sections, and it protects the kernel only from our own native code while apps are sandboxed already. Findings in BACKLOG.md; to come back to.
   - **NSH exposed: done** (2026-09-25). The Terminal in the Menu: NSH on a pseudo-terminal, drawn by the shell, Ctrl-C through the pty's SIGINT.
   - **The permission screen: done** (2026-09-25). Each permission with a switch, in the system's words with the app's reason below; asked before an installed app first opens from the Menu (Allow all, or one by one), and in Settings › Apps › the app › Permissions.

**After round 9** (2026-09-26 onwards), features rather than rounds:
- **Wi-Fi: done** (2026-09-26). wifid (`idl/Wifi1.pidl`): the network and its password are settings (the password a secret one, never logged), so the phone rejoins at boot; DHCP renewed at half the lease; a lost link or failed join tried again; Settings › Wi-Fi with a scan to pick from, status row and the status-bar mark; `pnut wifi [scan]`.
- **Long SMS: done, tested both ways on the phone** (2026-09-26). modemd moved to PDU mode with its own 23.040 codec (`modemd_pdu.c`: GSM 7-bit with the extension table, UCS-2, concatenation headers). Sending splits into parts. Receiving keeps the parts on the SIM until the set is complete, then files one message; after a day, a partial set is filed with a gap.
- **Mobile data: done** (2026-09-26). modemd multiplexes the modem's UART (27.010) while `modem.data` is on: AT on DLCI 1, PPP on DLCI 2 via a pty to NuttX's pppd (patched: DNS, full HDLC headers for the A7682E, a stop). `ppp0` gets a host netmask so Wi-Fi keeps the traffic while it's up. Wi-Fi's buffers were trimmed to keep the kernel heap at about 16 KB free with both.  TCP tested: 70 to 80 kbit/s each way (the 115200-baud UART caps it near 92), after a TUN MTU of 1440 (the path drops IP packets over 1480 bytes), a blocking pty master (O_NONBLOCK reached its pipes and lost bytes) and the mux sender at pppd's priority.
- **Private mesh messaging: done, not yet tried with a real node** (2026-09-26). Host API 2: Ed25519 and MeshCore's key exchange, natively in appd (`appd_ed25519.c`, after TweetNaCl, checked against RFC 8032 and Python's `cryptography`; sign 0.3 s, exchange 0.1 s on the phone). MeshCore: identity, signed adverts, contacts in the app's database, direct messages with ACKs (delivered marks), private channels from a setting.
- **UI gaps: done** (2026-09-26):
  - Contacts (kept by msgd; names in Messages, conversations and calls);
  - Polish letters from a held key;
  - the conversation, image, date and time templates for apps;
  - arranging Home's rows.
- **Power: sleeping with the modem on, off by default** (2026-09-26). RI wakes the chip (board `/dev/modem_sleep`, `/dev/awake`); modemd holds the chip awake per command and asks the modem after each RI. On battery it saves nothing measurable (about 48 mA either way, the modem idle on LTE). The modem's own sleep (`both`, `AT+CSCLK`, DTR) gets it to 18 to 20 mA, but the modem then misses most commands while the chip sleeps; the cause isn't found (BACKLOG), so the setting stays off.

- **Calls settings, as on a Nokia** (2026-09-26): Settings › Cellular › Calls. Send my caller ID and Show caller's number are the modem's settings (applied once the SIM is ready); Call waiting and Call divert are the network's, asked for through the new Telephony1 method `ss` and answered on the `pnut_ss` topic. With the Orange SIM only checking call waiting works: the rest sends the modem to GSM, where it switches itself off.
- **Robustness** (2026-09-26): the start-up guard (three crashes at start, then a safe start without the services; the board reports why it reset); apps started other than from the Menu are asked for their permissions (a notification); logd spares the flash (repeats counted, 120 lines a minute); apps hear their settings change (`PNUT_EV_SETTING`, which appd never sent); Meshtastic sends NodeInfo; cfgd moves the old mesh settings. Left: a thread killed in a file call keeps the file (a NuttX limitation; BACKLOG).
- **Wi-Fi and time** (2026-09-26): a password per network, the password editor at once for a new secured network, no wedged radio after a failed join; sysd sets the clock by NTP when the network can't.
- **App languages** (2026-09-27): the same app in C, AssemblyScript, JavaScript (QuickJS in the module) and Python (PocketPy in the module), measured on the phone (`pnut-os/sdk/examples/bench`). C and AssemblyScript are the ways to write apps; JavaScript and Python work but start in 5 to 7 s, take 3 to 4 MB and loop 100 to 470 times slower. appd now frees a killed app's runtime (it leaked 2.7 MB a time).
- **Calendar: designed** (2026-09-27), from the canvas's Calendar page (C1–C15). In order:
  1. time zones (`docs/time.md`);
  2. alarms kept by sysd (`org.pnut.alarm`), with reminders posted without starting the app;
  3. the Month and Week templates and agenda rows (`docs/ui.md`);
  4. the app in AssemblyScript, on the phone only (`docs/calendar.md`);
  5. CalDAV sync, with network access for apps and jobs;
  6. invitations.

**Testing:**
- The x86 simulator (`sim:pnut`) runs the daemons and the UI at other screen sizes.
- The phone is driven through the key FIFO and adb.
- QEMU runs Xtensa code (WAMR, AOT, the protected build) without flashing.
