# RFC 0030: Telephony

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-30
- **Last changed:** 2026-10-30
- **Supersedes / superseded by:** —

## Summary

Telephony owns the cellular modem. It powers the modem up and down, keeps
it registered, and tells the rest of the system the network's state and
time. It carries SMS for Messages, makes and takes voice calls, and gives
the device mobile data over PPP. It runs alone in the telephony program.
This RFC covers all of it, and the order it is built in.

## Problem

- **The service has a row, and nothing more.**
  [RFC 0007](0007-services.md) names Telephony, the owner of "the cellular
  modem and its control lines: calls, SMS, network state, mobile data". It
  gives Telephony a program of its own, since it is the most exposed
  service and calls must survive other faults. Nothing says:
  - what Telephony offers the rest of the system;
  - how it shares the modem's one serial line between commands and data;
  - what it does when the SIM is locked, the modem resets itself, or
    the network is GSM only.
- **Others are waiting on it.**
  - [RFC 0028](0028-time.md) takes its best time from the modem's
    network, on the `network_time` topic, which nothing publishes yet.
    Without a time source, Time cannot learn the sleep clock's rate.
  - Messages (RFC 0029, proposed alongside) expects Telephony to be its
    `sms` transport.
- **The previous system had all of it** ([current/](../current/)): modemd,
  about 7000 lines in the old code. It had:
  - SMS in PDU mode, with its own codec and long messages both ways;
  - calls, and the call settings;
  - mobile data over a multiplexed line;
  - the network's time;
  - diagnostics.

  This RFC keeps what worked, and what was learnt on the T-Deck Max, as
  its [hardware page](../hardware/t-deck-max.md) and board documentation
  record it.

## Proposal

### What others see

- **The state,** on the `telephony` topic, and from a `Status` call:
  - the modem: off, starting, on, failed;
  - the SIM: absent, needs its PIN, needs its PUK, ready, blocked,
    failed;
  - the registration: none, searching, home, roaming, denied;
  - the operator's name;
  - the technology: LTE or GSM;
  - the signal, in bars and in dBm;
  - mobile data: off, connecting, on, failed.

  The topic carries no phone numbers: any native code reads topics
  ([RFC 0005](0005-communication.md)).
- **The network's time,** on `network_time` as RFC 0028 describes. Each
  reading carries when it was taken, by the monotonic clock, and its
  uncertainty.
- **Calls,** on the `call` topic: idle, incoming, dialing, ringing at the
  other end, active, ended. Methods:
  - `Dial`, `Answer`, `HangUp`;
  - `SendTones` (DTMF), and muting the microphone;
  - `CallInfo`, which gives the other's number to a caller that may read
    it: the call under way, or else the last one, until the next.
- **SMS** go through Messages (RFC 0029), never past it. Telephony is the
  `sms` transport (below).
- **The PIN.**
  - A SIM that needs its PIN is reported, and Telephony waits. `EnterPin`
    hands the PIN over; `EnterPuk` hands the PUK with a new PIN.
  - Both fields are marked sensitive ([RFC 0026](0026-log.md)), so no log
    shows them, and nothing stores them.
  - The tries left are reported where the modem tells them (SIMCom's
    `AT+SPIC`, to check on the A7682E). A PUK used up is reported as
    blocked.
  - Keeping the PIN, so that the SIM unlocks after a restart, waits for
    Settings' secrets ([RFC 0025](0025-settings.md)), which wait for
    Security ([RFC 0014](0014-security.md)).
- **Who may.** In this RFC, the system UI and the developer's shell may
  do everything. Apps come with the runtime's permissions
  ([RFC 0008](0008-runtime.md)): placing calls, reading the call state,
  reading a caller's number. Sending SMS is Messages' to allow.
- **The system interface** ([RFC 0004](0004-layers.md)) is for the
  system UI and the developer's shell only. It has:
  - the serving cell's details;
  - a scan of the networks in reach;
  - one AT command passed through, for diagnosis.

  The public interface has no AT command.
- **Settings,** owned by Telephony:
  - `enabled`: the modem on or off; off shuts it down and cuts its rail;
  - mobile data on or off, and its access point name;
  - GSM allowed, off by default on the T-Deck Max, by its target's
    defaults ([RFC 0025](0025-settings.md); below);
  - whether to send the caller's number, and to show the other's;
  - `log.level`, as every owner has (RFC 0026).

