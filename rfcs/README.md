# RFCs: design proposals

An RFC ("request for comments") is a proposal for one part of the design:
a service, an interface, a rule, a screen, a piece of hardware. It says what
problem it solves, what it proposes, and what else was considered.

## The process

1. **Idea.** Optional: raise it in Discussions first, to see whether it is
   worth a proposal.
2. **Draft.** Copy `0000-template.md` to `NNNN-short-name.md` with the next
   free number, fill it in, set the status to *Draft*, and open a pull
   request titled `RFC NNNN: <title>`.
3. **Review.** Comments go on the pull request, on the lines they are about.
   The author revises in the same pull request; questions that stay open go
   in the RFC's *Open questions*.
4. **Decision.** The maintainer merges it with the status *Accepted*, or
   closes it (the file is then merged as *Rejected*, so the reasoning is
   kept).
5. **Later.** An accepted RFC is not rewritten to change its decision: a
   new RFC supersedes it, and both say so at the top. Small corrections
   (typos, clearer wording) are fine without a new RFC.

An accepted RFC becomes code, and the code's reference documentation
(interfaces, wire formats) is written from it.

## Status

| Status | Meaning |
|---|---|
| Draft | Open for review in its pull request |
| Accepted | Part of the design |
| Rejected | Considered and not taken; kept for the reasoning |
| Superseded | Replaced by a later RFC, named at the top |
| Implemented | Accepted and built; links to the code |

## Index

| RFC | Title | Status |
|---|---|---|
| [0001](0001-nuttx.md) | NuttX as the operating system | Accepted |
| [0002](0002-build-mode.md) | Build mode and trust | Accepted |
| [0003](0003-repositories.md) | Repositories and upstream | Accepted |
| [0004](0004-layers.md) | Layers and rules | Accepted |
| [0005](0005-communication.md) | Communication | Accepted |
| [0006](0006-manager.md) | The manager | Accepted |
| [0007](0007-services.md) | Services and programs | Accepted |
| [0008](0008-runtime.md) | The app runtime | Accepted |
| [0009](0009-packages.md) | Packages | Accepted |
| [0010](0010-file-layout.md) | The file layout | Accepted |
| [0011](0011-storage.md) | Storage for apps | Accepted |
| [0012](0012-developer-access.md) | Developer access | Accepted |
| [0013](0013-firmware-updates.md) | Firmware updates | Accepted |
| [0014](0014-security.md) | Security | Accepted |
| [0015](0015-network.md) | Network access for apps | Accepted |
| [0016](0016-usb-files.md) | USB file transfer | Accepted |
| [0017](0017-i18n.md) | Languages and regions | Accepted |
| [0018](0018-authenticator.md) | The authenticator | Accepted |
| [0019](0019-system-ui.md) | The system UI | Accepted |
| [0020](0020-notifications.md) | Notifications | Accepted |
| [0021](0021-sdk.md) | The SDK | Accepted |
| [0022](0022-build.md) | Building and testing pnut-os | Accepted |
| [0023](0023-service-library.md) | The service library | Accepted |
| [0024](0024-board.md) | Board | Accepted |
| [0025](0025-settings.md) | Settings | Accepted |
