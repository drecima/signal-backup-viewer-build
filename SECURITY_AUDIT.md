# Signal Backup Viewer v0.2 security and read-only audit

Date: 2026-09-13

## Result

The v0.2 source patch and packaged IPA implement the intended experimental
offline-viewer boundary. Device acceptance confirmed that a native local
backup imports, attachments restore, Note to Self and ordinary conversations
render, exact editor IDs can be copied, and visible history-changing controls
are unavailable.

## Enforced in the viewer build

- Uses the isolated bundle identifier `org.signalbackup.viewer`.
- Packages no notification or watch extensions and contains no provisioning
  profile or stale code signature.
- Creates only a synthetic local identity and skips server registration,
  registration websocket setup, and one-time-prekey upload.
- Stops Signal's normal active connection refresh and launch jobs.
- Interposes DNS lookup, BSD `connect` and `connectx`, and Network.framework
  connection starts.
- Removes the chat-list compose and camera controls and the conversation input
  toolbar.
- Disables send actions, reaction UI, visible read marking, reply, edit,
  deletion, forwarding, pinning, and message selection.
- Replaces message context menus with non-mutating inspection actions and
  `Copy Backup Editor ID`.
- Derives the imported message-backup ID from authenticated local metadata, so
  the original account ACI and a server connection are unnecessary.

## Deliberate local writes

The app container is not mounted read-only. Signal must write its restored
database, migrations, indexes, thumbnails, attachment state, and local viewer
identity material. The boundary is instead:

- The selected encrypted backup is not modified.
- Visible history-changing actions are unavailable.
- The restored database is a disposable private working copy.
- Deleting the viewer container removes that working copy.

## Residual boundaries

- The socket interposition and disabled Signal startup paths are an
  application-level containment layer, not a formal operating-system sandbox
  proof. A future Signal dependency using a different low-level networking
  path would require another audit before rebasing the patch.
- The iOS Files provider runs outside the viewer and may use iCloud to download
  the encrypted selected folder.
- Copying text or an editor ID places that value on the iOS pasteboard. Saving
  or sharing media is an explicit user-requested export from the disposable
  container.
- Signal's own database migration and media-processing code remains trusted to
  parse the authenticated backup safely.

For especially sensitive inspection, airplane mode or an independently
controlled device firewall can add an operating-system-level layer. The
unreviewed third-party `NetworkDisabler.dylib` is not required and is not part
of the release.

## Release checks

- Signal source tag: `8.29.0.1861-beta`
- Signal source commit: `f8170e0bf7b2e7fb70bcdfaedd0abe3b5030e8c5`
- Viewer harness commit: `914ca5ddecfd7e568c50c3ced47c549ea66e0649`
- Architecture: ARM64 iPhone device build
- Bundle identifier: `org.signalbackup.viewer`
- Extensions: absent
- Signing material: absent
- Viewer and network-interposition markers: present
- IPA SHA-256:
  `57a3d0c100d12778d352aa846148fba247d5c545e52f28b16a5c9d2f764bf9c6`
