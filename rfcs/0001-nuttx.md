# RFC 0001: NuttX as the operating system

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-09-28
- **Last changed:** 2026-09-28
- **Supersedes / superseded by:** —

## Summary

pnut-os is built on Apache NuttX. This RFC records what the system needs
from the layer below it, why NuttX, and what is still open. It decides the
base only: which of NuttX's parts the system uses is decided in later RFCs.

## Problem

Everything in pnut-os sits on an operating system: scheduling, memory,
drivers, file systems, networking, and the way separate programs run and
talk. The choice fixes the programming model for all the layers above, so
it comes first.

The system needs from it:

- **Separate programs:** parts of the system run as programs of their own,
  and one can fail without taking the others down.
- **The device's hardware** ([T-Deck Max](../hardware/t-deck-max.md)):
  Wi-Fi and Bluetooth LE on the ESP32-S3, light sleep, PSRAM, USB, and
  the buses its parts sit on.
- **Storage:** file systems on flash and on the SD card.
- **Networking:** TCP/IP over Wi-Fi and over the modem, and communication
  between programs.
- **Development:** a shell on the device and remote access.

## Proposal

**Apache NuttX**, programmed through its **POSIX** interfaces: programs are
tasks with their own file descriptors, and code written against POSIX is
not tied to the ESP32-S3, so it can move to another chip NuttX supports.

**What NuttX offers**, for the later RFCs to choose from:

- the ESP32-S3 port by Espressif, with Wi-Fi and Bluetooth LE through
  Espressif's libraries, and light sleep
- drivers for the device's parts
- file systems for flash, the SD card and read-only images, and an
  in-memory pseudo file system
- a network stack, with PPP, and local sockets
- a shell and remote access tools
- in nuttx-apps, a library of ported software, including three WebAssembly
  runtimes (WAMR, wasm3, toywasm). A runtime like these would help a great
  deal with separating programs: a WebAssembly module reaches only its own
  memory and the functions its host gives it, a separation the flat build
  does not give native programs.

None of these is decided by this RFC.

## Alternatives

- **Zephyr.** The largest ecosystem, and many ready-made parts: a settings
  store, a message bus (zbus), a cellular modem layer, a well-regarded
  Bluetooth host. But everything runs as threads in one image, with no
  processes, so separate programs do not carry over. Not chosen.
- **ESP-IDF (FreeRTOS).** Espressif's own, with the most complete Wi-Fi,
  Bluetooth and sleep support; but tied to Espressif's chips, and threads
  in one image as in Zephyr. Not chosen.
- **Linux.** Needs an MMU; the ESP32-S3 has none.
- **RIOT, Tock.** Little or no support for the ESP32-S3's radios.

## Costs and risks

- **Internal RAM.** Espressif's Wi-Fi and Bluetooth libraries take most of
  the kernel's internal RAM. They would under any of the alternatives too,
  but it is the tightest limit of the system (figures in
  [T-Deck Max](../hardware/t-deck-max.md#memory-and-storage)).
- **A smaller community** than Zephyr's: problems on this board fall to us.
- **Bluetooth:** NuttX's Bluetooth host is less widely used than Zephyr's
  or ESP-IDF's.

## Open questions

- **Build mode:** the *flat* build (kernel and programs share one address
  space, no protection between them) or the *protected* build (kernel
  separated from the programs), and what protection costs in RAM.
- **Following upstream:** how the NuttX and nuttx-apps trees track
  upstream, and whether the board port and fixes are offered back.
