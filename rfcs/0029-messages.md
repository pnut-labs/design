# RFC 0029: Messages

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-30
- **Last changed:** 2026-10-30
- **Supersedes / superseded by:** —

## Summary

Messages is the device's one store of text messages: SMS, and the messages
of mesh apps and other messaging apps. The transports carry them:
Telephony carries SMS, and an app carries its own network's messages.
Messages keeps the conversations, queues what is to be sent, and tells
the system UI what changed. It runs in the data program. This RFC
proposes SQLite on internal flash, whose writes take seconds there, and
weighs that against the alternatives.

## Problem

- **The store is named, but not designed.**
  [RFC 0007](0007-services.md) gives Messages "one store for SMS and mesh
  messages; Telephony and mesh apps deliver into it", and
  [RFC 0008](0008-runtime.md) and [RFC 0009](0009-packages.md) let an app
  offer "a messaging transport". Nothing says:
  - what the store holds;
  - how a transport delivers into it, or takes what is to be sent;
  - how big it may grow, and how fast it writes;
  - who may read it.
- **Telephony's SMS needs it first** ([RFC 0030](0030-telephony.md)). An
  SMS the modem has received has to land somewhere that survives a
  restart before the modem lets go of it.
- **The previous system had the same shape** ([current/](../current/)):
  msgd, one SQLite database of threads and messages, with one transport
  per provider and an outgoing queue the provider took from. On the
  phone it carried long SMS both ways. Its database lived on the memory
  card, with internal flash only as a fallback, because writes there
  took seconds (below). Its waiting list was in memory, 32 deep; here the
  queue is in the store.

## Proposal

### What it keeps

- **Conversations.** There is one conversation for each transport and peer.
  - The peer is a phone number for SMS, or a node's or a channel's
    address on a mesh.
  - A conversation knows its transport and peer, how many messages it
    has, how many are unread, and the start of the newest text.
- **Messages,** each in one conversation. A message has:
  - its direction, in or out;
  - its text;
  - for a conversation with several people, such as a mesh channel, the
    sender, when it is not the peer;
  - the time the device stored it, by the device's clock;
  - the time it was sent, when the network gives it;
  - its status.
- **The order is the order they were stored,** in a conversation and in
  the list of conversations. Times are shown, but do not sort: the
  device's clock may be wrong until Time has set it
  ([RFC 0028](0028-time.md)).
- **Statuses.**
  - Going out: **queued** while its transport cannot take it, then
    **sending** once the transport has it, then **sent**. It becomes
    **delivered** where the network reports delivery, or **failed** with
    the transport's reason.
  - After a restart, a message left sending is **not known to be
    sent**. It is not sent again on its own; the owner can send it
    again. A message still queued is sent as usual.
  - Coming in: **unread** until shown, then **read**.
- **Text is UTF-8,** up to the limit below. Telephony delivers an SMS
  longer than that as several messages, in order. Pictures and other
  attachments, MMS among them, are not part of this RFC.
- **Peers are shown as they came:** a number, or an address. Names come
  from Contacts, once it exists (RFC 0007).

### Transports

A transport is a way messages travel. Its name is fixed:

- `sms` belongs to the program that runs Telephony, by the programs'
  table ([RFC 0006](0006-manager.md), RFC 0007), and Messages refuses
  it to anyone else;
- an app's transport is named by the app's id
  (`org.meshcore.companion`), which the runtime vouches for.

Then:

- **Registering.**
  - Telephony registers `sms` when it starts.
  - An app's transport comes from its manifest. Packages announces it
    at install (RFC 0009), and Messages asks Packages for the installed
    ones when it starts.
  - When an app is removed, its transport goes. Its conversations stay,
    read-only, and its queued messages fail with "transport removed".
- **Delivering.**
  - A transport hands Messages a message it received: the peer, the
    sender if any, the text, the time it was sent, and a key.
  - The key names the message on its network. For an SMS it is the
    sender, the network's time stamp (a long one's first part's), a long
    one's reference, and a hash of the text, since one sender can send
    two messages in the same second. Messages stores a message once for
    each key, so a delivery repeated after a reset is not stored twice.
  - Messages has the message on flash before it answers. Only then does
    the transport let go of it: Telephony deletes the SMS from the
    modem after that answer, not before.
  - When Messages cannot keep a message, because there is no room or
    the flash is failing, it says so, and the transport keeps it: an
    SMS stays in the modem. The owner is told (below).
