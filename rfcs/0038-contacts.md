# RFC 0038: Contacts

- **Status:** Accepted
- **Author:** Mateusz Pianka
- **Created:** 2026-11-18
- **Last changed:** 2026-11-18
- **Supersedes / superseded by:** —

## Summary

The Contacts service keeps the owner's address book and the call history,
in the data program, as [RFC 0007](0007-services.md) names it: one small
file for each contact, matched to phone numbers by their last nine digits,
and the calls of the last 90 days. The Phone and Contacts app is built
with it, on the system UI's templates, and Messages then names its
conversations by contact.

## Problem

- **Numbers have no names.** Messages shows each conversation's peer as it
  came, a number ([RFC 0029](0029-messages.md)); an incoming call will
  show only a number too ([RFC 0030](0030-telephony.md)).
- **Nothing keeps the calls.** RFC 0030 gives the call history to
  Contacts, and Contacts does not exist; until it does, a call leaves no
  trace.
- **Phone and Contacts are the next apps** in [RFC 0037](0037-apps-and-the-system-ui.md)'s
  order, "the Contacts service first", and no RFC says how the service
  keeps or serves contacts.
- **The design boards decide what the owner sees** (Phone-Spec,
  Phone-Contacts, Phone-Contact, Phone-Contact-LoRa, Phone-Edit,
  Phone-MyCard, Phone-Recent): this RFC gives them a service, and fixes
  what the boards leave to the system.

## Proposal

### What the owner sees

From the boards, unchanged:

- **One list for Phone and Messages.** A contact holds phone numbers,
  email addresses and LoRa nodes, and both apps use them. A contact with
  only a LoRa node cannot be called: its primary action is Message, and
  Call is offered nowhere.
- **Show as** is what lists, banners and the sleep screen show. It is
  vCard's `NICKNAME`, so other apps keep it once contacts sync.
- **Favourites** sit on top of Contacts and fill Phone's shortcut slots.
  Nothing else reorders the list: it is by Show as, in the language's
  order ([RFC 0017](0017-i18n.md)).
- **No photos:** every contact shows its initials.
- **Unknown numbers** show grouped as the boards show them
  (`+48 22 123 45 67`), and offer **Add to contacts** in Options.
- **Two contacts with one number** are offered for merging, never merged
  silently.
- **In case of emergency (ICE):** up to 4 contacts, each with a relation
  (mother, brother…), kept on **My card**, never as a mark on the contact
  itself: the Emergency contact switch in Edit writes to My card. My card
  is an ordinary contact, and which contact is "me" is a setting. Its
  **Show when locked** switch decides whether the emergency screen shows
  them. They are vCard 4.0's `RELATED;TYPE=emergency`, pointing to the
  contact's UID (RFC 6350 §6.6.6), with the relation as a second `TYPE`.
- **The call history stays on the phone,** never synced.
- **Without the modem** Phone leaves the menu, and Contacts still works:
  its LoRa nodes and email addresses, and the numbers kept for syncing.

### What a contact holds

