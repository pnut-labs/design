# RFC 0010: The file layout

- **Status:** Draft
- **Author:** Mateusz Pianka
- **Created:** 2026-10-05
- **Last changed:** 2026-10-05
- **Supersedes / superseded by:** —

## Summary

Where everything lives: a tree named as on Linux, the partitions of the
internal flash, and the apps' storage on external storage. Internal flash
holds the firmware, the system's state and the package store. Installed
apps keep their data on external storage, which the device prepares for
them; without it, they do not run.

## Problem

The RFCs so far name places without placing them:

- [RFC 0008](0008-runtime.md): the firmware's list of default apps,
  `/etc/apps/defaults`;
- [RFC 0009](0009-packages.md): the package store, a read-only partition
  of its own; each app's data and cache; external storage, reached through
  two permissions;
- [RFC 0005](0005-communication.md): every service's socket, at a
  well-known path.

Internal flash is small, often 16 MB, half of it taken by the firmware,
and apps' data does not fit beside the system.

## Proposal

### What NuttX allows

- **Mount points, sockets and other special files live only in NuttX's
  in-memory tree,** never inside a mounted volume: NuttX refuses to create
  one at a path inside a volume (`inode_reserve()` in its
  `fs/inode/fs_inodereserve.c`). Local sockets are put under `/var/run`,
  shared memory under `/var/shm`, message queues under `/var/mqueue`, by
  default (`NET_LOCAL_VFS_PATH`, `FS_SHMFS_VFS_PATH`,
  `FS_MQUEUE_VFS_PATH`). So `/var` cannot be one volume: volumes are
  mounted below it.
- **NuttX mounts a read-only image built into the firmware at `/etc`**
  when it starts (its `ETC_ROMFS` option).
- **littlefs mounts on a block device** as well as on flash, so it can
  hold a partition of an SD card or an eMMC. NuttX reads MBR and GPT
  partition tables.

### The tree

Named as on Linux, with the Filesystem Hierarchy Standard's meanings:

| Path | What | Where |
|---|---|---|
| `/bin` | native programs built into the firmware | the firmware |
| `/etc` | the firmware's configuration: `init.rc`, `apps/defaults` (RFC 0008), default settings, the project's key (RFC 0009) | the firmware, read-only |
| `/usr/lib/apps/<id>` | built-in WebAssembly apps | the firmware, read-only |
| `/usr/share` | fonts, icons, themes, translations | the firmware, read-only |
| `/opt/<id>` | the package store: installed apps (RFC 0009) | internal flash, read-only |
| `/var/lib/<service>` | services' state: settings, messages, contacts, grants | internal flash |
| `/var/lib/apps/<id>/data`, `.../cache` | built-in apps' data and cache | internal flash |
| `/var/log` | logs kept across restarts | a soft link in memory, to `/var/lib/log` on internal flash |
| `/var/opt/<id>/data`, `.../cache` | installed apps' data and cache: the Filesystem Hierarchy Standard keeps the changing data of `/opt` packages in `/var/opt` | the apps' storage (below) |
| `/var/run/<service>` | services' sockets (RFC 0005) | memory |
| `/var/shm`, `/var/mqueue`, `/var/sem` | shared memory, message queues, named semaphores | memory |
| `/tmp` | temporary files, gone at restart | memory |
| `/dev` | devices; uORB topics in `/dev/uorb` | memory |
| `/proc` | tasks, memory, mounts | memory |
| `/media/<kind><n>` | external storage | the storage itself |

### Internal flash

| Partition | Mounted at | Mode | A factory reset |
|---|---|---|---|
| Whatever the device's boot needs | — | — | keeps it |
| Firmware: one or two slots, decided in the firmware-update RFC | — | — | keeps it |
| Device data: serial number, calibration, device keys | not mounted; Board and Security read it | read-only | keeps it |
| System | `/var/lib` | read-write | erases it |
| Package store | `/opt` | read-only, switched for changes (RFC 0009) | erases it |

Sizes are set for each device. On 16 MB, for example: two firmware slots
of 4 MB, 64 KB of device data, 2 MB for the system, and the rest, about
5.9 MB, for the package store.