- **Sending.**
  - The system UI asks Messages to send a text to a peer over a
    transport, as does an app that may. Messages answers once the
    message is stored, without waiting for the network.
  - The transport opens a stream, and Messages sends it what is queued,
    oldest first, as the transport gives credit
    ([RFC 0004](0004-layers.md), [RFC 0005](0005-communication.md)).
  - A message whose transport takes it at once is stored as sending, in
    the send's one transaction. A message that had to wait is written
    as sending before the transport gets it: a second transaction, only
    for messages that waited. So no message is handed over twice, even
    across a reset.
  - The transport reports each message as sent, delivered or failed.
- **A transport that is not running** leaves its messages queued:
  Telephony's until it runs and has the network, an app's until the app
  runs. How an app is started for that is an open question: RFC 0008
  starts an app when it is opened, called, scheduled or allowed in the
  background. Nothing in the queue expires on its own; the owner can
  delete a queued message.
- **Deleting** a conversation drops its queued messages unsent. A report
  about a deleted message is answered "not found".

### What others see

- **The interface is `pnut.Messages`,** and its settings' owner is
  `messages`.
- **A topic, `messages`,** says that a conversation changed. It gives the
  conversation's id, how many messages are unread in all, and whether
  the store is full. It gives no text and no peer, since any native code
  can read a topic (RFC 0005). Its messages are numbered, so a reader
  sees one it missed.
- **Reading.**
  - Conversations come in their order, a page at a time.
  - A conversation's messages come a page at a time.
  - Marking a conversation read.
  - Deleting a message or a whole conversation.
- **Who may.**
  - **The system UI** may do all of it.
  - **A transport** may deliver into its own conversations, take its own
    queue, report statuses, and read its own conversations.
  - **An app** reads and writes its own transport's conversations as a
    transport. To read the other conversations, SMS among them, it needs
    a permission to read messages. To send over another transport, it
    needs a permission for that transport: sending SMS costs money. The
    permissions are named in Messages' `.proto` (RFC 0005); this RFC
    fixes what each allows.
  - **A command typed in the developer's shell** may do what the system
    UI may, by the rule for the shell in [RFC 0025](0025-settings.md).
- **Notifications.** Once Notifications exists, Messages posts one
  notification for each conversation with unread messages, in a group of
  its own ([RFC 0020](0020-notifications.md)). It clears it when the
  conversation is read. It also tells the owner when the store is full
  and a delivery was refused. Until Notifications exists, the topic and
  the log are all there is.

### Where it keeps them

- **One SQLite database,** `/var/lib/messages/messages.db`, in the system
  partition on internal flash ([RFC 0010](0010-file-layout.md)). The data
  program links SQLite ([RFC 0011](0011-storage.md)).
- **Writes there take seconds.** The previous system measured SQLite on
  this flash on 2026-09-25, each statement its own transaction:

  | Per statement | littlefs, internal flash | FAT, memory card |
  |---|---|---|
  | insert | 2.2 s | 355 ms |
  | update, delete | 3.0–4.5 s | 300–420 ms |
  | read one row | 43 ms | 31 ms |
  | the latest 20 rows | 27 ms | 36 ms |

  littlefs never writes in place: each change rewrites and erases whole
  4 KB blocks, and SQLite's journal multiplies that.
- **So the store writes little, and in batches:**
  - A delivery is one transaction, and a send is one. They must be on
    flash before Messages answers. So is a waiting message's hand-over
    to its transport (above).
  - Later statuses (sent, delivered, failed) and marks of read are kept
    in memory. They are written together, at most every minute, or with
    the next transaction.
  - A reset loses at most a minute of them. A message left sending
    then shows as "not known to be sent", and messages read just before
    show unread again, their notifications posted again.
  - The journal is emptied rather than deleted (`journal_mode =
    TRUNCATE`), which costs fewer directory writes.
  - Step 1 of the work measures a delivery again with these choices,
    on this firmware.
- **All of SQLite runs on the program's workers,** reads too: a read
  takes tens of milliseconds, over a handler's budget (RFC 0007).
- **The page cache, 64 KB, is in PSRAM,** where the heap of `full`, the
  T-Deck's configuration, lives.
- **A size limit,** by default **1 MB** of the system partition. The
  partition is 2 MB on the T-Deck Max. It holds the settings, Time's
  state and the log, which takes up to about 340 KB, and littlefs needs
  free blocks of its own. The limit is the device's fact, so it belongs
  in Board's description ([RFC 0024](0024-board.md)); until the schema
  has it, it is a firmware option.
- **Making room.**
  - When a new message would pass the limit, Messages deletes the
    oldest read messages first, by the order they were stored. An
    outgoing message counts as read once it is sent, failed, or not
    known to be sent.
  - The limit counts the database's pages in use. Free pages are given
    back (`auto_vacuum = INCREMENTAL`, set before the first table), with
    the batched writes, never in a delivery: it is a write of seconds
    too.
  - Unread and queued messages are never deleted to make room. If
    nothing else is left, a delivery is refused, the topic says the
    store is full, and the transport keeps the message.

