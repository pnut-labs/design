# RFC 0029: Messages

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-10-30
- **Last changed:** 2026-10-30
- **Supersedes / superseded by:** —

## Summary

Messages is the device's one store of text messages: SMS, and the
messages of mesh apps and other messaging apps. The transports carry
them: Telephony carries SMS, and an app carries its own network's
messages. Messages keeps the conversations, queues what is to be sent,
and tells the system UI what changed. It runs in the data program. It
keeps the messages in files on internal flash, appended to: SQLite
there, which this RFC first proposed, takes seconds per write, and
longer as the store grows (measured below).

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
    the send's one write. A message that had to wait is written as
    sending before the transport gets it: a second write, only for
    messages that waited. So no message is handed over twice, even
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

- **Files on internal flash, appended to:** a log for each conversation,
  in `/var/lib/messages/`, in the system partition
  ([RFC 0010](0010-file-layout.md)). A log's first record names its
  transport and peer.
- **Not SQLite,** which this RFC first proposed. It was measured on the
  T-Deck Max on 2026-10-30, with SQLite 3.45.1, a delivery as one
  transaction, the journal truncated and free pages given back:

  | The store | One delivery |
  |---|---|
  | nearly empty, 40 KB | 3.7 s |
  | 405 messages, 136 KB | 8.3 s |
  | 808 messages, 232 KB | 11.5–13.7 s |
  | at the limit, 1 MB, extrapolated | about 35–50 s |

  littlefs never writes in place: when a file changed in its middle is
  synced, littlefs copies everything after the change into new blocks.
  SQLite rewrites the database's first page at each commit, so each
  commit rewrites about the whole file, and wears the flash as much.
  Other journal settings took 2–5 s on the empty store, and grow the
  same way.
- **Appending is fast.** Adding a record to a file took 0.07–0.24 s with
  its sync, whatever the file's size. Reading runs at about 1.2 MB/s,
  and one record at a known place takes 6–8 ms (measured on 2026-10-30).
- **Records.** Each is its length, the record (protobuf, from Messages'
  `.proto`) and a checksum. There are four kinds:
  - a message, with all it has (above);
  - a status: sending, sent, delivered or failed, with a failure's
    reason;
  - read: the conversation is read up to a message;
  - deleted: a message is.
- **Every write is on flash before Messages answers,** in about a tenth
  of a second:
  - a delivery or a send appends its message, and a new conversation is
    a new file;
  - statuses and marks of read are written as they come, not gathered,
    so a reset loses none. A message its transport had taken when the
    device reset shows as "not known to be sent".
- **littlefs commits an append whole when it syncs.** A record cut
  short, as by a crash between a write and its sync, fails its checksum
  when the logs are read at start, and its file is cut back to the last
  whole record.
- **In memory, built at start.** Messages reads every log at start, on a
  worker: under a second for a full store. It keeps, in PSRAM, about
  100 KB at the limit:
  - each conversation's counts, the start of its newest text, and its
    file's size;
  - each message's place in its file, and its status;
  - the keys of delivered messages, so that a message delivered twice is
    stored once;
  - the next ids: a message's and a conversation's are 64-bit counters.
- **Reading.** The conversations come from memory. A page of messages is
  read at the places memory gives: about 15 ms.
- **Deleting.** A deleted message gets its record, and its space comes
  back when its conversation is rewritten (below). A deleted
  conversation's file is removed.
- **All file work runs on the program's workers, one job at a time:** a
  write takes a tenth of a second, over a handler's budget (RFC 0007),
  and a rewrite takes seconds. The loop keeps the memory, and gives each
  job copies of what it needs ([RFC 0023](0023-service-library.md)).
- **A size limit,** by default **1 MB** of the system partition. The
  partition is 2 MB on the T-Deck Max. It holds the settings, Time's
  state and the log, which takes up to about 340 KB, and littlefs needs
  free blocks of its own. The limit is the device's fact, so it belongs
  in Board's description ([RFC 0024](0024-board.md)); until the schema
  has it, it is a firmware option.
