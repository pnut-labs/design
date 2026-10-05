# RFC 0016: USB file transfer

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

A computer reaches the owner's files over USB: by MTP, the default, while
the device keeps using them, or as a USB drive, for large copies. Only the
owner's files on external storage are shown, never the device's private
storage. Storage runs both, with an MTP responder of the project's own.

## Problem

- [RFC 0012](0012-developer-access.md) has the device ask, when it is
  plugged into a computer, what the USB port is for, with file transfer
  as the default, and leaves file transfer to this RFC.
- [RFC 0010](0010-file-layout.md) keeps the owner's files on the FAT
  partition of each external storage, mounted at `/media/<kind><n>`, and
  [RFC 0011](0011-storage.md) gives that storage to the Storage service.
- NuttX has the USB side of MTP, a driver that hands the USB endpoints to
  a program, but no MTP responder: the part that lists, sends and
  receives files. It also has USB mass storage.

## Proposal

### Two ways

| Choice | Is | While it is on |
|---|---|---|
| **File transfer** (the default) | MTP: the computer sees the files, and asks for them one by one | the device keeps using its storage as usual |
| **USB drive** | USB mass storage: the computer gets the owner's partition as a disk | the partition is lent to the computer and unmounted on the device, since FAT cannot have two writers |

Both are among the choices the device offers when it is plugged in
(RFC 0012), which gain "USB drive". MTP works on Windows and Linux as they
are, and on macOS with a free app; a USB drive works everywhere, and is
the faster for large copies.

### What a computer sees

**Only the owner's files:** the FAT partition of each external storage,
such as a memory card or a built-in eMMC, with the folders apps keep there
for the owner (RFC 0011). Never the apps' partition, internal flash, or
anything of the system's.

Over MTP, each external storage is one storage on the computer. As a USB
drive, each owner's partition is one drive.

### Storage runs it

The Storage service owns external storage (RFC 0011), so it runs both:

- **MTP:** a responder module in Storage, whose transfers run on the data
  program's workers, which are there for flash
  ([RFC 0007](0007-services.md)). Files the computer adds or removes are
  announced like any other change.
- **USB drive:** Storage unmounts the owner's partition, lends it to USB
  mass storage, and mounts it again when the computer lets go or the
  cable is pulled. Meanwhile apps' own folders and the shared files are
  unavailable, and Storage announces it; installed apps keep running,
  since their data is on the apps' partition, which is not lent.

### The responder

A small MTP responder of the project's own, with only what computers
actually use: listing storages and folders, reading, writing, deleting and
renaming files, and telling the computer what changed. cmtp-responder
(Apache-2.0, Tizen's responder, written for Linux) serves as a reference.

### Locking

As for adb (RFC 0012): a session can only begin while the device is
unlocked, and stays open when the screen locks. While the device is
locked, the port only charges until it is unlocked and the owner answers.

## Alternatives

- **MTP only.** One way, but large copies are slow.
- **USB drive only.** Already in NuttX, but the device loses the owner's
  files for as long as a computer has them.
- **Porting cmtp-responder.** Mature, but written for Linux, and larger
  than what computers use.
- **uMTP-Responder.** The most active responder, but GPL-3.0, which would
  put GPL code in the firmware.
- **Ending a session when the screen locks.** More private, but a long
  copy would stop whenever the screen locked.

## Costs and risks

- **A responder to write and keep:** MTP has many operations, and
  computers differ in which ones they use.
- **A partition lent as a USB drive** can be left damaged if the cable is
  pulled while the computer writes.
- **Speed** is limited by the chip's USB and the storage; to be measured.
- **A computer already connected** keeps the owner's files after the
  screen locks, until the cable is pulled.
