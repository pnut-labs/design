# RFC 0034: Radio

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-11-13
- **Last changed:** 2026-11-13
- **Supersedes / superseded by:** —

## Summary

Radio owns the low-power packet radios: LoRa and other sub-GHz radios,
and later 802.15.4 and UWB. It gives one caller at a time a radio with
the configuration it asks for, sends that caller's packets and passes it
the packets heard, and keeps every send within the rules of the owner's
region. Mesh networks such as MeshCore and Meshtastic are apps that use
it. It runs in the **radios** program ([RFC 0007](0007-services.md)).

## Problem

RFC 0007 lists Radio among the services: it "sets a radio up and lends it
to whatever runs the protocol, such as a mesh app". No RFC says what it
offers, who gets the radio when two want it, or who keeps to the law: in
Europe the 868 MHz band is split into sub-bands, each with its own
transmit power and a share of air time per hour (1 %, 0.1 % or 10 %).

The first device has one radio, a Semtech SX1262 built for 863–870 MHz,
on an SPI bus it shares with the screen and the memory card. Its NuttX
driver sends and receives whole packets and is configured by ioctl; the
chip can also listen on a duty cycle, sleeping between short listens. A
radio hears one configuration at a time, so MeshCore and Meshtastic,
each on its own, cannot both listen at once.

## Proposal

### What others see

| Call | Does |
|---|---|
| **Open** | takes a radio, named as Board names it ([RFC 0024](0024-board.md)), with a configuration (below) and a way of listening; answered with a stream of the packets heard, until the caller closes it or goes |
| **Send** | a packet, on the open radio's configuration; answered once it is on the air, or refused (below) |
| **Configure** | changes the open radio's configuration, between packets |
| **Close** | gives the radio back |

A topic, **`radio`**, says for each radio whether it is open, by whom,
how it listens, and the air time used in the last hour, for the status
bar and the settings.

- **A configuration** is the radio's kind's own: for LoRa, frequency,
  bandwidth, spreading factor, coding rate, sync word, preamble length
  and transmit power. Each kind has its configuration message; an
  802.15.4 or UWB radio fits the same calls with its own.
- **A packet heard** comes with its signal strength, its signal-to-noise
  ratio and when it ended. The stream uses credit
  ([RFC 0004](0004-layers.md)): a caller that stops reading loses the
  oldest packets, never stalls the radio.

### One network at a time

A radio has one caller at a time. It goes to the app the owner chose for
it (a setting, below); with none chosen, to the first that asks. Another
caller is told it is busy, and the `radio` topic says who has it, so
an app can tell its user why it cannot listen.

### The region's rules

Radio checks every packet before it is sent:

- **Band:** the frequency and the whole bandwidth inside a sub-band of
  the region's band plan, and inside the band the radio is built for
  (Board's description).
- **Power:** at most the sub-band's limit, counted at the antenna, with
  the antenna's gain (Board's description gives each antenna's).
- **Air time:** each packet's time on the air, worked out from its
  length and configuration, counted per sub-band over the last hour. A
  packet over the sub-band's share is refused, with the time from which
  it would fit.

The band plans (EU868 first; others as devices for them come) are a
table in Radio, by name. The region is a setting: from the mobile
network's country when there is one, which Telephony is to publish
([RFC 0030](0030-telephony.md) does not yet), otherwise the plan that matches the radio's
band; the owner may set it.

### Listening

Each caller chooses how its radio listens:

- **Always** (the default): the radio receives all the time, and each
  packet wakes the processor. About 13 mA in light sleep on the first
  device, with MeshCore listening.
- **Duty cycle:** the radio sleeps between short listens, for less
  current; it hears only senders whose preamble lasts longer than its
  sleep and one listen, so a protocol chooses it when its senders do
  that.

Between packets the processor sleeps; the radio's interrupt wakes it.

### Mesh networks are apps

MeshCore, Meshtastic and their like are WebAssembly apps
([RFC 0008](0008-runtime.md)) that open the radio through the runtime.
To keep listening once closed, an app needs the permission to stay in
the background; a packet heard wakes it. Their messages reach the
Messages store as a transport ([RFC 0029](0029-messages.md)), as each
app chooses.

### Settings

([RFC 0025](0025-settings.md))

| Setting | Starting value |
|---|---|
| `radio.region`: the band plan | from the network's country, else the radio's band |
| `radio.<kind>.app`: the app the radio goes to | none: the first that asks |
| `radio.<kind>.antenna`: internal or external, where the device has a switch | internal |

RFC 0025's settings are declared before use and have no lists, so the
per-radio settings are named by the radio's kind as Board names kinds
(`radio.lora.app`); a second radio of a kind would take numbered ones.

An external antenna fitted but not chosen detunes the internal one (by
about 15 dB on the first device): the setting switches the antenna line
Board's description names (the switch is Radio's, RFC 0024), and the
system UI asks when the radio is first used.

### Who may use a radio

- **Apps** need the permission to use radios, asked when first needed,
  and declare the kinds they need in their manifest (RFC 0008), so a
  device without one does not offer them.
- **The rules hold for everyone:** no caller, native or not, sends
  outside the region's rules; a native program that opened the driver
  itself would go around them, which RFC 0004's one owner per device
  forbids.

### Where it runs

The radios program (RFC 0007), beside Bluetooth and NFC, on its event
loop: packets are short and few, and the driver's waiting is in the
kernel.

## Rules it follows

- **One owner per device** ([RFC 0004](0004-layers.md)): Radio alone
  opens the radios' drivers; callers get packets, not the device.
- **Policy where it belongs:** the law is Radio's, every protocol keeps
  to it; which network has the radio is the owner's; the protocols are
  the apps'.
- **No lending:** RFC 0004 lets an owner lend its device; Radio does not
  lend its radios, since a borrower could leave the region's rules. This
  changes RFC 0007, which had Radio lend a radio to the protocol.
- **Streams with credit** (RFC 0004): a slow reader loses packets, not
  the radio.

## Alternatives

- **Lending the device** (RFC 0004's lending): the protocol drives the
  radio itself, with the lowest latency; but the driver's own controls
  let it leave the rules, and WebAssembly apps hold no device files.
- **Time-sharing between networks:** Radio would alternate between
  configurations, and each network would miss what is sent while the
  other listens: worse for both than one that hears everything.
- **Air time left to the protocols:** less in Radio, but one careless
  app would break the law for the device.
- **Mesh protocols built in** as native services: faster and always
  there, but each network's software would be firmware's to follow and
  update; as apps, each comes and is updated through the store.

## Costs and risks

- **Exact timing through packets:** LoRaWAN's receive windows open a set
  time after a send, to within microseconds. Packets through Radio would
  need "listen at" times; left until a LoRaWAN app comes.
- **The band plans are law,** and laws change: the table is data, kept up
  to date like the time zones.
- **Air time is counted, not measured:** from each packet's length and
  configuration, by the chip maker's formula, rounded up, so that an
  error keeps within the law.
- **The shared SPI bus:** a packet is a few hundred bytes; a screen
  refresh may delay it a little, not lose it.