| Field | Limit |
|---|---|
| First name, last name | 64 bytes each |
| Show as | 64 bytes |
| Phone numbers, each with a kind (mobile, home, work, other) and one the default | 8, of 32 bytes, as typed |
| Email addresses, each with a kind | 4, of 128 bytes |
| LoRa nodes: the mesh (MeshCore, Meshtastic) and the node's id there | 4, of 64 bytes |
| Favourite | yes or no |
| On My card only: emergency contacts, each a contact and a relation | 4 |
| Its UID (vCard's, made when it is created) | 48 bytes |
| Its revision (vCard's `REV`), the time of the last change | — |

A contact is at most about 2 KB encoded, and fits one message of the
service library's 4 KB ([RFC 0023](0023-service-library.md)). Up to
**1000 contacts.** Fields the boards do not show (birthday, company,
notes) are not kept: when CardDAV comes (below, Later), the fields a sync
brings that the phone does not show are kept for it, in a field of their
own.

### Matching a number

- **A number is stored as typed** ("+48 601 234 567", "601234567").
- **It matches by its last 9 digits,** after dropping everything but the
  digits and an international prefix (`+` or `00`) followed by the home
  country's code. The home country is the SIM's, or the network's when
  the SIM does not say (pnut-labs/pnut-os#120). So `+48 601 234 567` and
  `601234567` are one number; `+44 7700 900123` and `07700 900123` are
  one in the United Kingdom.
- **A number of fewer than 9 digits** (`112`, a short code) matches only
  the same digits.
- **Full E.164,** with each country's numbering plan, waits for a
  libphonenumber-like table; the key above can be made again from the
  numbers as typed then, since they are kept.
- **Several contacts with one number:** Match gives the first, by Show
  as, and says that there are more, which the Phone and Contacts app
  shows as the offer to merge.

### The call history

- **The calls of the last 90 days,** however many, newest first: each
  call's number as it came, the contact matched then (if any), its
  direction, whether it was answered, missed or declined, when it began,
  and how long it lasted.
- **Contacts keeps it itself,** from Telephony's `call` topic and
  `CallInfo` (RFC 0030), so no app needs to run for a call to be kept.
- **A call is dropped** once it is older than 90 days, by the device's
  clock, and only once Time has set the clock ([RFC 0028](0028-time.md)):
  a clock still wrong after a restart drops nothing.
- **A cap, as a guard:** at most 5000 calls (about 300 KB); past it, the
  oldest go first, even within 90 days. That is about 55 calls a day for
  90 days.
- **The owner can remove one call** (Recent's X), or all of them.

### The interface

`pnut.Contacts`, in the data program, with its settings' owner
`contacts`:

| Method | What it does |
|---|---|
| **List** | Contacts a page at a time, by Show as, favourites first; My card apart, at the head. A cursor, so a page asked again after a change does not skip or repeat. |
| **Get** | One contact, whole. |
| **Put** | A contact, whole: a new one (given its id), or one replaced. It carries the revision it was made from, and is refused if the contact changed since. |
| **Edit** | Single fields of a contact (Show as, a number added, Favourite), named in a field mask, the rest untouched; for the Emergency contact switch, which changes My card from a contact's screen, and for Messages' Add to contacts. |
| **Delete** | A contact. It also leaves My card's emergency contacts. |
| **Find** | Contacts whose Show as or names hold a text, a page at a time; for Contacts' Sym and for Messages' To. |
| **Match** | A number to a contact: its id and Show as, and whether more contacts share the number. |
| **Calls** | The call history, a page at a time, newest first. |
| **Forget call** | One call, or all of them. |

- **A topic, `contacts`,** counts changes: screens read again when it
  moves. Like Messages' topic, it says nothing about who changed, since
  any native code can read a topic (RFC 0005). A second count moves when
  the call history changes.
- **My card** is the setting `contacts.me` ([RFC 0025](0025-settings.md)),
  the id of the contact that is the owner's.
- **Ids** are 32-bit counters, never given again (as Messages' are), so
  that a call kept, a conversation named, or an emergency contact on My
  card never points to a contact made after its own was deleted.

### Who may

- **Read everything:** the system UI (banners, the incoming call, the
  sleep screen's emergency contacts), the Phone and Contacts app, and
  Messages: the built-in apps that need the whole list.
- **Write:** the Phone and Contacts app and the system UI. Messages adds
  a contact through the Phone and Contacts app (below), not itself.
- **Other apps** see no contact, only the one the owner picks through the
  intent **pick a contact** ([RFC 0008](0008-runtime.md)), which the
  Phone and Contacts app handles and answers with that contact.
- **A command typed in the developer's shell** may do what the system UI
  may (RFC 0025's rule); a shell command, `contacts`, lists, shows, finds
  and matches, for tests.
- **Nothing logs a name or a number** above debug, as for messages.

### Where it keeps them

In `/var/lib/contacts/`, on the internal flash ([RFC 0010](0010-file-layout.md)),
never on the card:

- **One file for each contact,** encoded as the interface's own message,
  written whole beside the old one and renamed over it, as
  [RFC 0011](0011-storage.md)'s whole files are. A change rewrites only
  that file, a few hundred bytes: littlefs rewrites a file whole, which
  is what made SQLite slow here (measured for Messages on 2026-10-30, RFC
  0029).
- **In 32 folders** by the id's last bits, so that no folder holds more
  than about 32 files at 1000 contacts: littlefs finds a file in a folder
  by walking it.
- **An index for each folder:** its contacts' ids, Show as, names, the
  keys of their numbers and the favourite mark, rewritten with the
  contact (under 8 KB). Read at start, they give List and Match from
  memory, and Find from the indexes' names, without opening a contact's
  file. A missing or damaged index is made again from its folder's
  files.
- **The next id,** in a small file, as Messages keeps its own.
- **The calls,** in a file for each month, records appended; a month
  older than 90 days goes whole, and the days inside the oldest month
  left are passed over when reading.

### The Phone and Contacts app

One app, `org.pnut.phone`, behind Menu's Phone and Contacts tiles; the
title switches its views (Dial, Recent, Contacts), as Phone-Spec says.
On RFC 0037's templates:

| Screen | Template |
|---|---|
| Contacts: My card, Favourites, then letter groups | List, with views |
| Recent | List |
| A contact; My card | Detail, with options (Message ▸ via…, Edit, Share, Delete) |
| Edit contact (2 pages) | Form |
| A number's kind, a relation | Choice |
| Dial: the digits typed, matching contacts below | List with a live field (RFC 0037's dialler) |

- **The keys** are Phone-Spec's: Sym finds, Space pages by letter group,
  Enter calls the default number, E opens; in Recent, Enter calls back, E
  opens, X clears one entry.
- **Calls** (dialling, the call screen, a second call, the incoming call)
  are Telephony's and their own work: this RFC gives them names, and
  Recent.
- **Pick a contact** and **Add to contacts** need RFC 0037's intents and
  chain (its step 4): the part these need, the system UI dispatching an
  intent from Messages to this app and its answer coming back, is built
  with this app.

### Messages, by contact

Once Contacts exists, Messages (RFC 0029's app):

- **Names conversations:** Show as and initials instead of the number;
  an unknown number grouped as above.
- **One conversation per contact:** the conversations of a contact's
  numbers (and its LoRa nodes, once those have transports) show as one,
  their messages in the order they were stored, which ids keep across
  conversations (RFC 0029). The store does not change: it keeps a
  conversation per transport and peer, and the app joins them.
- **A reply goes to the number last written from;** the composer shows
  which number, and Options changes it for the next message.
- **A new message's recipient:** typing in To finds contacts as it is
  typed (Messages reads Contacts), or takes a number; a key opens the
  Phone and Contacts app's **pick a contact** to browse.
- **Add to contacts** in an unknown number's options opens this app's
  Edit, the number filled in, through the chain; back returns to the
  thread.
- **A contact's notification** ([RFC 0020](0020-notifications.md)) shows
  Show as too.

### Later

- **CardDAV** sync on the same accounts as Calendar and Reminders, vCard
  3.0 and 4.0, and a vCard import and export, come with the accounts
  (pnut-labs/pnut-os#99).
- **The sleep screen's emergency pages** come with the lock screen.
- **Full E.164 matching,** as above.

### The order of the work

Notifications ([RFC 0020](0020-notifications.md), with RFC 0037's step 5
for apps) come before this, by the owner's choice: an incoming message
alerts nobody today. Then:

1. **The service with the Phone and Contacts app,** together: the
   `.proto`, the files and indexes, Match, the call history from
   Telephony, the `contacts` command; the app's Contacts, Recent,
   contact, Edit and My card on the templates, and the part of the
   intents and the chain that pick a contact and Add to contacts need.
2. **Messages by contact:** names, one conversation per contact, To's
   search and the picker, Add to contacts.
3. **Dial and calls** with Telephony's call work (RFC 0030's step 3).

## Rules it follows

- **Its place:** a service of the data program (RFC 0007), beside
  Messages, on the service library (RFC 0023), its files under
  `/var/lib/contacts/` (RFC 0010).
- **It bends RFC 0011's line** that "services such as Messages and
  Contacts link SQLite directly": as Messages did (RFC 0029), for the
  same measured reason, a store of small files.
- **It bends RFC 0037's order:** notifications come before Phone and
  Contacts, and the part of step 4's intents that picking and adding a
  contact need moves into step 3. RFC 0037 is changed to say so.
- **Permissions:** contacts are personal data; only the built-in apps
  that need them read them whole, and other apps get one contact, the
  owner's pick (RFC 0008's intent).
- **Calls are kept by the service, not an app,** as RFC 0037 wants of
  every event that must not wait for an app.

## Alternatives

- **vCard files as the store:** CardDAV and export would be a copy, but
  every List and Match parses text, and the fields the phone does not
  show must survive its edits. Kept for sync, not storage.
- **SQLite:** rejected for Messages after measuring it on littlefs (a
  commit rewrites nearly the whole file, 12 s at 232 KB); a thousand
  contacts would be about that size.
- **One file for all the contacts:** every change rewrites all of them.
- **Contacts in Messages' store:** they would share a program already, but
  Phone, the system UI and the call history need them without Messages,
  and RFC 0007 makes them a service of their own.
- **The last 500 calls, by count:** the owner chose the last 90 days; the
  cap above only guards against a flood.
- **One conversation per number:** the owner chose one per contact.

## Costs and risks

- **Flash:** a contact's file is a few hundred bytes (littlefs keeps a
  small file inside its folder's metadata); 32 indexes of a few KB; the
  calls up to about 300 KB. Under 1 MB at the limits.
- **Memory:** List and Match work from what the indexes give at start,
  kept in the data program's memory (Show as, the numbers' keys, the
  favourite mark): about 100 bytes a contact, 100 KB at 1000, in PSRAM.
  Find reads the names from the indexes on flash, about 250 KB at 1000.
  To measure, with the data program's other stores
  (pnut-labs/pnut-os#143).
- **Start:** reading 32 indexes, not 1000 files. To measure on the T-Deck
  Max, with 1000 made-up contacts.
- **Matching by 9 digits** can join two numbers that differ only in their
  country code or a longer prefix. Rare for one owner's contacts; the
  number as typed is always shown, and full E.164 can replace it.
- **A contact edited in two places** (Phone, then Messages' Add to
  contacts on the same contact) is refused by its revision, not lost.
- **The call history** depends on Telephony's `call` topic: a call that
  starts and ends while the data program restarts is lost. Telephony
  keeps no history of its own.

## Open questions

- **A LoRa node's id** on each mesh (MeshCore's public key or its prefix,
  Meshtastic's node number) is fixed when the mesh stacks are built; the
  field holds either, with its mesh.
- **Relations:** the boards show mother, brother and "emergency contact";
  the list is vCard's `RELATED` types (parent, child, sibling, spouse,
  friend, co-worker, contact) plus "other", unless the owner wants others.