- **Airplane mode** belongs to the whole device, not to Telephony, and
  no service may depend on the system UI (RFC 0004). So the system
  program keeps it and publishes it on a topic, as Board publishes the
  privacy switches (RFC 0024); the system UI only switches it. Telephony
  obeys it with `AT+CFUN=4`: the modem stays powered, off the air, and
  comes back quickly. A call under way ends; queued SMS wait; the SIM
  and its PIN are still reachable. Which module of the system program
  keeps it is an open question.

### The modem's line

The modem is behind one UART, run at 115200 baud, and both commands and
mobile data need it. Telephony multiplexes it with 3GPP 27.010
(`AT+CMUX`) from the moment the modem answers:

| Channel | Carries |
|---|---|
| 1 | AT commands, and what the modem reports unasked: calls, SMS, registration, the network's time |
| 2 | mobile data: PPP, through a pseudo-terminal |

- **One code path from the start,** so turning data on changes nothing
  for the commands. The previous system multiplexed only while data was
  on, and its troubles came from the multiplexer's ways: a fresh channel
  refusing its first command, channel 1 silent while channel 2 dials,
  the modem leaving the multiplexer on its own. Starting it at bring-up
  meets them once, where they are handled. Data reached 70–80 kbit/s
  through it, near the line's limit of about 92.
