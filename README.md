# pnut-labs design

The architecture of **pnut-os**, an operating system for small phones: an
e-paper handheld with a keyboard, cellular, LoRa mesh messaging, Wi-Fi and
Bluetooth, built on Apache NuttX, with apps as WebAssembly packages.

This repository is where the system is designed before it is built. The plan
is to write the design down here first, agree on it in review, and then
rewrite the system to it. The code lives elsewhere; this repository holds
what the system should be and why.

## What is here

| Path | What |
|---|---|
| [`rfcs/`](rfcs/) | Design proposals, one per topic, numbered. Each is discussed in its pull request and becomes part of the design when merged. [How it works](rfcs/README.md). |
| [`current/`](current/) | The system as it is today (pnut-os v2 on the LilyGo T-Deck Max): the starting point, and what the RFCs change. |
| [`hardware/`](hardware/) | The devices pnut-os runs on: the LilyGo T-Deck Max (more may follow). |

Principles and the service map will come here as the first RFCs are
accepted.

## How to take part

- **Read** the documents in GitHub; diagrams are Mermaid and render in place.
- **Comment** on a proposal in its pull request, on the lines you mean.
- **Ask** or float an idea before it is a proposal in
  [Discussions](../../discussions).
- **Propose** a change as an RFC: copy [`rfcs/0000-template.md`](rfcs/0000-template.md)
  and open a pull request (see [`rfcs/README.md`](rfcs/README.md)).

Pull requests are merged by the maintainer once the discussion has settled.

## Writing style

Plain words, short sentences. Say what a thing does before how. Each
document says its status at the top (draft, accepted, superseded) and the
date it last changed.
