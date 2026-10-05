# RFC 0013: Firmware updates

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

The firmware has two slots, and MCUboot starts it. A new firmware is
written into the unused slot while the device runs, then started on
trial; the manager confirms it once the system is up, and otherwise
MCUboot goes back to the previous one. Updates come from the project's
update server, or over USB with developer access.

## Problem

- [RFC 0010](0010-file-layout.md) leaves to this RFC whether the firmware
  has one slot or two.
- NuttX can start the firmware with no bootloader at all, from the start
  of the flash. Then nothing can switch between two firmwares, and a
  failed update leaves a device that only starts again when it is flashed
  over USB.
- [RFC 0002](0002-build-mode.md) lets the owner build their own firmware,
  and [RFC 0009](0009-packages.md) ties native apps to the firmware build
  they were made for.

## Proposal

### Two slots and MCUboot

The firmware is started by **MCUboot**, a widely used bootloader for small
devices; NuttX supports Espressif's port of it as the bootloader that
starts the firmware.

| Part | What |
|---|---|
| MCUboot | the bootloader |
| The first slot | the running firmware |
| The second slot | where an update is written, and the previous firmware is kept |
| The scratch area | MCUboot's room for swapping the two |

- **Swapping, with a way back:** MCUboot swaps the two slots when it
  starts an update, so the previous firmware stays in the second slot.
  If the new one is not confirmed, MCUboot swaps them back at the next
  start.
- **Sizes are set for each device.** On 16 MB, the two slots take 4 MB
  each. NuttX's defaults keep 128 KB for MCUboot before the first slot,
  and 256 KB for the scratch area, about 384 KB together, taken from the
  package store's share in RFC 0010's example.
- **Swapping without a scratch area:** MCUboot can also swap by moving
  the slots' contents along by one sector, which needs only one spare
  sector (4 KB) instead of the scratch area. It is used if Espressif's port
  supports it on the device; otherwise the scratch area is made smaller.

### Installing an update

**Packages** installs firmware updates as well as apps: it already
fetches from servers, checks signatures, looks for updates at night, and
writes to flash on the data program's workers (RFC 0009).

1. **Download** into the second slot, while the device runs from the
   first.
2. **Check** the signature (below), then mark the update for a trial.
3. **Restart,** when the owner says so (below). MCUboot swaps the slots
   and starts the new firmware on trial.
4. **Confirm.** The manager ([RFC 0006](0006-manager.md)) confirms the new
   firmware to MCUboot once every service is ready and the system UI is
   up. If the device restarts before that, MCUboot goes back to the
   previous firmware.
5. **Convert stored data** only after confirming: until then, data stays
   as the previous firmware can read it, so going back always works.

### Signing

- **Packages checks the signature before writing:** the project's key,
  which is in the firmware, or a key the owner has added in Developer
  options for their own builds.
- **MCUboot checks that the image is whole,** by its hash. Its keys are
  built into it, with no way to add the owner's later, so the signature
  check is Packages'.
- **Writing firmware with the chip's own USB loader** always remains
  possible, and passes no check. Hardware secure boot, where the chip
  itself refuses unsigned firmware, is left to the Security RFC.

### Where updates come from

- **The project's update server:** a signed index of firmware versions,
  with their sizes and hashes, in a stable and a testing channel, like the
  repositories of RFC 0009.
- **USB, with developer access** ([RFC 0012](0012-developer-access.md)):
  the image is handed to Packages, which installs it as above.

### When

- **Downloading** happens by itself over Wi-Fi; over mobile data only if
  the owner allows it.
- **Installing** needs a restart. It happens when the owner says so, or
  at night if the owner turned automatic installs on, and only while
  charging or above a battery level.
- **The owner is warned first** when installed native apps will stop
  working (below).

### Downgrades

Refused over the air. Over USB with developer access they are allowed,
on the developer's responsibility: an older firmware may not read data
converted by a newer one, and may need a factory reset (RFC 0010).

### Native apps

A native app is built for one firmware (RFC 0009). After an update it does
not start until it is rebuilt and installed again; the owner is told
before installing the update.

### Recovery

- **If neither firmware starts,** the chip's own USB loader, built into
  the chip, can still write a firmware.
- **A key combination at start-up,** set for each device, offers:
  - **safe mode:** the system without installed apps, to remove one that
    stops the device from working;
  - **factory reset** (RFC 0010).

### Starting values

| What | Value |
|---|---|
| Battery level below which an update is not installed | 50% |
| Channel | stable |

## Alternatives

- **One slot and a small recovery image.** More room, but the device is
  out of action while it updates.
- **One slot, overwritten in place.** The most room, but a failed update
  leaves a device that starts only after flashing over USB.
- **nxboot,** NuttX's own small bootloader in nuttx-apps. Smaller, but
  less proven.
- **Espressif's ESP-IDF bootloader.** Ties the update path to one chip
  vendor.
- **MCUboot checking signatures.** The check would be at start-up, but
  every owner's key would have to be built into MCUboot, so building one's
  own firmware would mean building and flashing one's own bootloader too.
- **A patch to MCUboot** reading the owner's keys from the device data
  partition. Signatures checked at start-up too, but a patch to carry in
  the fork, for little gain before hardware secure boot: until then the
  chip's own loader writes any firmware anyway.
- **Updating from a file on external storage.** Works without a network,
  but is not needed while USB is there.

## Costs and risks

- **The firmware takes half of a 16 MB device:** two slots of 4 MB.
- **Swapping takes time at restart,** and wears the flash; to be measured.
- **Without hardware secure boot,** anyone with the device and a USB cable
  can write any firmware.
- **A downgrade over USB** may need a factory reset.
- **Native apps stop working** at every firmware update until rebuilt.