### The data program

Messages is the first service of the data program, which RFC 0007 gives
Messages, Contacts, Packages and Storage. The program starts once the
system program is ready. Messages asks Service states who calls, and
keeps its log level in Settings ([RFC 0026](0026-log.md)).

### Starting values

| What | Value |
|---|---|
| a message's text | 1530 bytes: ten SMS parts of plain text |
| a peer, a sender | 32 bytes |
| a transport's name | 64 bytes, as an app's id |
| a key | 64 bytes |
| a failure's reason | 64 bytes |
| the start of the newest text, in a conversation | 48 bytes |
| conversations in a page | 8 |
| messages in a page | 2, since a message is up to 1.5 KB and an answer at most 4080 bytes ([RFC 0023](0023-service-library.md)) |
| the store's limit | 1 MB on the T-Deck Max |
| statuses kept in memory before written | a minute |

### The order of the work

1. **The data program and Messages:**
   - the store, and a delivery's time measured on the phone;
   - transports registering, delivering and taking their queue;
   - reading, and the topic.

   It is tested with a test transport, and with a `messages` command for
   the shell (`list`, `show`, `send`), like `settings` (pnut-os #40).
2. **Telephony's SMS transport,** in Telephony's RFC, 0030.
3. **The system UI's screens,** and Notifications, when they come.

## Rules it follows

- **One owner.** Messages alone opens `messages.db` (RFC 0004). Others
  reach the messages through its interface.
- **Transports are optional.** A device without a modem has no `sms`;
  Messages works with whatever transports exist.
- **Stored before answered,** so nothing is lost between a transport and
  the store. Statuses after that are not, by choice (above).
- **Topics carry nothing private.**
- **Where it bends a rule:**
  - SQLite allocates memory in every statement, where
    [RFC 0023](0023-service-library.md) has a service allocate only at
    start. Its allocations go to the PSRAM heap, and its page cache is
    sized once.
  - The database's connection is touched by the workers, where RFC 0004
    keeps a module's state on its loop. Each statement runs on one
    worker at a time, behind a lock, and only its result comes back to
    the loop.

## Alternatives

- **The memory card,** as the previous system did: six to ten times
  faster writes. RFC 0010 keeps services' state on internal flash, so that
  messages work without a card. A card can be removed, and the store
  would then be gone until it returns.
- **Files instead of SQLite:** a file for each conversation, appended to.
  Possibly faster on littlefs, but not measured. Paging, counting and
  deleting old messages would be Messages' own code. The previous
  system's first release did this and moved to SQLite.
- **Another database.** FlashDB was measured in the previous system:
  faster writes on littlefs (about 0.4 s), but slow reads of many
  records, and its whole size reserved up front.
- **Each transport keeps its own messages.** The system UI would merge
  them, and every transport would build a store. RFC 0007 chose one
  store already.
- **No limit,** filling the system partition. Then a full store would
  also stop the settings and the log from being written.

## Costs and risks

- **Flash for code.** SQLite added 377 KB to the previous system's
  firmware. Today's firmware is 1.3 MB of a 4 MB slot (2026-10-29).
- **RAM.** SQLite held 184 KB of the heap with one database open in the
  previous system, in PSRAM.
- **Speed.** Seconds per delivery, as above. A burst of SMS after a day
  out of reach is stored one after another, and stays in the modem
  meanwhile.
- **Flash wear.** Each transaction rewrites several 4 KB blocks. littlefs
  spreads them over the partition. A hundred messages a day is far from
  its limits, but that is why statuses are batched.
- **Privacy at rest.** Internal flash is not encrypted by default
  ([RFC 0014](0014-security.md)). A copy of the flash holds the texts.
- **To be measured** when it is built:
  - a delivery's time, with the journal and the batching above;
  - SQLite's size in this firmware;
  - the space one message takes in the store.

## Open questions

- **Where the store lives:** internal flash, at seconds per write, as
  proposed; the memory card when one is in, with internal flash as a
  fallback, as the previous system did, which changes RFC 0010; or files
  instead of SQLite.
- **The store's limit:** 1 MB of the T-Deck's 2 MB system partition, as
  proposed; or a bigger system partition while the package store is
  still empty, since resizing it later means moving data: in practice a
  factory reset (RFC 0010).
- **Starting an app to send:** how Messages has the runtime start an app
  whose transport has queued messages, which RFC 0008 does not provide
  yet.
- **MMS and attachments:** later, in an RFC of their own.
