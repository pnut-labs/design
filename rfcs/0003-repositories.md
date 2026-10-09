# RFC 0003: Repositories and upstream

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-09-28
- **Last changed:** 2026-10-09
- **Supersedes / superseded by:** —

## Summary

pnut-os lives in the `pnut-labs` organisation: its own forks of NuttX and
nuttx-apps, based on NuttX releases, and a repository for everything that
is not NuttX. Our changes to NuttX are kept as separate commits, so each
could be offered upstream, but nothing depends on upstream taking them.

## Problem

pnut-os builds on NuttX and nuttx-apps ([RFC 0001](0001-nuttx.md)), and
changes them: the board, its drivers, and fixes found along the way. The
system needs a home for those changes, a base to follow, and a way to keep
them apart from the system itself.

Offering the changes upstream cannot be the plan. NuttX requires every
commit made with AI tools to say so (`Assisted-by:`), accepts a sign-off
only from a person, and its community expects contributions to be
substantially the contributor's own work. Much of this project's code is
written with AI tools.

## Proposal

### The repositories

| Repository | What |
|---|---|
| [`pnut-labs/design`](https://github.com/pnut-labs/design) | The design: RFCs and hardware descriptions |
| [`pnut-labs/nuttx`](https://github.com/pnut-labs/nuttx) | A fork of `apache/nuttx` |
| [`pnut-labs/nuttx-apps`](https://github.com/pnut-labs/nuttx-apps) | A fork of `apache/nuttx-apps` |
| [`pnut-labs/pnut-os`](https://github.com/pnut-labs/pnut-os) | The system |

Every repository's default branch is `master`, and changes reach it only
through pull requests.

### Where things live

- **The NuttX fork:** the board (`boards/xtensa/esp32s3/lilygo-tdeck-max/`)
  with its configurations and documentation page, the drivers for its parts
  in NuttX's `drivers/`, and fixes to NuttX.
- **The nuttx-apps fork:** fixes to nuttx-apps.
- **pnut-os:** everything that is not NuttX: the system's programs, apps,
  tools and SDK.

The test: a change that makes sense to someone using NuttX without pnut-os
goes in a fork; the rest goes in pnut-os.

Drivers stay in `drivers/` because their parts are generic (UC8253,
TCA8418, BQ27220, DRV2605L, BHI260AP) and other boards can use them; the
board directory holds only this board's wiring.

### The build configuration

A build is described in two parts, each kept where it belongs:

- **The board's configuration**, in the NuttX fork with the board (for
  example `lilygo-tdeck-max:nsh`): the hardware and its drivers. It builds
  and can be tested without pnut-os.
- **pnut-os's additions**, in pnut-os, as a short fragment: its programs,
  what its services need from NuttX, and its start-up.

pnut-os's build merges the fragment onto the board's configuration with
NuttX's `tools/merge_config.py`, then configures NuttX with the result.
When both set an option, the fragment wins, so a release move checks the
fragment against what changed in the board's configuration.

### Following upstream

- The forks follow **NuttX releases**. `master` is a release with our
  commits on top.
- **Moving to a new release** replays our commits onto it, in a pull
  request.
- **A single upstream fix** needed before the next release is picked from
  upstream `master` with a reference to its original commit.

### Commits

Our commits in the forks stay separable:

- **One commit per change:** a general fix, a driver, or a step of the
  board, never mixed.
- **Named as NuttX names them:** after the part they touch
  (`fs/fat: …`, `drivers/lcd/uc8253: …`, `boards/lilygo-tdeck-max: …`).
- **With `Assisted-by:`** in NuttX's form when made with AI tools
  (`Assisted-by: AGENT_NAME:MODEL_VERSION`).
- **`Signed-off-by:` only by a person**, never by a tool.
- **A pull request to a fork is one change,** squashed into one commit
  when it merges, as in design and pnut-os; the forks keep the branch.

### Fixes found by pnut-os

A bug in NuttX found by pnut-os, by its tests in the simulator or on a
device, is fixed in the fork:

- **in a pull request of its own,** named for what it fixes
  (`net/local: …`), saying what failed and how it was found;
- **the test that found it stays in pnut-os,** and runs against every fork
  commit pnut-os moves to ([RFC 0022](0022-build.md)): the forks' own
  checks only build;
- **a fix that upstream already has** is picked from there instead
  (above).

### Offering changes upstream

Not assumed. A person may offer a fix or a driver upstream when they have
reviewed it, can explain it, and take responsibility for it under NuttX's
rules.

## Alternatives

- **Following upstream `master`.** Newest fixes, and Espressif's ESP32-S3
  work lands there first; but it moves fast and sometimes breaks.
- **Freezing on one base.** Nothing changes underneath, but the gap to
  upstream keeps growing.
- **The board outside the NuttX tree** (a "custom board" in pnut-os). Keeps
  the fork to general fixes only, but the board belongs with NuttX, where
  boards live.
- **Drivers in the board directory.** Ties generic drivers to one board.
- **pnut-os's configuration in the NuttX fork**, next to the board's. The
  usual NuttX place, but it only works with pnut-os present, and every new
  pnut-os option needs a pull request in the fork too.
- **pnut-os's whole configuration in pnut-os.** One repository per change,
  but it repeats the board's settings, which then have to be kept in step.
- **Personal forks.** Not a shared home.

## Costs and risks

- **Every release move is work:** replaying our commits, and checking what
  changed underneath.
- **Carrying everything ourselves:** fixes that are not taken upstream stay
  ours to maintain.
