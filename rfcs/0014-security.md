# RFC 0014: Security

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

The Security service's design: keys come from a device key the software
never sees, combined with the owner's PIN, so encrypted data opens only on
that device and only after the owner unlocks it. Security runs the screen
lock, encrypts the apps' storage, keeps a password vault in KeePass's
format, holds keys for apps, and keeps passkeys and TOTP secrets.

## Problem

[RFC 0007](0007-services.md) makes Security a service in a program of its
own, and leaves its design to this RFC:

- keys and passkeys, in a secure element where the device has one,
  otherwise encrypted in flash, with the processor's key hardware where it
  has some;
- the vault behind a password manager, which itself is an app;
- the screen lock, the fingerprint reader, and the unlock state.

Since then, [RFC 0010](0010-file-layout.md) has made the apps' storage
encrypted and tied to the device, with device keys in the device data
partition, and [RFC 0013](0013-firmware-updates.md) has left hardware
secure boot to this RFC.

A device may have no secure element and no fingerprint reader. Its chip
may still offer hardware AES and SHA, a random number generator, and
eFuses: bits that can be written once and never changed.

## Proposal

### Where keys come from

Two things together:

- **The device key,** in an eFuse that only the chip's HMAC unit can use:
  software asks the unit to derive a key from it, and never sees the
  device key itself. A copy of the flash or of the memory card is useless
  away from the device. The fuse is burned once, when the device is first
  set up.
- **The owner's PIN or password,** stretched with a slow key-derivation
  function, so that guessing it takes time.

Security combines the two into the **unlock key**, from which every other
key is derived. Data encrypted under it opens only on that device, and
only after the owner unlocks it.

- **On a chip without such a unit,** the device key is kept in the device
  data partition instead. It is weaker: anyone who reads the flash over
  USB has it, and is left with guessing the PIN.
- **NuttX has no driver for the HMAC unit;** one is added in the NuttX
  fork ([RFC 0003](0003-repositories.md)).

### What is encrypted

| What | How |
|---|---|
| **The apps' storage** (RFC 0010) | a block layer under littlefs encrypts every sector with AES-XTS, using the chip's AES hardware, under a key derived from the unlock key; file names and sizes are hidden too |
| **Security's own secrets:** the vault, apps' keys, passkeys, TOTP secrets | under keys derived from the unlock key |
| **Internal flash as a whole** | not by default: the chip's flash encryption is permanent once turned on, and gets in the way of flashing the device again. It may be offered later as the owner's one-way choice |

- **After a restart,** the apps' storage stays closed until the owner
  unlocks the device the first time. Calls, SMS, the services and the
  built-in apps work from start-up, since their data is on internal flash
  (RFC 0010); installed apps start after the first unlock.
- **The block layer** is a general NuttX block driver that encrypts the
  sectors of the device under it, added in the NuttX fork.

### The screen lock

- **Unlocking:** a PIN or a password. A fingerprint, where the device has
  a reader, is a shortcut: the PIN or password is needed after every
  restart, and again after a while. The owner sets after how long, or
  after how many unlocks by fingerprint, it is asked again; one of the two
  always applies. By default, after 72 hours.
- **Wrong attempts:** after every few wrong attempts, the device waits
  before allowing another, longer each time.
- **Wiping after too many attempts:** the owner can set a number of wrong
  attempts after which the device wipes itself. Off by default.
- **A duress PIN:** the owner can set a second PIN that, when entered,
  wipes the device instead of unlocking it.
- **Asking for the PIN:** turning on Developer options, or native apps,
  needs it ([RFC 0002](0002-build-mode.md)).

**Wiping** destroys the keys: Security erases what the unlock key is
derived from, so everything encrypted under it is unreadable at once; the
partitions are then formatted, as in a factory reset (RFC 0010).

**Unlocking follows [RFC 0004](0004-layers.md):** the system UI shows
the lock screen and passes what the owner entered to Security; Security
checks it and announces the unlock state, which Storage and the other
services follow ([RFC 0011](0011-storage.md)).

### The password vault

The vault is a **KeePass file (KDBX 4)**, so the same vault opens in
KeePassXC and other KeePass apps on a computer.

- **Its own master password,** as in KeePass, since the file can leave the
  device. On the device, Security can keep the vault's key wrapped under
  the unlock key, so the owner opens it with the screen PIN or a
  fingerprint after entering the master password once.
- **Lighter settings:** KeePass's usual key derivation, Argon2, wants far
  more memory than the device has; the vault uses settings the device can
  run, which a computer opens just as well.
- **Only the password manager app** reaches the vault, through its own
  permission ([RFC 0009](0009-packages.md)); showing a password asks the
  owner first.
- **Passkeys and TOTP secrets** are kept in the vault, as KeePassXC keeps
  them. Security computes TOTP codes and signs with passkeys itself, so
  their secrets never leave it.

### Keys for apps

An app can keep keys in Security: Security makes the key, keeps it, and
signs or encrypts with it when the app asks, so the key never leaves
Security. For example, a mesh app's identity key. An app reaches only its
own keys, and they are deleted with the app. Keeping keys itself, with the
runtime's cryptography ([RFC 0008](0008-runtime.md)), remains possible.

### Later

- **Hardware secure boot,** where the chip itself refuses firmware that is
  not signed: revisited with the next hardware. NuttX does not offer it for
  every chip, burning its fuses cannot be undone, and it would lock out
  self-built firmware.
- **Acting as an authenticator** for other devices, for passkeys and TOTP,
  over Bluetooth, USB or NFC: a later RFC.

## Alternatives

- **The device key in the device data partition** on every chip. No new
  driver, but anyone who reads the flash has it.
- **The owner's PIN alone.** No device key to keep, but a short PIN can be
  guessed away from the device.
- **Encrypting each file** instead of each sector. No block layer, but
  file names and sizes stay visible, and every app's files go through the
  encryption one by one.
- **The chip's flash encryption from the start.** Everything on internal
  flash protected, but permanently, and flashing the device again becomes
  hard.
- **A vault format of the project's own.** Free of KeePass's limits, but
  the vault would open nowhere else.
- **Wiping after too many attempts by default.** Safer for a lost device,
  but a child or a pocket can wipe it.

## Costs and risks

- **In the flat build any native code can read Security's memory**
  (RFC 0007), keys in use included.
- **A forgotten PIN loses the installed apps' data;** only the vault can
  be opened elsewhere, with its master password.
- **The fuse is burned for good,** taking one of the chip's eFuse key
  blocks.
- **On chips without the HMAC unit,** the device key is only as safe as
  the flash.
- **The vault's lighter key derivation** makes guessing its master
  password faster than with KeePass's usual settings.
- **The block layer** costs time on every read and write of the apps'
  storage; to be measured.
- **Two drivers to carry** in the NuttX fork: the HMAC unit and the
  encrypting block layer.
