# RFC 0018: The authenticator

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

The device can act as a security key for computers and other devices:
FIDO2 first (passkeys, two-factor logins, SSH keys), OpenPGP card and PIV
later, over USB, NFC and Bluetooth LE. It shows which site and account
each request is for, and the owner confirms on its screen. It is built on
canokey-core, inside the Security service.

## Problem

[RFC 0014](0014-security.md) keeps passkeys and TOTP secrets in the
password vault, and leaves acting as an authenticator for other devices
to a later RFC.

- **How a security key talks to a computer:** as a USB HID device, over
  NFC, or over Bluetooth LE; phones also offer a "hybrid" way, through a
  QR code and a relay server run by Google or Apple.
- **NuttX has no USB HID device class.** Its USB device classes cover
  serial ports, networking, mass storage, MTP, adb, firmware updates and
  video.
- **nuttx-apps has NimBLE,** a Bluetooth LE host.
- **canokey-core** (C, Apache-2.0) implements FIDO2, OpenPGP card, PIV and
  TOTP in one core meant to be ported: each board provides a port layer
  (timing, randomness, flash, user presence, NFC, crypto hardware), and
  the core stays as it is.

## Proposal

### What it offers

| What | Is | When |
|---|---|---|
| **FIDO2** (with the older U2F) | passkeys and two-factor logins on websites; SSH keys kept on the device, which OpenSSH supports | first |
| **OpenPGP card** | GPG keys for signing and encrypting mail, signing commits, SSH through gpg-agent | later |
| **PIV** | the smart-card standard for logging into Windows or macOS, VPNs, SSH through PKCS#11, signing documents | later |

**TOTP codes** need no connection: the device shows them on its screen
(RFC 0014).

### How it talks to other devices

| Way | Needs |
|---|---|
| **USB** | a USB HID device class, written for NuttX in the NuttX fork ([RFC 0003](0003-repositories.md)); OpenPGP card and PIV will need a smart-card reader class (CCID) too |
| **NFC** | an NFC chip that can act as a card; the NFC service carries the exchange to Security ([RFC 0007](0007-services.md)) |
| **Bluetooth LE** | the FIDO2 Bluetooth framing, on NimBLE; the Bluetooth service carries it to Security |

The hybrid way, through a QR code, is not offered: it depends on relay
servers run by Google and Apple.

**On USB,** the security key appears beside file transfer as one
composite device whenever the device is connected, where the chip can
combine them ([RFC 0016](0016-usb-files.md)). Where it cannot, "Security
key" is one more choice when the device is plugged in, added to the
choices of [RFC 0012](0012-developer-access.md).

### Which credentials

Two kinds, and the owner chooses when a credential is created:

- **a passkey in the vault,** which also works in KeePassXC on a computer
  (RFC 0014);
- **a credential of the authenticator's own,** which never leaves
  Security, like one on a hardware key.

### Confirming

- **Every use is confirmed on the screen,** which shows the site and the
  account it is for: a plain security key cannot show what is being
  approved.
- **Verifying the owner** uses the device's own PIN or fingerprint
  (RFC 0014), not a separate FIDO PIN.
- **Attestation,** the proof of which model of key made a credential, is
  none or self-signed: there is no certified attestation key, so services
  that accept only certified keys refuse it.

### Built on canokey-core

| Part | What |
|---|---|
| **The core:** the FIDO2, OpenPGP, PIV and TOTP applets | canokey-core, in a fork in the pnut-labs organisation, like the NuttX forks (RFC 0003): pinned to a version until a newer one is tested and verified |
| **The port** | the project's: runs inside Security; the core's wait for the user's presence becomes the confirmation on the screen; randomness and timing from NuttX |
| **USB** | the project's NuttX drivers, HID now and CCID later, feeding the core |
| **Bluetooth LE and NFC** | the project's: canokey-core has no Bluetooth transport, and its NFC code is written for one particular NFC chip |

**Changes to the core itself,** carried in its fork:

- **using the passkeys in the vault:** its FIDO2 applet knows only its own
  credentials, so its credential lookup is extended;
- **storage,** if needed: its credentials go into Security's encrypted
  storage; whether that only means pointing it at the right place is to
  be checked.

**The core runs on a worker of its own** in the Security program
([RFC 0004](0004-layers.md)): it waits for the owner's confirmation while
a request is open, and must not hold up Security's other work. Security's
pool of workers grows by one for it.

## Alternatives

- **OpenSK,** Google's authenticator. Apache-2.0, but in Rust, which
  would bring a Rust toolchain into the build.
- **pico-fido.** Complete, but AGPL-3.0.
- **An authenticator of the project's own.** No fork to carry, but FIDO2,
  OpenPGP and PIV are large to write and to get right.
- **The hybrid way, through a QR code.** What phones offer, but it relies
  on servers run by Google and Apple.
- **Only the authenticator's own credentials.** No change to the core,
  but passkeys in the vault would not work for other devices.
- **A separate FIDO PIN.** As on hardware keys, but a second PIN to
  remember, on a device that has a screen and a lock already.

## Costs and risks

- **Two USB drivers to write and carry:** HID now, CCID later.
- **A fork of canokey-core** to keep up to date.
- **Keys are not in separate hardware,** as they are on a hardware key:
  in the flat build, native code can read Security's memory (RFC 0007),
  keys included.
- **Bluetooth security keys** are not supported equally by every browser
  and system.
- **NFC** works only on devices whose NFC chip can act as a card.
- **Services that accept only certified keys** refuse it.