- **Making room.**
  - When the store passes the limit, Messages rewrites conversations
    without their oldest read messages, by the order they were stored.
    An outgoing message counts as read once it is sent, failed, or not
    known to be sent.
  - A conversation is rewritten into a new file, which is then renamed
    over the old one: littlefs renames atomically, so a reset leaves one
    or the other. Statuses are folded into their messages, and deleted
    messages left out. Rewriting 200 KB takes about 3 s; it runs after
    the delivery that passed the limit, never before Messages answers.
  - A conversation whose file is mostly deleted messages is rewritten
    the same way.
  - Unread and queued messages are never deleted to make room. If
    nothing else is left, a delivery is refused, the topic says the
    store is full, and the transport keeps the message.
- **The store has an interface of its own inside Messages,** so that
  another engine can replace the files later, such as SQLite where it
  writes fast, without a change to what transports and the system UI
  see.

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

- **One owner.** Messages alone opens its files (RFC 0004). Others reach
  the messages through its interface.
- **Transports are optional.** A device without a modem has no `sms`;
  Messages works with whatever transports exist.
- **Stored before answered,** so nothing is lost between a transport and
  the store; statuses and marks of read too.
- **Topics carry nothing private.**
- **Where it bends a rule:** the memory that indexes the store grows
  with it, where [RFC 0023](0023-service-library.md) has a service
  allocate only at start. It is allocated in blocks in PSRAM as the
  store grows, up to what the limit needs, and given back as it shrinks.

## Alternatives

- **SQLite on internal flash,** this RFC's first proposal: seconds per
  delivery, and longer as the store grows (measured above).
- **SQLite with each page in a small file of its own,** through a
  storage layer of its own for SQLite. It keeps SQL, and its time would
  not grow with the store: about 1–2.5 s per delivery, estimated from
  4 KB writes measured at 0.09–0.38 s. Some hundreds of files in a
  directory have not been measured.
- **SQLite on another file system,** in the package store, unused so
  far: NuttX's SmartFS, or FAT with wear levelling. Not measured; less
  resistant than littlefs to a power cut; and a change to the flash's
  layout (RFC 0010).
- **The memory card,** as the previous system did: six to ten times
  faster writes. RFC 0010 keeps services' state on internal flash, so
  that messages work without a card. A card can be removed, and the
  store would then be gone until it returns.
- **Another database.** FlashDB was measured in the previous system:
  faster writes on littlefs (about 0.4 s), but slow reads of many
  records, and its whole size reserved up front.
- **Each transport keeps its own messages.** The system UI would merge
  them, and every transport would build a store. RFC 0007 chose one
  store already.
- **No limit,** filling the system partition. Then a full store would
  also stop the settings and the log from being written.

## Costs and risks

- **Code of its own** for what SQLite would give: paging, counting,
  finding a message delivered twice, making room, and the records'
  format, which has a version.
- **Flash for code.** Small, with no SQLite in the firmware: it added
  550 KB with its shell (measured on 2026-10-30). Today's firmware is
  1.3 MB of a 4 MB slot.
- **RAM.** The index, about 100 KB at the limit, in PSRAM.
- **Speed.** About a tenth of a second per write. A rewrite to make room
  takes seconds; deliveries wait for it, and stay with their transport
  meanwhile. A full store takes about a second to read at start.
- **Flash wear.** Each write rewrites its file's last block and
  littlefs's entry for it; a rewrite to make room rewrites its
  conversation. A hundred messages a day, with their statuses, rewrite
  each block of the partition about once a day: far from its limits.
- **Privacy at rest.** Internal flash is not encrypted by default
  ([RFC 0014](0014-security.md)). A copy of the flash holds the texts.
- **To be measured** when it is built:
  - a delivery's time through Messages;
  - reading a full store at start;
  - the space one message takes in the store.

## Open questions

- **Another engine, later.** Files are the store for now. Whether SQLite
  or another engine replaces them, on devices or file systems where it
  writes fast, is left open; the store has an interface of its own
  inside Messages for that.
- **The store's limit:** 1 MB of the T-Deck's 2 MB system partition, as
  proposed; or a bigger system partition while the package store is
  still empty, since resizing it later means moving data: in practice a
  factory reset (RFC 0010).
- **Starting an app to send:** how Messages has the runtime start an app
  whose transport has queued messages, which RFC 0008 does not provide
  yet.
- **MMS and attachments:** later, in an RFC of their own.
