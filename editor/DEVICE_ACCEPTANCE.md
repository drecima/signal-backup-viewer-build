# SignalBackupEditor v1.0 device acceptance

## Safety preparation

1. Keep the original `SignalBackups` directory unchanged and separately
   backed up.
2. Make a complete copy of the directory, including metadata, the encrypted
   archive, and every media file.
3. Run SignalBackupEditor only against that copy. The editor may replace files
   inside the supplied copy.
4. Use the recovery key only through the editor's interactive secret input or
   other documented protected input. Do not put it in a shell command, script,
   screenshot, log, issue, or repository.
5. Keep the accepted Viewer v0.3.2 IPA and use a fresh LiveContainer data folder
   for each import comparison.

## Automated pre-device gates

The v1.0 test fixture must be generated from a copied backup and include:

- a no-op round trip;
- one new incoming text message;
- one new outgoing text message;
- one new Note to Self text message;
- one new text message with an attachment;
- one Unicode body containing characters outside the BMP;
- one styled range and one mention using verified UTF-16 offsets;
- one timestamp collision that is expected to fail;
- one invalid recipient/chat reference that is expected to fail;
- one deliberately corrupted HMAC that is expected to fail before writing.

Negative-construction tests must fail before replacing the copied backup. At
minimum, cover:

- an incoming message authored by Self;
- a Note to Self message authored by a non-Self recipient;
- a normal quote with neither text nor attachments;
- a link preview whose URL is absent from the message body;
- a direct story reply placed in a group chat;
- a reaction with an invalid author or timestamp;
- a revision whose direction details differ from its parent;
- an invalid attachment client UUID;
- a poll with an empty question or invalid repeated vote;
- a pin update with an unresolved author or target timestamp.

Before packaging the fixture, verify:

- input and output HMACs independently;
- fresh main-archive IV/nonces on successful rewrites;
- complete protobuf parse with no dropped frames or unknown fields;
- frame ordering and all reference-graph invariants;
- attachment ciphertext, keys, hashes and plaintext sizes;
- atomic-failure behavior by interrupting the operation before replacement;
- absence of the recovery key and plaintext message/attachment data from logs
  and temporary directories.

Record only non-sensitive hashes, versions, operation names, selectors and
expected outcomes in the test manifest.

## Viewer test

Import the unedited copied backup into a fresh Viewer v0.3.2 container first and
record the baseline. Then import the edited test copy into another fresh viewer
container.

Confirm:

- all pre-existing chats and media remain present;
- all four constructed messages appear once in the intended chat and position;
- incoming/outgoing/Note to Self presentation is correct;
- author names, self mentions, group events and quoted authors are not Unknown;
- Unicode text, styles and mentions render at the intended ranges;
- attachment type, payload, dimensions, caption, filename, playback and
  thumbnail behavior match the manifest;
- delivery/read state and timestamps match the chosen constructor;
- no mutation controls appear in the viewer;
- relaunching preserves the imported archive.

Copy the `sbe1:<dateSent>` identifier for every constructed message and verify
that each resolves exactly once in the editor.

## Unmodified Signal test

Only after the viewer test passes:

1. Use a disposable Signal installation/account state compatible with the
   pinned backup version.
2. Restore the edited copied backup through an unmodified Signal build.
3. Confirm the same content and reference behavior as in the viewer.
4. Allow attachment restoration to finish before judging missing media.
5. Relaunch and re-check the constructed messages, replies, reactions and
   attachment playback.
6. Export a new backup from the restored state and inspect it with the editor.
   The new messages must survive a second export/import cycle without
   structural drift.

## Acceptance result

v1.0 passes only if all automated gates, the fresh-viewer import, the
unmodified-Signal restore, and the second-generation export/import succeed.
Any generic Signal restore error requires retaining the failed copied fixture
and collecting logs before changing code.
