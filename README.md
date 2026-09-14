# Signal Backup Viewer v0.3.2

This is an unsigned, experimental iOS build based on Signal iOS
`8.29.0.1861-beta`. It is intended only for importing and inspecting a local
Signal backup in an isolated app container.

## Isolation and read-only changes

- Separate bundle identifier: `org.signalbackup.viewer`.
- Signal notification/watch extensions are removed from the IPA.
- Registration account creation is replaced by a fixed synthetic local
  identity. It does not create or attach a Signal server account.
- Local-backup decryption uses the encrypted backup ID in `metadata`, rather
  than requiring the original account ACI.
- DNS lookup, BSD `connect`/`connectx`, and Network.framework connection starts
  are blocked inside SignalServiceKit.
- Chat activation, background launch repair jobs, and expiration jobs are not
  started.
- The conversation composer, reaction picker, compose/camera buttons, read
  marking, deletion, editing, replying, forwarding, selection, and pin changes
  are disabled or absent.
- Viewing, copying text, message details, media playback, and local media
  export remain available.

## v0.2 registration-bootstrap fix

The first device test showed that v0.1 reached Signal's normal post-registration
profile setup before the backup importer. It then failed with `noIdentityKey`
because the synthetic offline account had never run Signal's account-creation
prekey bootstrap.

v0.2 now generates and persists local ACI/PNI identity and prekey material using
Signal's existing registration key generator, while compile-time viewer guards
skip the restricted websocket and one-time-prekey upload stages. No generated
key material is transmitted.

The iOS Files provider is a separate operating-system process. If a selected
backup is stored in iCloud, iOS may download its encrypted files before handing
them to the viewer. The viewer's own network paths remain blocked.

## v0.3 identity and read-only hardening

v0.3 recovers the original local ACI from authenticated backup relationships
when group-membership evidence identifies exactly one self member. The importer
uses that ACI for restoration and persists it as the viewer identity, correcting
self mentions and group-update attribution that v0.2 could render as Unknown.
Ambiguous group evidence is rejected instead of guessing.

The media viewer retains Save, Share, and Go to Message but removes Delete and
Forward. Sticker-pack forwarding and long-text forwarding are also removed.
The existing conversation composer and message-mutation restrictions remain.

## v0.3.1 LiveContainer bootstrap fix

The first v0.3 device attempt accepted the recovery key but then entered Signal's
normal profile-setup path and failed with `noIdentityKey`. The viewer runtime
gate depended on `Bundle.main.bundleIdentifier`, which is not a stable viewer
identity when the unsigned app is hosted by LiveContainer.

v0.3.1 makes viewer mode active by default in every binary compiled with
`SIGNAL_BACKUP_VIEWER`. A runtime environment switch can explicitly disable it
for diagnostics, preserving a non-constant branch for Xcode while no longer
depending on the host-reported bundle identifier.

A second device test still failed with `noIdentityKey`. Comparison with the
working v0.2 patch showed that v0.3 had accidentally omitted the local identity
generation and the later websocket/prekey-upload bypasses. The runtime fix was
valid but could not activate code that was absent.

## v0.3.2 identity-bootstrap restoration

v0.3.2 restores the exact Signal-native local ACI/PNI identity and prekey
generation sequence that passed the v0.2 device test. It persists that material
before returning the synthetic offline identity, then skips restricted websocket
setup and network one-time-prekey rotation. The restored bootstrap now runs
before v0.3's authenticated original-ACI inference and read-only import path.

## Build verification

The public GitHub Actions v0.3.2 build completed successfully on September 14,
2026.

- Source commit: Signal iOS `f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5`
  (`8.29.0.1861-beta`).
