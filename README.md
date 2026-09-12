# Signal Backup Viewer v0.1

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

The iOS Files provider is a separate operating-system process. If a selected
backup is stored in iCloud, iOS may download its encrypted files before handing
them to the viewer. The viewer's own network paths remain blocked.

## Build verification

The public GitHub Actions build completed successfully on September 12, 2026:

- Source commit: Signal iOS `f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5`
  (`8.29.0.1861-beta`).
- Harness commit: `526c1705375fadb13f7ab681a838c9cbfad61d1b`.
- Workflow run: `34700975904`.
- Bundle identifier, app name, version, ARM64 architecture, absence of signing
  material/extensions, viewer menu marker, and network interposition section
  were checked after downloading the artifact.
- IPA SHA-256:
  `53c7ebe65750c4ea932a6ca3674aa19d94a2fadb74e5b99f33ab5067d1311dc9`.

These checks prove that the intended source compiled and was packaged. Device
testing is still required to prove launch, import, attachment viewing, and
effective network isolation under the chosen sideloading environment.

## First device test

1. Install/sign the IPA as a separate app. Do not inject Signal tweaks or the
   third-party `NetworkDisabler.dylib`.
2. Choose the path for a user without an old device, then choose local backup.
3. In the Files picker, open the parent folder so that `SignalBackups` is
   visible, but do not enter `SignalBackups`. Select that parent folder and
   enter the 64-character recovery key.
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

This first build should not be trusted with the only copy of a backup. Keep the
source backup unchanged. The built-in network block replaces the need for an
unreviewed `NetworkDisabler.dylib`.

## Backup Editor ID

Long-press a message and choose **Copy Backup Editor ID**. It copies a selector
such as:

```text
sbe1:1789135600123
```

SignalBackupEditor v0.6.1 accepts this value anywhere `--select` is accepted.
It resolves the backup's `dateSent` value and refuses zero or multiple matches.

## Source

The minimal build harness and full viewer patch are public at:

<https://github.com/drecima/signal-backup-viewer-build>