- **What the modem reports unasked arrives between answers** on channel
  1, and Telephony's reader sorts the two, as the previous system's did.
  A third channel only for those reports would be cleaner. Whether the
  A7682E can send them there (SIMCom's `AT+CATR`) is to check.
- **The multiplexer is Telephony's own,** on its event loop: it reads
  the UART's frames as events, gives channel 1 to the AT engine
  directly, and channel 2 to `pppd` through a pseudo-terminal. It is
  written after the previous system's (824 lines), which handled:
  - the control channel's messages (`MSC` for the lines, flow control);
  - a modem that restarted, from its unframed `RDY` line;
  - closing a multiplexer a stopped program left open;
  - frames written whole to the UART, with no bytes lost.

  On the loop, nothing may block. The UART and the pseudo-terminal's
  master side are not blocking: frames wait in a bounded queue and go
  out as the UART takes them, and what `pppd` writes is read in batches
  and framed whole. The previous system learnt that a frame for every
  byte `pppd` writes cuts uploads from 80 to 10 kbit/s.
- **NuttX's own multiplexer,** `netutils/cmux` in nuttx-apps, was
  weighed. As it stands it cannot be used here:
  - It cannot be stopped or closed.
  - It polls the UART every 100 µs, which RFC 0004 forbids, and which
    would keep the processor from light-sleeping.
  - It has no control-channel messages.
  - It runs on a 3 KB stack.

  Making it fit would be a rewrite in the fork (Alternatives).

### Starting, stopping and recovering

- **At the device's start the board leaves the modem's rail off,**
  unless it finds the modem already running after a reset of the chip.
  It does that because cutting the rail of a running modem can damage
  the modem's flash (SIMCom). So Telephony first asks whether the modem
  is on.
- **A modem found on** may still be in the multiplexer of a telephony
  program that stopped. Telephony closes the multiplexer blindly, then
  speaks plain AT.
- **Powering up:** the rail on, then the power key for the time SIMCom
  gives, then waiting for the modem to answer.
- **Powering down, always in order:**
  1. `AT+CPOF`, or else the power key held for 3 s;
  2. wait until the modem stops answering, about 6 s;
  3. only then cut the rail.

  `nxinit` gives a stopping program 3 s before it kills it, too short
  for this and `pppd` together. The telephony program stops gently, as
  the system program does, and the time becomes 10 s: one firmware
  option, a ceiling for every program, which most never reach.

  This happens when the `enabled` setting goes off, when the program is
  asked to stop, and before the device powers off at a low battery: Power
  then stops the other programs before its own (a change to Power,
  [RFC 0007](0007-services.md)), and the manager gives Telephony the time
  to finish.
- **A modem that stops answering** is powered down in order and up
  again, with a growing wait. At most three times in an hour; then it
  stays off, reported as failed, until the owner turns it on again.
- **A modem that restarts itself,** as it does on GSM (below), sends an
  unframed `RDY` instead of frames. Telephony then:
  - stops PPP and closes its pseudo-terminal;
  - starts the multiplexer again;
  - sends its first command twice, since a fresh channel refuses the
    first one;
  - rebuilds its state from the modem (RFC 0004).
- **While channel 2 dials, channel 1 does not answer** for some seconds.
  Commands wait rather than fail.
- **Every setting the modem keeps is set again at each start,** since the
  modem keeps some across power cycles and a value left from before
  cannot be trusted:
  - network mode and bands (`AT+CNMP`, `AT+CNBP`);
  - combined attach (`AT+CEMODE=2`), without which SMS fail on networks
    without IMS;
  - PDU mode and new-SMS reports (`AT+CMGF=0`, `AT+CNMI=2,1`);
  - registration reports (`AT+CREG=2`, `AT+CEREG=2`);
  - the network's time (`AT+CTZU=1`, and its reports);
  - the ring indicator for SMS too (`AT+CFGRI=1`);
  - the caller's number shown and sent (`AT+CLIP`, `AT+CLIR`), once the
    SIM's phone book is read (`PB DONE`).
- **A SIM is read at each start of the modem.** The SIM and the memory
  card share one holder, and a SIM changed is seen only at the modem's
  next start. With no SIM the modem is off already (next point), so the
  owner turns the `enabled` setting off and on.
- **No SIM: the modem goes off.** With no SIM it can do nothing useful
  here (next point), and it would cost 35–60 mA. The state says why.
- **Emergency calls are not promised on the T-Deck Max.** Without a SIM,
  or with a locked one, there is no voice over LTE. Such a call falls
  back to GSM, where this modem switches itself off. With a locked SIM,
  Telephony passes the call to the modem, and the modem decides; with
  no SIM, the modem is off.

### Power and its lines

- **Its control lines come from Board's description**
  ([RFC 0024](0024-board.md)). A part has one device file, so the
  modem's part, besides its UART, gains four:
  - the rail;
  - the power key;
  - the ring indicator;
  - DTR.

  Power already reads its battery's files from Board's description
  (pnut-os #39; RFC 0024 as amended by design #52). Until the schema
  changes, the four lines are firmware options.
- **The ring indicator wakes the processor** for a call or an SMS, so the
  device light-sleeps with the modem registered. The rail does not hold
  the processor awake; the UART keeps it awake while in use. Measured
  without a SIM (2026-10-17): 60 mA with the processor awake, 35 mA
  asleep. The previous system found no saving with the modem registered,
  about 48 mA either way.
- **The modem's own sleep stays off** (`AT+CSCLK`): with it on and the
  processor asleep, the modem missed commands (2026-09-26). Its cause is
  in the backlog.

### Networks: LTE, and GSM with care

On GSM, the T-Deck Max's modem switches itself off, most likely because
its supply sags under GSM's transmit bursts. So:

- **By default the modem uses LTE only,** on the bands Board's
  description gives. The previous system had a bands setting; bands
  are the device's fact now.
- **Calls need voice over LTE,** which the operator must allow for the
  line.
- **GSM is a setting.** Where only GSM reaches, the owner may allow it.
  - If the modem has registered on no network at all two minutes after
    GSM was allowed, Telephony goes back to LTE only.
  - Restarts of the modem on GSM count towards the three an hour above.
    Past them, GSM is turned off again.
  - Either way Telephony changes a setting the owner set, so the state
    and the log say why.
- **A device whose modem copes with GSM** allows it by default, through
  its target's defaults (RFC 0025).

### The network's time

- **The readings Time can trust** come from the network's own message
  (`AT+CTZU=1` and its report), stamped by the monotonic clock when the
  line arrives. Summer time comes from that message.
- **Hourly readings.** RFC 0028 wants a reading at most hourly while a
  source is cheap. While the modem is registered, and only once the
  network has sent its time since the modem started, Telephony reads the
  modem's clock (`AT+CCLK?`) at the first wake after the hour. Not every
  network sends the time; without it, the modem's clock is whatever it
  started with, and there is no hourly reading.
  - That clock runs on the modem's own crystal since the network last
    set it, and gives whole seconds and an offset in quarter hours.
  - Such a reading is marked on the topic as the modem's clock, with an
    uncertainty of a second plus its age since the network's message at
    100 ppm, a bound assumed for the crystal, not measured.
  - Time must not learn the sleep clock's rate from such readings.
    RFC 0028 is amended to say so, and its topic gains each reading's
    kind.
- **The uncertainty of a network message's reading** is the second it
  is truncated to, and the time the line takes to arrive: a second in
  all.

### SMS

- **PDU mode,** with Telephony's own codec:
  - GSM 7-bit with its extension table, and UCS-2;
  - the headers that join a long message's parts.
- **In the modem's memory,** not the SIM's, which holds 25: `AT+CPMS`
  with `ME`. The capacity is to check. A long message whose parts are
  still coming holds its slots until it is whole.
- **Arriving:**
  - The modem stores each SMS and says where.
  - Telephony reads it, and delivers it to Messages once all parts are
    in. A message whose parts are still missing after a day is
    delivered with a mark where the gap is.
  - Each delivery carries the key RFC 0029 defines for an SMS, and
    Messages stores a message once for each key.
  - Telephony deletes the SMS from the modem only after Messages has
    stored it. After a reset between the two, the delivery comes again,
    and the key keeps it from being stored twice.
  - At start, Telephony reads what the modem kept. Until Messages is
    ready, the SMS stay in the modem.
  - When the modem's memory is full, the network keeps the messages and
    tries again later. Telephony delivers promptly, so it does not fill.
- **Leaving:**
  - Telephony takes Messages' queue, one message at a time.
  - It splits the message into parts and sends each.
  - It reports the message as sent, or as failed with the network's
    reason.
- **Delivery reports** come later (step 5). They need the network's
  reference kept with the message across restarts, which Messages would
  hold.

### Calls

- **One call at a time.** Call waiting, holding and joining calls come
  later.
- **The sound.** The modem has its own microphone. Its speaker output
  and the codec's share the speaker, chosen by a switch in front of the
  amplifier. That switch and the amplifier are Audio's
  ([RFC 0007](0007-services.md)), and Audio does not exist yet. For a
  call, Telephony asks Audio to route the modem, and Audio gives the
  line back after.
- **An incoming call** is announced on the `call` topic. The system UI
  rings, through Audio and Haptics once they exist.
- **The call history** is Contacts' (RFC 0007), and a missed call is a
  notification ([RFC 0020](0020-notifications.md)). Telephony gives
  both what they need on the `call` topic and through `CallInfo`. Until
  they exist, nothing keeps them.

### Mobile data

- **PPP runs on channel 2** while the data setting is on and the network
  is there. It stops when the modem goes off, and while a call needs the
  line, if the network cannot carry both.
- **`pppd` is NuttX's,** in nuttx-apps. The previous system had to patch
  it:
  - for the name servers;
  - for the A7682E's full HDLC address and control headers;
  - to stop it.

  The fork has none of these yet; they are redone there, written to go
  upstream. How it is used matters too: a TUN MTU of 1440, since the
  path drops larger packets, and a blocking pseudo-terminal.
- **Choosing the link and the name servers is Network's**
  ([RFC 0015](0015-network.md)). Until Network exists, Telephony gives
  the link a host-only netmask, so that Wi-Fi keeps the traffic while it
  is up, as the previous system did. That leaves a known hole: when
  Wi-Fi comes up after mobile data, the PPP link stays the default
  route.

### How the program runs

- **One event loop,** with commands queued and their answers handled as
  events. A command's long wait, up to a minute for an SMS and three for
  a network scan, is a timer, not a blocked thread.
- **One thread besides, `pppd`'s,** which is NuttX's code. RFC 0004
  asks for one loop and workers; the multiplexer runs on the loop.
- **It starts after the system program,** for Board's description, its
  settings and Service states.

### Not in this RFC

USSD, cell broadcast, the SIM's own menus (SIM toolkit), several SIMs or
eSIMs, satellite messaging, MMS, call waiting, holding and joining calls,
and choosing a network by hand. Each comes later, in an RFC of its own
or as a change to this one.

### The order of the work

1. **The modem, and the network:**
   - power, the lines and the multiplexer;
   - the state, the topic, and the PIN;
   - the network's time, and the change to Power's power-off.

   Time gets its first real source here.
2. **SMS,** once Messages is built.
3. **Calls,** with the routing to the speaker (see Open questions).
4. **Mobile data,** with `pppd`'s fixes in the fork.
5. **Call settings, delivery reports and diagnostics:**
   - divert and call waiting;
   - delivery reports;
   - the cell's details, and a network scan.

Each step is a pull request of its own, tested on the phone with the test
number.

## Rules it follows

- **One owner.** Telephony alone opens the modem's UART and lines
  (RFC 0004). Others reach the modem through its interface.
- **No raw access in public.** An AT command passes through only in the
  system interface, for diagnosis.
- **Optional.** A device without a modem runs no telephony program, and
  nothing else breaks: Messages has no `sms` transport, and Time no
  `network_time`.
- **Topics carry nothing private:** no numbers, no texts.
- **Secrets are not kept** until Settings can keep them: the PIN is
  asked for at each start.
- **It runs alone** (RFC 0007), so that a call keeps working when
  another program fails.
- **Where it bends a rule:**
  - `pppd` brings its own thread.
  - `pppd` may run inside the telephony program, though RFC 0007 lists
    it among the programs the manager starts (Open questions).

## Alternatives

- **Plain AT until data is on,** then the multiplexer, as the previous
  system did. That meant two code paths, and a mode switch while the
  modem was busy.
- **NuttX's multiplexer, rewritten in the fork:** a way to stop it,
  events instead of polling, the control channel, and a bigger stack.
  Most of it would change, upstream would gain from it, and it would
  take longer than Telephony's own.
- **A faster UART** (`AT+IPR`). It is untried, and the board has no flow
  control wired.
- **The modem's own TCP/IP stack** instead of PPP. Apps would then need
  a socket layer of Telephony's, not the kernel's.
- **The modem's own USB,** which carries AT and data on separate
  interfaces. On the T-Deck Max it is only on test pads.
- **GSM by default.** It reaches more places, but on this device it
  stops the modem.
- **SMS on the SIM,** listed by Telephony, with no store. The owner
  chose Messages first.
- **The modem on without a SIM,** for emergency calls that it most
  likely cannot complete here.

## Costs and risks

- **RAM.** One program with its loop, the multiplexer's buffers, and
  `pppd`. The previous system had about 16 KB of the kernel's heap left
  with Wi-Fi and mobile data up together, the tightest case.
- **Power.** The device draws about 48 mA with the modem on and idle on
  LTE, against 28 mA awake without it (hardware page). The `enabled`
  setting, and the rail off with no SIM, are the savings.
- **Code.** The multiplexer, about 800 lines, is Telephony's own.
- **Risks:**
  - the modem's quirks, some known only from the previous system;
  - GSM stopping the modem;
  - `pppd`'s fixes, needed again in the fork.
- **To be measured:**
  - the time from power-up to registered;
  - the current with the processor light-sleeping and the modem
    registered;
  - the current in airplane mode, the modem off the air;
  - the multiplexer's throughput, on this firmware: a frame for every
    byte `pppd` writes would cut it to an eighth;
  - the modem's SMS memory.

## Open questions

- **Calls before Audio:** build Audio's routing first, as proposed, or
  let Telephony switch the amplifier and its input until Audio exists.
- **`pppd`:** a thread of the telephony program, which hands it the
  channel's pseudo-terminal, as the previous system did; or a program
  the manager starts with that pseudo-terminal's name, as RFC 0007
  says.
- **Mobile data before Network:** as proposed, with the host-only
  netmask and its hole; or data waits for Network.
- **The modem's lines in Board's description:** add them to the schema
  now, as proposed, or keep them firmware options for now.
- **The multiplexer:** Telephony's own on its loop, as proposed; or
  NuttX's, rewritten in the fork to go upstream.
- **Airplane mode's keeper** in the system program: Board, beside the
  privacy switches, or Power.
