# Signal backup format coverage inventory

This inventory is derived from Signal iOS commit
[`f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5`](https://github.com/signalapp/Signal-iOS/tree/f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5),
the source used for Signal iOS `8.29.0.1861-beta`. It is an audit checklist,
not a claim that every field is currently editable.

Authoritative inputs:

- [Backup.proto](https://github.com/signalapp/Signal-iOS/blob/f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5/SignalServiceKit/Protos/Backups/Backup.proto)
- [LocalBackup.proto](https://github.com/signalapp/Signal-iOS/blob/f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5/SignalServiceKit/Protos/Backups/LocalBackup.proto)
- [Encrypted stream provider](https://github.com/signalapp/Signal-iOS/blob/f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5/SignalServiceKit/Backups/Archiving/FileStreams/BackupArchiveProtoStreamProvider.swift)

After importing the v0.12.1 source, assign each area one of four states:
Supported, Preserved opaque, Refused safely, or Missing.

## Container and cryptography

| Area | Fields or invariant | Audit requirement |
|---|---|---|
| Local metadata | version; encrypted backup ID; 12-byte metadata IV | Authenticate/derive correctly and preserve version |
| Main archive | optional nonce header; HMAC; AES encryption; gzip; varint-delimited protobuf frames | Validate HMAC before mutation; fresh IV on rewrite |
| Header | version, backupTimeMs, mediaRootBackupKey, currentAppVersion, firstAppVersion, debugInfo | Preserve unknown and unedited fields |
| Frame order | AccountData first; referenced frames before referrers; ChatItems in global render order | Full validation before replacement |
| Unknown fields | protobuf fields unknown to this editor version | Preserve byte values through parse/serialize and report separately |

Signal's stream order at this commit is chunking, gzip compression, encryption,
HMAC generation, and optional nonce header for output; input reverses the
transforms after validating the HMAC.

Signal iOS also calls libsignal's bulk `validateMessageBackup` API. The
validator makes two stream passes so it can authenticate and parse independently.
Its unknown-field result is separate from hard validation failures. The editor
should use this validator as a release gate when practical, while retaining its
own value-preserving protobuf tests. Libsignal's ComparableBackup JSON is
canonicalized and explicitly not a value-preserving serialization format.

## Top-level frame families

| Frame | Important substructures |
|---|---|
| AccountData | profile/name/avatar, username link, subscriber data, account settings, iOS/Android settings, bio |
| Recipient | Contact, Group, DistributionListItem, Self, ReleaseNotes, CallLink |
| Chat | recipient, archive/pin/mute/unread state, expiration timer, style |
| ChatItem | identity, time, expiry, revisions, direction, message type, pin details |
| StickerPack | pack ID and key |
| AdHocCall | call ID, recipient, state and timestamp |
| NotificationProfile | schedule, allowed members, calls/mentions, name/emoji/color |
| ChatFolder | included/excluded recipients, type, ordering and visibility settings |

The earlier feature roadmap concentrated on ChatItem mutations. AccountData,
recipient profiles, full group snapshots, distribution lists, call links,
notification profiles, chat folders, chat styles, and ad-hoc calls therefore
need explicit coverage classification rather than being silently ignored.

## ChatItem envelope

| Area | Fields |
|---|---|
| Identity | chatId, authorId, dateSent |
| Expiration | expireStartDate, expiresInMs |
| Revisions | ordered nested ChatItems |
| Direction | incoming, outgoing, directionless |
| Incoming | dateReceived, dateServerSent, read, sealedSender |
| Outgoing | dateReceived and repeated per-recipient SendStatus |
| Pinning | pinnedAtTimestamp and expiring/never-expiring choice |
| Legacy | sms |
| Item variant | standard, contact, sticker, remote-delete, update, payment, gift badge, view-once, direct-story reply, poll, admin-delete |

## Standard messages and shared components

| Component | Fields and constraints |
|---|---|
| Text | body and repeated bodyRanges |
| BodyRange | UTF-16 start/length; mention ACI or style |
| Quote | targetSentTimestamp, authorId, quoted text/attachments and quote type |
| Reaction | emoji, authorId, sentTimestamp and sortOrder |
| LinkPreview | URL, optional title/image/description/date |
| Long text | FilePointer rather than inline body |
| Revisions | oldest-to-newest nested ChatItems |
| SendStatus | recipientId, timestamp, pending/sent/delivered/read/viewed/skipped/failed and sealed-sender/failure metadata |

## Attachments

| Component | Fields and constraints |
|---|---|
| MessageAttachment | FilePointer, NONE/VOICE_MESSAGE/BORDERLESS/GIF flag, wasDownloaded, clientUuid |
| FilePointer presentation | contentType, fileName, width, height, caption, blurHash |
| FilePointer integrity | incrementalMac and chunk size |
| LocatorInfo | encryption key, plaintextHash or encryptedDigest, plaintext size |
| Remote locator | transit CDN key/number/timestamp and media-tier CDN number |
| Local locator | localKey, generally required for locally available attachments |
| Quote attachment | contentType, fileName and optional thumbnail MessageAttachment |
| Sticker data | packId, packKey, stickerId, emoji and FilePointer |
| Contact avatar | FilePointer |
| Link-preview image | FilePointer |
| View-once media | optional MessageAttachment |

Duration and waveform are not direct fields of `FilePointer` in the pinned
backup schema. If v0.12.1 exposes them, document where they are encoded or
derived rather than treating them as ordinary FilePointer fields.

## Specialized message variants

| Variant | Substructures requiring coverage |
|---|---|
| ContactMessage | name parts, phones, emails, postal addresses, avatar, organization, reactions |
| StickerMessage | sticker pointer and reactions |
| RemoteDeletedMessage | tombstone with envelope identity/time |
| ViewOnceMessage | optional attachment and reactions |
| DirectStoryReplyMessage | text/long text or emoji, plus reactions |
| Poll | question, multi-select flag, options, votes, ended state and reactions |
| PaymentNotification | amounts, fee, note, transaction status/identifiers/blobs |
| GiftBadge | credential presentation and state |
| AdminDeletedMessage | deleting admin recipient |
| PinMessageUpdate | target timestamp and author |
| PollTerminateUpdate | target timestamp and question |

Payments and gift badges contain sensitive or ledger-related opaque data. They
should initially be preserved or refused, not freely synthesized.

## Updates and calls

The schema includes:

- individual and group calls with state, direction, participant references,
  start/end timestamps and read state;
- simple updates such as identity changes, blocks, session refreshes, message
  requests and unsupported messages;
- expiration, profile-name, learned-profile, thread-merge and session-switch
  updates;
- group creation, title, avatar, description, access-level, announcement,
  membership, invitation, join-request, invite-link, migration, timer, member
  label and termination updates.

Each group update carries a distinct combination of ACI, PNI, count, access
level, timestamp and boolean fields. Do not implement them through one generic
dictionary mutation unless type-specific validation remains enforced.

## Whole-backup uniqueness and completion constraints

Libsignal's model additionally rejects or requires conditions that are not
visible from field types alone:

- AccountData must exist.
- The Self recipient must exist and must be unique.
- If chat folders exist, exactly one ALL folder must exist.
- Contact phone numbers, usernames, ACIs and PNIs must not collide.
- Group master keys, distribution-list IDs and call-link root keys must not
  collide.
- The release-notes recipient must be unique.

These checks belong in the final validation pass after every edit, including
edits that appear unrelated to recipients.

## Reference graph to validate

- Frame recipient IDs from Chat, ChatFolder, NotificationProfile, AdHocCall and
  distribution lists.
- ChatItem chatId and authorId.
- SendStatus recipientId.
- Quote authorId and targetSentTimestamp.
- Reaction authorId.
- Poll voterId.
- AdminDeletedMessage adminId.
- Group-update ACIs/PNIs and group snapshot membership.
- Pin and poll-termination target timestamps.
- Revision ordering and final-message relationship.
- Attachment FilePointer to physical local-media file and key/hash/size data.

The schema audit is complete only when every protobuf field in the pinned files
maps to a documented state and the test suite proves preservation of unknown
fields.

## Signal iOS importer constraints

The pinned Signal iOS importer defines additional semantic failures in
`BackupArchive+Errors.swift`. The editor must reject these combinations before
writing, even when the protobuf wire data is syntactically valid.

| Area | Required construction checks |
|---|---|
| Author and direction | An incoming message cannot be authored by Self; a Note to Self item cannot use a non-Self author; incoming, outgoing and directionless details must match the message kind and destination chat |
| Quotes | Quote authors must resolve; a normal quote must contain quoted text or attachments; quote timestamps must identify a valid target when target linkage is required |
| Link previews | A preview requires a URL, and the URL must occur in the message body; preview-image pointers must satisfy ordinary attachment validation |
| Reactions | Author/address and sent timestamp must be valid; reaction references and ordering must remain internally consistent |
| Revisions | Only supported message types may carry revisions; nested revisions must retain the expected direction details and ordering |
| Story replies | A direct story reply requires a valid ACI, cannot be empty and cannot be constructed in a group thread |
| Polls | Questions and options must be non-empty and within Signal limits; votes must reference valid voters/options and must not violate repeated-vote rules |
| Pins | Target timestamp and author must resolve; pin state must remain within Signal's maximum pinned-message rules |
| Attachments | Client UUIDs and pointer data must be valid; long-text messages require the corresponding long-text attachment representation |
| Content size | Standard messages cannot be empty or exceed importer limits unless represented through the supported long-text path |
| Calls and updates | Call records must use a compatible thread/recipient type; ad-hoc calls require a call-link recipient; group updates cannot be empty |
| Identifiers | ACI, PNI, service ID, E164, profile key and contact-identity data must decode and must not collide with existing records |
| Contacts and chats | Contacts require a usable identifier; a non-Self contact cannot claim local identifiers; custom chat colors and gradients must meet Signal's component/count rules |
| Distribution lists and payments | Distribution-list membership/privacy constraints must hold; payment notifications cannot be synthesized in group chats |

These rules are a minimum list derived from the pinned importer's public error
enum, not a substitute for running Signal's complete importer and libsignal
validator. Every new constructor should have at least one valid fixture and one
fixture for each relevant rejected combination.
