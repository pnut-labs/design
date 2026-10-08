# RFC 0022: Building and testing pnut-os

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-08
- **Last changed:** 2026-10-08
- **Supersedes / superseded by:** —

## Summary

How pnut-os is built and tested: a working tree with the NuttX forks as
submodules, pnut-os entering NuttX's build through nuttx-apps' `external`
directory, one `make` for every target (the device, NuttX's simulator,
QEMU), tests from unit tests to the device, and CI that runs them on every
pull request. The code follows NuttX's style, under the Apache 2.0
license; the design documents are CC BY 4.0.

## Problem

[RFC 0003](0003-repositories.md) says where things live (the board and
drivers in the NuttX fork, the system in `pnut-os`) and how a build is
described (the board's configuration plus pnut-os's fragment, merged with
NuttX's `tools/merge_config.py`). It does not say how a developer gets a
tree that builds, how pnut-os's code joins NuttX's build, what is built
for testing, or how changes are checked before they merge. `pnut-os` is
still empty, and neither it nor `design` has a license.

What NuttX offers:

- **the Make build;** NuttX's CMake build does not support the ESP32-S3;
- **its simulator,** which runs NuttX as a program on the computer;
- **QEMU configurations for the ESP32-S3** (Espressif's QEMU fork);
- **cmocka,** in nuttx-apps, for unit tests;
- **`nxstyle`,** its style checker.

## Proposal

### The working tree

`pnut-os` holds the NuttX forks as **git submodules**, pinned to exact
commits:

```
pnut-os/
  nuttx/          submodule: pnut-labs/nuttx
  apps/           submodule: pnut-labs/nuttx-apps
  ...
```

`git clone --recursive` gives a tree that builds, and every pnut-os
commit names the fork commits it was tested with. Moving the forks forward
is a pull request to `pnut-os` that updates the submodules, with the build
and tests run on the result.

### Joining NuttX's build

pnut-os's code enters NuttX's build through **nuttx-apps' `external`
directory:** NuttX's own way for programs outside its tree, a link from
`apps/external` to pnut-os's code. The build is NuttX's **Make** build.

A **top-level `Makefile`** in `pnut-os` is the one entry point:

| Command | Does |
|---|---|
| `make <target>` | merges the board's configuration with pnut-os's fragment, kept with the target's `init.rc` in `configs/<target>/` (RFC 0003), configures NuttX, and builds the image |
| `make <target> flash` | writes the image to the device, with the chip's tools |
| `make sim` | builds the simulator target and runs it |
| `make qemu` | builds the QEMU target and runs it |
| `make test` | runs the unit and system tests (below) |
| `make style` | runs `nxstyle` on pnut-os's code |

### Targets

A target is a board configuration plus pnut-os's fragment. How the rest
of `pnut-os` is laid out is documented in the repository itself, next to
the code (RFC 0003):

| Target | Is | For |
|---|---|---|
| **the device** | the board's configuration in the NuttX fork, with the drivers | the real thing |
| **`sim`** | NuttX's simulator, on the computer: the screen in a window, the device's parts stood in for | daily work, system tests; also the SDK's simulator ([RFC 0021](0021-sdk.md)) |
| **`qemu`** | Espressif's QEMU, running the ESP32-S3's instruction set | the runtime and the kernel on the real processor, without a device |

More devices are more targets.

### Tests

| | Where | Checks | Speed |
|---|---|---|---|
| **Unit tests** | on the computer, with cmocka, pnut-os's code apart from NuttX where it can be | encoding, state machines, parsing | seconds |
| **System tests** | the simulator | pnut-os boots, services become ready and talk to each other; scripts drive it through its console and adb over TCP ([RFC 0012](0012-developer-access.md)) | minutes |
| **Smoke tests** | QEMU | booting, and the runtime on the real instruction set | minutes |
| **On the device** | the device | everything the others cannot: radios, the screen, power | by hand for now; a test rig later |

### CI

- **Every pull request to `pnut-os`** builds every target and runs the
  unit, system and smoke tests, on GitHub Actions. The checks are required
  by the `master` ruleset: nothing merges with them failing.
- **The forks' CI** builds the board's configurations, so a change to a
  driver or the board is caught where it is made.
- **One container image** holds the toolchains (Espressif's for the
  device, the computer's compiler for the simulator, QEMU, wasi-sdk): CI
  and the SDK (RFC 0021) use the same one, so a build passes or fails the
  same way everywhere. Its versions are pinned, and moved on purpose.

### Language and style

- **System code is C,** C11, as NuttX's.
- **NuttX's coding style,** checked with its `nxstyle` tool in CI, so code
  moves between pnut-os and the forks unchanged.
- **Commit messages** in `pnut-os` name the part they touch, as NuttX's do
  (`system/settings: …`); each pull request is squashed into one commit
  (RFC 0003).
- **Every source file** carries an SPDX license line, as NuttX's do.

### Licenses

| Repository | License |
|---|---|
| `pnut-os` | **Apache 2.0,** like NuttX: permissive, and contributors grant a patent license |
| `design` | **CC BY 4.0:** the documents may be shared and built on, with credit |
| the forks | Apache 2.0, as upstream |

`pnut-os` starts with its `LICENSE` file; `design` gains one in a pull
request of its own.

## Alternatives

- **A workspace manifest** (Google's `repo` tool, or a script with a lock
  file) instead of submodules. The same pinning, but one more tool to
  learn; submodules are git's own.
- **The repositories side by side,** each at its own `master`. Simplest,
  but nothing says which fork commits a pnut-os commit was tested with.
- **NuttX's CMake build.** Newer, but it does not support the ESP32-S3.
- **A style of the project's own,** with clang-format. Automatic
  formatting, but code would be reformatted every time it crossed into a
  fork.
- **A copyleft license** (GPL). It would keep changes open, but NuttX's
  ecosystem is Apache 2.0, and code could not move freely between the
  repositories.
- **Only the simulator, or only QEMU,** for tests. The simulator is fast
  but not the real processor; QEMU is the real processor but slower, and
  runs none of the device's parts.

## Costs and risks

- **Submodules are easy to forget:** a tree cloned without `--recursive`,
  or not updated after a pull, builds the wrong forks. The Makefile checks
  and says so.
- **The simulator is not the device:** its stand-ins for the device's
  parts drift from the real drivers unless kept in step.
- **CI on every pull request** builds three targets and runs the tests;
  how long that takes is to be measured, and the container image's size
  with it.
- **The device's tests stay by hand** until there is a test rig.
