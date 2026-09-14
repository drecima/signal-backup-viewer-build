# SignalBackupEditor v1.0 roadmap

## Recovered project state

The persistent release history contains packaged Linux x86_64 editors from
v0.5.0 through v0.12.1. It also contains versioned acceptance-test backups for
v0.8.0, v0.8.1, v0.9.0, v0.10.0, v0.11.0, v0.12.0, and v0.12.1.

The current GitHub repositories do not contain the editor source, tests,
changelog, or prior roadmap. Before changing behavior, import and verify the
exact source bundled with v0.12.1. Do not reconstruct an older editor from
memory or silently replace already-tested behavior.

## Release reconciliation

These milestones were represented by packaged editor and acceptance-test
artifacts and should be treated as implemented but requiring source and test
audit:

| Release | Intended scope | Status |
|---|---|---|
| v0.8 | Attachment captions/metadata, reactions, replies and quotes | Packaged; audit exact v0.12.1 source |
| v0.9 | Link previews, timestamps, delivery/read states and reference repair | Packaged; audit exact v0.12.1 source |
| v0.10 | Sender/chat reassignment, styled text, revisions and expiration | Packaged; audit exact v0.12.1 source |
| v0.11 | Attachment add/remove and common group/system events | Packaged; audit exact v0.12.1 source |
| v0.12 | Polls, contacts, stickers and specialized message types | Packaged; v0.12.1 acceptance fixture exists |
| v1.0 | Safe arbitrary message construction | Next implementation milestone |

Artifact existence is not equivalent to complete device acceptance. The
v0.12.1 source and tests must be recovered before the table is converted into
a per-command verified coverage matrix.

## Phase 0 - preserve known-good baselines

- Preserve Viewer v0.3.2 at
  `archive/viewer-v0.3.2-device-accepted`.
- Preserve the v0.12.1 editor ZIP and v0.12.1 acceptance backup byte-for-byte.
- Record SHA-256 values after materializing the packages.
- Import the exact v0.12.1 editor source, tests, README, and changelog into an
  editor-specific repository or an `editor/` tree.
- Confirm that the imported source reproducibly creates the preserved v0.12.1
  binary before development continues.

## Phase 1 - v0.12.1 correctness and safety audit

The existing editor should satisfy all of these gates before arbitrary message
construction is added:

- A no-op operation preserves all decoded protobuf field values, unknown fields,
  frame ordering, attachment bytes, and attachment-to-frame relationships.
- Re-encryption uses a fresh random IV and produces a valid HMAC.
- Input authentication completes before any mutation is written.
- Every affected recipient, chat, author, timestamp, quote, poll, pin, reaction,
  revision, attachment, and group reference is validated.
- Ambiguous selectors, duplicate `dateSent` values, integer overflow, invalid
  UTF-16 body ranges, malformed UUID/ACI data, and unsupported message variants
  fail closed.
- A failed edit does not leave a partially rewritten archive.
- The recovery key is read interactively or through a protected file descriptor;
  it is not accepted in ordinary command-line arguments, printed, logged, stored
  in reports, or committed.
- Decrypted frames and attachment plaintext are not persisted as temporary
  files.
- Tests cover both format version 0.11 and 0.12 inputs seen in the existing
  acceptance history.
- A final encrypted output is checked with libsignal's official bulk message
  backup validator when a compatible binding or helper is available. Treat
  unknown fields as a separately reported soft error and structural failures
  as hard errors.
- A canonical libsignal comparison is used only as an additional semantic
  round-trip check. It is not value-preserving and cannot replace protobuf-level
  preservation tests.

## Phase 2 - schema coverage audit

Use the exact Signal iOS source commit targeted by the viewer and editor:
`f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5`
(`8.29.0.1861-beta`).

Compare the editor's parser, validator, selector model, mutation commands, and
test fixtures against every field in:

- `SignalServiceKit/Protos/Backups/Backup.proto`
- `SignalServiceKit/Protos/Backups/LocalBackup.proto`
- Signal's encrypted proto stream implementation

The initial inventory is in [FORMAT_COVERAGE.md](FORMAT_COVERAGE.md). Convert
each row to Supported, Preserved opaque, Refused safely, or Missing after the
v0.12.1 source is imported. Unknown protobuf fields must be preserved even when
the editor cannot interpret them.

## Phase 3 - v1.0 arbitrary message construction

v1.0 should construct new messages without requiring the user to duplicate an
existing simple message manually.

Required construction inputs:

- destination chat selector;
- author/sender selector;
- incoming, outgoing, or directionless variant;
- collision-free `dateSent`;
- ordinary text or a supported specialized message template;
- optional quote, body ranges, reactions, attachments, link previews, expiry,
  pin details, and delivery state.

Required invariants:

- referenced Recipient and Chat frames precede the new ChatItem;
- new ChatItems are inserted in global rendering order;
- author and directional details are compatible with the destination;
- outgoing status entries reference valid recipients;
- quotes, poll termination, pin updates, revisions, and reactions point to
  valid records;
- body ranges use UTF-16 code units and valid ACIs;
- attachment identifiers, local keys, sizes, hashes, flags, and file records
  remain internally consistent;
- timestamp collisions are rejected or resolved only through an explicit user
  option;
- the final encrypted archive passes the same full validation as an unedited
  archive.

Start with four templates:

1. incoming text message;
2. outgoing text message;
3. Note to Self text message;
4. text message with one existing or newly supplied attachment.

Add specialized constructors only after those four pass end-to-end restoration.

## Input-file policy

The editor may rewrite the path supplied by the user. It does not need to
create a second output location automatically.

The user-facing documentation and startup warning must state:

> Never run SignalBackupEditor on your only or original backup. Make a complete
> copy of the entire backup directory first, keep the original unchanged, and
> run the editor only on the copy.

For scripting, an explicit `--in-place` acknowledgement is preferable to a
mandatory output path. If the current v0.12.1 interface already overwrites its
input, preserve that workflow and add the warning rather than introducing an
automatic copy.

Before replacement, write to a temporary file in the same directory, flush and
close it, validate the completed encrypted archive, then atomically replace the
input archive. Preserve the original file if any stage fails.

## Phase 4 - device acceptance

[DEVICE_ACCEPTANCE.md](DEVICE_ACCEPTANCE.md) defines the final test. Automated
tests prepare the fixture and perform structural/cryptographic validation.
A human device check is still required for Signal rendering and restore
behavior.

## v1.0 definition of done

- The exact v0.12.1 baseline is reproducible.
- The v0.12.1 command set passes the correctness and schema coverage audit.
- The four initial arbitrary-message constructors pass unit, integration, and
  encrypted round-trip tests.
- No plaintext backup, attachment, or recovery key is persisted or logged.
- The edited copy imports into a fresh Viewer v0.3.2 container.
- The edited copy passes restoration through an unmodified compatible Signal
  build.
- Documentation lists every supported, preserved, refused, and missing schema
  field.
- Backup merging and external-history import remain out of scope.