- **Logs live in the system partition,** in `/var/lib/log`, and
  `/var/log` is a soft link to it in the in-memory tree (NuttX's
  `FS_LINKS`). The log service is
  their only writer and caps their size by rotating them.
- **Every littlefs volume costs RAM** for its caches, about 3 KB with
  NuttX's defaults, so internal flash has only two.

### The apps' storage

Installed apps keep their data and cache on external storage, in **the
apps' storage**.

- **The owner chooses which storage** holds it, one at a time; by
  default a built-in eMMC if the device has one, otherwise the memory
  card.
- **The device prepares it** by splitting it in two, with an MBR
  partition table:

  | Partition | File system | Mounted at | Holds |
  |---|---|---|---|
  | The owner's files | FAT | `/media/<kind><n>` | music, maps, photos: what a computer or camera reads |
  | The apps' partition | littlefs | `/var/opt` | installed apps' data and cache |

  The apps' partition takes **10% of the storage, at least 32 MB and at
  most 2 GB**. Preparing a storage erases it, so the owner confirms first.
  The system UI shows the apps' partition on the storage, so the owner
  knows where the space went.
- **It is encrypted** and tied to the device; how is designed in the
  Security RFC. A storage prepared on another device is not used until the
  owner prepares it again.
- **Installed apps need it.** Without the apps' storage, installed apps do
  not run. If it is removed while they run, the apps using it are stopped
  at once (their data is gone, so `stop` cannot save anything), the system
  UI tells the owner, and background apps start again when it returns.

**Built-in apps and services keep their data on internal flash,** in
`/var/lib`, so the device works without external storage: messages,
calls and the built-in apps do not depend on a card.

### External storage

Every external storage is mounted under **`/media`**, named by its kind
and number: `/media/sd0`, `/media/emmc0`, `/media/usb0`, and network
shares when there are any, such as `/media/nfs0`. On the apps' storage,
that is its FAT partition. The two permissions of RFC 0009 reach these:
an app's own folder, and the shared files.

### In memory

- **`/tmp`** is NuttX's in-memory file system, with a fixed size.
- **NuttX takes that memory from the file systems' heap,** which is the
  kernel's internal RAM by default, the scarcest. NuttX can give file
  systems a heap of their own in a chosen part of memory (`FS_HEAPSIZE`,
  `FS_HEAPBUF_SECTION`), so it goes to external RAM. The littlefs caches
  and RFC 0005's shared-memory rings come from the same heap, so they move
  too.

### Factory reset

A factory reset formats the system partition and the package store, and
the apps' partition on the apps' storage when it is present. It keeps the
firmware and the device data, and the owner's files on external storage.

## Alternatives

- **Android's names** (`/system`, `/data`, `/sdcard`). Familiar from
  phones, but the project's tools and habits come from Linux.
- **A partition of their own for the logs.** Their wear and size would be
  kept apart, but it would cost another littlefs volume's RAM, and a fixed
  part of the flash.
- **Apps' data on internal flash,** in a partition of its own. No external
  storage needed, but it would leave a few megabytes for every app's data.
- **The whole external storage as FAT,** apps' data in a hidden folder.
  Nothing to prepare, but FAT is not safe when power is lost during a
  write, the path would change with the medium, and the shared-files
  permission would have to be kept out of the folder by path.
- **The whole external storage as littlefs.** The simplest for the device,
  but a computer could not read the owner's files.
- **A GPT partition table.** Newer, but cameras and older computers
  expect MBR on memory cards.
- **`/tmp` on flash.** More room, but wear for files that are thrown away.

## Costs and risks

- **Installed apps do not run without external storage,** and stop when
  it is removed.
- **Apps' data is not protected** until encryption is designed in the
  Security RFC.
- **The system partition holds every service's state and every built-in
  app's data** in about 2 MB on a 16 MB device.
- **The owner's files are on FAT,** which can be damaged when power is
  lost during a write.
- **littlefs on an SD card** has not been measured; on internal flash, the
  previous system measured 2–4.5 s for each SQLite commit.
- **A file systems' heap in external RAM** has to be checked on each
  device: a chip may be unable to reach external RAM while it writes
  flash.
- **The internal partitions' sizes are fixed;** changing them later means
  moving data.
