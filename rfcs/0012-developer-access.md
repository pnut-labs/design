# RFC 0012: Developer access

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

How a developer's computer reaches the device: adb, over USB or the
network, with each computer authorized on the device's screen. It offers
the full shell, files, installing, logs and port forwarding, and runs only
while Developer options are on. When the device is plugged into a
computer, it asks what the USB port is for: file transfer by default, and
adb or the hardware debugger in Developer options.

## Problem

- [RFC 0002](0002-build-mode.md) lets the owner install native apps over
  USB only, with a switch in Developer options and each install confirmed
  on the screen.
- [RFC 0009](0009-packages.md) adds WebAssembly apps over USB, under the
  same conditions, and leaves to this RFC how a computer talks to the
  device.
- Developers also need a shell, files, logs and debugging. The previous
  system had a tool of its own over a USB serial port, at about 5 KB/s.

## Proposal

### adb

The developer's tool is **adb**, Android's debug bridge: its client runs
on every common operating system. NuttX's adb daemon, `adbd` in
nuttx-apps, is built from the microADB project.

- **microADB is pinned.** Upstream nuttx-apps downloads microADB's master
  branch, whatever it holds at the time; the nuttx-apps fork changes that
  to one fixed commit, moved only once a newer one has been tested and
  verified ([RFC 0003](0003-repositories.md)).
- **Keys are checked for real.** The previous system had to add the check
  of a computer's signature to microADB; it is a commit in the fork.
- **`adbd` runs only while Developer options are on.** The manager starts
  and stops it ([RFC 0006](0006-manager.md)).

| Command | Does |
|---|---|
| `adb shell` | the full shell, NSH, with full rights: in the flat build a developer has them anyway (RFC 0002) |
| `adb push`, `adb pull` | files, anywhere |
| `adb install` | hands the package to Packages, which installs it after the confirmation on the screen (RFC 0009) |
| `adb logcat` | the Log service's logs, live and kept |
| `adb forward` | port forwarding, for debugging services |

### Transports

- **USB,** when the owner chooses adb for the port (below).
- **The network,** over Wi-Fi, through a separate switch in Developer
  options. It is off by default, and off again after every restart.

**SSH** is possible as well, over the network: dropbear, in nuttx-apps,
behind its own switch with the same rules. By default it accepts keys
only, never passwords; a developer can change that in its configuration
file, on their own responsibility.

### Authorizing a computer

- **Only with Developer options on.**
- **Each computer's key is accepted on the device's screen** the first
  time, and remembered. Developer options can revoke them all.
- **A connection can only be opened while the device is unlocked.** A
  connection already open stays open when the screen locks.

### The USB port

When the device is plugged into a computer, **the system UI asks what the
port is for:**

| Choice | Offered | Is |
|---|---|---|
| **File transfer** (the default) | always | MTP: the owner's files, designed in its own RFC |
| **adb** | with Developer options on | the developer's tool (above) |
| **Serial and JTAG** | with Developer options on, where the chip has it | the console and the hardware debugger |

- **While the device is locked,** the port only charges until it is
  unlocked and the owner answers.
- **On some chips** the port is either a fixed serial-and-JTAG interface
  or a general USB device, never both at once. Choosing "Serial and JTAG"
  then gives up the other two until the port is switched back. Where the
  chip allows, one composite device carries adb and a serial console
  together.
- **The Board service switches the port.** Board is the service that
  keeps what is specific to one device ([RFC 0007](0007-services.md)), and
  what the port can be differs from device to device.

### Debugging

- **Native code:** GDB over JTAG, when the port is in "Serial and JTAG".
- **Crashes:** crash reports are kept with the logs, and pulled with adb.
- **WebAssembly apps:** WAMR has a debugger interface (lldb's remote
  protocol). It costs memory, so it is on only with Developer options, and
  is designed later, with the SDK.

## Alternatives

- **SSH only.** Standard and encrypted, but over the network only (or USB
  networking), heavier, and without installing or logs built in.
- **A protocol of the project's own** over a USB serial port, like the
  previous system's tool. Needs the project's tools on every computer.
- **Only standard USB classes,** a serial console and MTP. No special
  tools, but no installing, logs or forwarding.
- **One USB use chosen once,** in Settings, instead of asking. Fewer
  questions, but the port would stay in a mode the owner forgot about.
- **Refusing every connection while the device is locked,** open ones
  included. Safer, but a debugging session would end whenever the screen
  locked.

## Costs and risks

- **adb over the network is not encrypted:** anyone on the same network
  can read the session, and change what it carries, such as the commands
  sent to the shell. Developer options are for developers, who are
  expected to use it only on networks they trust; SSH is the encrypted
  choice.
- **microADB is pinned and patched** in the fork, so its fixes are taken
  by hand.
- **The shell has full rights:** an authorized computer can read and
  change everything on the device.
- **A computer already authorized** keeps working after the screen locks,
  until the connection closes.