- Harness build commit: `de65d35d4510b0ff17bbf43bbe14b9f5eb303099`.
- Successful workflow run: [34846700802](https://github.com/drecima/signal-backup-viewer-build/actions/runs/34846700802).
- Bundle identifier, app name, version, ARM64 architecture, absence of signing
  material/extensions, viewer menu marker, LiveContainer-safe runtime marker, local identity creation,
  websocket/prekey-upload bypasses, network interposition section, and
  original-ACI recovery marker were checked during packaging and again after
  downloading the artifact.
- IPA SHA-256:
  `e0a28d8ab74696f253c6a7c86da5c18a13eb5dae62e81d1a08e09a9cc94766df`.

These checks prove that the intended source compiled and was packaged. The
corrected v0.3.2 behavior has also passed the device checks below. The exact
pre-acceptance repository state is preserved on branch
`archive/viewer-v0.3.2-device-accepted`.

## Device acceptance

The v0.3.2 IPA has passed the intended device workflow under LiveContainer:

- Installation and launch succeed without an injected network-disabling tweak.
- Selecting the parent folder containing `SignalBackups` works.
- A correct recovery key imports both the latest tested backup and an older
  backup; attachments restore.
- Note to Self, ordinary one-to-one conversations, and tested group content
  render normally.
- Self mentions and group-update attribution no longer render the local user as
  Unknown.
- Message text, images, generic files, audio, and disappearing-message status
  items expose exact `sbe1:<dateSent>` editor identifiers.
- The copied identifiers match SignalBackupEditor records, including
  attachments and a disappearing-messages-disabled status item.
- Compose, reply, reaction, edit, delete, forward, pin, selection, read-marking,
  media deletion/forwarding, and other tested history-changing actions are
  unavailable.
- Relaunching preserves the imported local archive.

This accepted viewer build is the reference environment for validating edited
backup copies. Further viewer changes should be limited to concrete defects.

## Installation and use

1. Install/sign the IPA as a separate app. Do not inject Signal tweaks or the
   third-party `NetworkDisabler.dylib`. Use a fresh LiveContainer data folder;
   do not reuse failed v0.3 or v0.3.1 registration state.
2. Choose the path for a user without an old device, then choose local backup.
3. In the Files picker, select either `SignalBackups` or its parent folder.
   The viewer selects the newest canonical backup and then asks for the
   64-character recovery key.
4. If the reused Signal setup UI asks for a phone number, enter a syntactically
   valid dummy number such as `+1 650-555-0100`. It is used only as a local
   placeholder and no verification message is requested by the viewer build.
5. Confirm the backup and leave the viewer open while attachment restoration
   completes.

After import, confirm:

- Airplane mode is not required, but it is a useful additional first-test
  precaution.
- Conversations and timestamps are visible.
- Images, voice memos, and files open.
- No composer, reply, reaction, delete, edit, forward, or mark-read behavior is
  available.
- Long-pressing a message offers **Copy Backup Editor ID**.
- Relaunching the viewer preserves the imported archive.

Keep the source backup unchanged. The built-in network block replaces the need
for an unreviewed `NetworkDisabler.dylib`.

## Security boundary

The source patch statically removes or disables Signal's normal connection
refresh, registration websocket, one-time-prekey upload, background launch
jobs, read marking, composers, reaction UI, and mutating message menus. It also
interposes DNS resolution, BSD `connect`/`connectx`, and Network.framework
connection starts.

This is a reviewed application-level containment layer, not a formal proof that
every future Signal or iOS networking implementation must pass through those
symbols. Keep the viewer isolated from the normal Signal app, do not grant it
contacts or notification permissions, and retain the original encrypted
backup. The system Files provider may independently use iCloud to obtain a
selected encrypted folder.

See [`SECURITY_AUDIT.md`](SECURITY_AUDIT.md) for the complete reviewed boundary
and residual limitations.

## Backup Editor ID

Long-press a message and choose **Copy Backup Editor ID**. It copies a selector
such as:

```text
sbe1:1789135600123
```

SignalBackupEditor v0.6.2 accepts this value anywhere `--select` is accepted.
It resolves the backup's `dateSent` value and refuses zero or multiple matches.


## SignalBackupEditor v1.0 work

The companion editor has a reconciled implementation plan and pinned-schema
audit:

- [v1.0 roadmap](editor/ROADMAP.md)
- [Backup-format coverage inventory](editor/FORMAT_COVERAGE.md)
- [v1.0 construction specification](editor/V1_CONSTRUCTION_SPEC.md)
- [Device acceptance procedure](editor/DEVICE_ACCEPTANCE.md)

The preserved v0.12.1 editor package is the required source baseline. Import
and reproduce that exact source before implementing arbitrary message
construction.

## Source

The minimal build harness and full viewer patch are public at:

<https://github.com/drecima/signal-backup-viewer-build>
