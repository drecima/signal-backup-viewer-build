# SignalBackupEditor v1.0 construction specification

This document defines the first safe arbitrary-message constructors. It targets
Signal iOS commit
[`f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5`](https://github.com/signalapp/Signal-iOS/tree/f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5)
(`8.29.0.1861-beta`). Implementation begins only after the exact v0.12.1
source and tests are recovered and reproduce the preserved binary.

## Safety contract

A constructor is a transaction over a complete copied `SignalBackups`
directory. It must:

1. authenticate and parse the entire input before mutation;
2. resolve every selector to exactly one existing record;
3. construct in memory without changing the supplied files;
4. validate the full reference graph and Signal-specific semantic rules;
5. serialize with all unedited and unknown fields preserved;
6. use fresh cryptographic randomness and generate a valid HMAC;
7. validate the completed encrypted temporary archive;
8. atomically replace the archive in the supplied copy only after every check
   succeeds.

Never run the editor on the only or original backup. The program may update the
path supplied by the user, so the user must first make a complete copy of the
backup directory, including every media file.

## Common constructor inputs

| Input | Rule |
|---|---|
| Destination chat | Required selector; must resolve to exactly one Chat and its Recipient |
| Direction | Incoming or outgoing for the first release; directionless is reserved for type-specific system constructors |
| Author | Must resolve to exactly one Recipient and be compatible with direction and chat |
| `dateSent` | Required millisecond timestamp initially; must not collide with an existing or newly staged message identity |
| Body | Non-empty UTF-8 text within Signal's inline-message limit, otherwise use the established long-text path |
| Received timestamp | Defaults to `dateSent` only when Signal accepts that relationship; explicit override must be nonnegative and valid |
| Expiration | Off by default; start and duration must be supplied and validated together when enabled |
| Read state | Explicit or documented template default for incoming messages |
| Delivery state | Explicit or documented template default for outgoing messages; all status recipients must resolve |
| Optional components | Body ranges, quote, reactions, preview, attachment and pin details go through the existing audited v0.12.1 builders |

Do not infer an author from display text. Use stable recipient IDs, ACIs or an
unambiguous existing selector. Do not silently adjust a colliding timestamp.

## Template 1: incoming text

Required invariants:

- the author is not Self;
- in a one-to-one chat, the author is the chat's contact recipient;
- in a group, the author is a valid member represented by a compatible contact;
- directional details are incoming;
- `dateReceived` is valid and `dateServerSent`, if supplied, is valid;
- outgoing send statuses are absent;
- the StandardMessage contains non-empty inline Text and no unsupported fields.

The initial CLI/API should require the author explicitly for group chats.

## Template 2: outgoing text

Required invariants:

- the author is the unique Self recipient;
- directional details are outgoing;
- `dateReceived` is valid;
- incoming-only fields are absent;
- each SendStatus recipient and timestamp is valid for the destination;
- the StandardMessage contains non-empty inline Text.

The exact default SendStatus set must be derived from Signal's pinned exporter
and the recovered v0.12.1 delivery-state implementation. Until that audit is
complete, require an explicit supported delivery-state policy rather than
inventing group recipient statuses.

## Template 3: Note to Self text

Required invariants:

- the destination Chat points to the unique Self recipient;
- the author is Self;
- the directional form and any status entries match a genuine Note to Self
  record produced by the pinned Signal version;
- the StandardMessage contains non-empty inline Text.

The recovered fixture must provide the canonical envelope. Do not treat an
ordinary outgoing contact message as automatically equivalent to Note to Self.

## Template 4: text with one attachment

This extends one of the three text envelopes above. It must use the existing
audited v0.12.1 attachment pipeline and create a MessageAttachment containing:

- a FilePointer with content type and valid LocatorInfo;
- a local attachment key when the file is available in the local backup;
- plaintext size and the integrity representation required by the existing
  media pipeline;
- a valid, unique client UUID when present;
- a mutually exclusive attachment flag;
- width, height, filename, caption and blur hash only when meaningful and
  validated for the payload.

The media file and archive frame change are one transaction. A failure after
staging either part must leave the copied input unchanged. Duration and waveform
are not direct FilePointer fields in this schema; the source audit must identify
their real storage or derivation before exposing them as constructor options.

## Frame placement

Referenced Recipient and Chat frames already exist and precede the new item.
Insert the ChatItem among all ChatItems in global rendering order, not merely
next to other messages from the same chat. If two items would have the same
ordering identity, fail unless a future explicit collision-resolution option is
selected.

The constructor must not renumber unrelated frame IDs. Any new referenced frame
type added by a later constructor requires its own collision-free ID allocation
and ordering rules.

## Validation gates

Every constructed backup must pass:

- the editor's protobuf value/unknown-field preservation comparison;
- complete reference and uniqueness checks;
- Signal iOS importer constraints catalogued in
  [FORMAT_COVERAGE.md](FORMAT_COVERAGE.md);
- the official compatible libsignal bulk backup validator;
- independent HMAC and archive-decryption verification;
- the automated and device process in
  [DEVICE_ACCEPTANCE.md](DEVICE_ACCEPTANCE.md).

ComparableBackup canonical JSON may be used as an additional semantic
comparison, but never as the serialization source or proof of byte-value
preservation.

## Proposed interface

The final spelling should follow the recovered v0.12.1 command style. The
semantic operation should resemble:

```text
message create text --chat <selector> --direction <incoming|outgoing> \
  --author <selector> --date-sent <milliseconds> --body <protected input>
```

Note to Self should be an explicit template or destination mode. Attachment
creation adds one input file plus audited presentation metadata. An explicit
`--in-place` acknowledgement is recommended because the supplied copied
backup may be replaced; a mandatory second output location is not required.

Secrets and sensitive content should be accepted through interactive or
protected input. Recovery keys must never be accepted as ordinary command-line
arguments.
