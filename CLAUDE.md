# bluebubbles-helper (xcelerate fork)

This is **Stack 1** of a 2-stack iMessage architecture for the xcelerate CRM.
This repo provides the dynamic library (`BlueBubblesHelper.dylib`) that gets
injected into Apple's `Messages.app` on macOS, exposing private-API hooks the
BlueBubbles server uses for things AppleScript can't reach (typing
indicators, reactions, share-name, etc.). Forked from upstream BlueBubbles
with custom patches for the xcelerate VIP persona.

## What's custom in this fork

- **`set-nickname-info` action** — writes the macOS Share Name + photo via
  raw `setPersonalNickname:` ivar setter, bypassing Apple's first-time-setup
  gate. Lets us brand a session ("Elli Lacy" + selfie) without a human ever
  clicking through the iMessage onboarding flow.
- **`installAutoShareNicknameHook`** — auto-allows share-nickname for every
  chat (otherwise Messages.app prompts the user every time).
- **`autoAddParticipantsToContactsForChat:`** — every incoming sender gets
  auto-imported into macOS Contacts.app via the patched server's osascript
  helper, so iCloud syncs the contact across the user's devices.
- **`block-handle` event** — exposes `IMHandle.setBlocked:` so the CRM's
  conversion-blocker stage 2 can hard-block at the Messages.app level.

Lines of interest in `Messages/MacOS-11+/BlueBubblesHelper/BlueBubblesHelper.m`:
- ~106-125 `+load` triggers `installAutoShareNicknameHook` after delay
- ~130-185 `installAutoShareNicknameHook` + `allowShareForChat:` + `autoAddParticipantsToContactsForChat:`
- ~750-770 `set-nickname-info` action handler with raw `setPersonalNickname:` setter

## Sister repos

- [support-xcele/bluebubbles-server](https://github.com/support-xcele/bluebubbles-server) — the BlueBubbles server fork (rebranded as xMessage). This dylib is shipped inside the .app bundle that repo builds.
- [support-xcele/imessage-bridge](https://github.com/support-xcele/imessage-bridge) — Stack 2: a Linux/Railway-hosted Rust service wrapping rustpush. Different architecture — multi-Apple-ID per process, no Mac per account. Both stacks coexist; Stack 1 hosts the high-stability VIP, Stack 2 hosts the disposable cohort.
- [support-xcele/xcelerate-telegram](https://github.com/support-xcele/xcelerate-telegram) — the CRM that talks to BOTH stacks via `lib/server/imessage-bridge.js`.

## How to build + ship

```bash
# 1. Clone alongside bluebubbles-server so the build script can find this dylib
git clone https://github.com/support-xcele/bluebubbles-helper
git clone https://github.com/support-xcele/bluebubbles-server

# 2. Build the dylib (Xcode + arm64e, see CONTRIBUTING.md for the gotchas)
cd bluebubbles-helper/Messages/MacOS-11+
xcodebuild -project BlueBubblesHelper.xcodeproj -configuration Release

# 3. The build output is NOT auto-copied — manually drop it into the
#    bluebubbles-server appResources path:
cp build/Release/BlueBubblesHelper.dylib \
   ../../../bluebubbles-server/packages/server/appResources/private-api/macos11/

# 4. Then build the .app per bluebubbles-server's build pipeline.
```

The build script in `bluebubbles-server` does NOT recompile this dylib —
it just bundles whatever's already in `appResources/private-api/macos11/`.
That's why we keep the dylib checked in to bluebubbles-server (against
upstream's wishes, but it's our fork).

## Running production

- The deployed app at `/Applications/xMessage.app` injects this dylib into
  `Messages.app` when xMessage launches. Verify via `pgrep -afl xMessage |
  grep -v grep` and `lsof -nP -iTCP:1234 -sTCP:LISTEN` on the Mac mini.
- Health check from anywhere: `curl -sS http://127.0.0.1:1234/api/v1/icloud/contact?password=$PW` (password in `~/Library/Application Support/bluebubbles-server/config.db`).
- Brand verified by `data.name = "Elli Lacy"` and a base64 avatar in the response.

## Build gotchas (from prior trial-and-error)

1. **Helper dylib not auto-copied** — the bluebubbles-server build pipeline
   bundles the dylib but doesn't recompile it. Always rebuild + copy
   manually before building the .app.
2. **Pods need arm64e ARCHS** — without it, the dylib compiles for arm64
   only, and Apple's runtime refuses to load it into `Messages.app`
   (which is arm64e). Fix in the Pods target's build settings.
3. **`--deep` resign invalidates Gatekeeper** — every `--deep` resign of
   `xMessage.app` triggers a "do you want to open this app?" dialog the
   FIRST time the new build launches. On a headless Mac mini this is
   blocking. Avoid `--deep` unless absolutely necessary.

## When working on this repo

- The helper's behavior is invisible — it patches Messages.app's runtime.
  Don't expect logs in the helper itself; logs appear in the
  bluebubbles-server logs (forwarded via the IPC channel).
- Test changes against the staging Apple ID before deploying to the VIP
  account. A bad helper crashes Messages.app, which kills the bridge.
- The arm64e + private-API combination is fragile across macOS versions.
  Pin the Mac mini's macOS until you've tested the new version.

## Last commit context

`9de381b` — "feat(helper): block-handle event + auto-share-nickname hook"
(2026-05-03). The `wip/auto-backup-2026-05-05` branch contains nothing for
this repo — only build artifacts in `Messages/MacOS-11+/build/` were dirty
when the local workspace was archived, and those are regenerable.
