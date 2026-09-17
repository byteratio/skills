---
name: iphone-dev
description: Conventions for native iPhone app development — SwiftUI + SwiftData local-first apps, XcodeGen project generation, CLI-only build/test/deploy loops (simulator by default, signed device deploys via devicectl), XCTest with visible coverage, and the free-personal-team signing gotchas. Use when creating an iOS/iPhone app, adding iOS targets or Makefile loops to a project, debugging Xcode signing/provisioning/deploy problems, or porting an existing app's logic into a native Swift client.
---

# iPhone app development

Conventions distilled from shipping a SwiftUI app end-to-end, from
empty repo to a signed build running on a physical phone.

**Assumed context**: small-scale apps, a free personal Apple account,
no App Store distribution. A paid developer team changes several rules
below — the signing section flags which ones.

## Architecture defaults

- **SwiftUI + SwiftData, local-first.** A small app should not require
  a reachable server. Data lives on-device in SwiftData; network calls
  are for public APIs only (https, no ATS exceptions). If the app
  starts as a thin client for an existing backend, expect the "can we
  drop the server?" pivot — port the backend's logic into pure Swift
  functions over the SwiftData arrays rather than keeping a server
  alive for one phone.
- **SwiftData modeling**: real object relationships, not integer
  foreign keys. `@Attribute(.unique)` for natural keys. SwiftData
  to-many arrays don't preserve order — add a `sortOrder` field and a
  `sortedX` computed property when order matters. Dates as `Date`, not
  strings. Enums as `String`-raw `Codable`.
- **Schema changes on a live install**: adding models or optional
  properties lightweight-migrates silently on the next launch; renames,
  deletions, or type changes without a `SchemaMigrationPlan` make the
  store fail to open — the deployed app then crashes at startup with
  the user's data intact but hostage. Redeploys (same bundle ID) never
  wipe the data container; only deleting the app does. Plan an
  export/backup path before users accumulate real data.
- **Views talk to `@Query` + `modelContext` directly.** No Store /
  repository layer unless something genuinely needs caching. Pure logic
  (matching, parsing, importing) lives in enum namespaces
  (`enum Foo { static func … }`) — testable without UI.
- **Deployment target**: current-major-minus-one unless a needed API
  says otherwise. One target, `TARGETED_DEVICE_FAMILY: "1"`, portrait
  unless the app earns rotation.

## Project generation — XcodeGen, never a committed .xcodeproj

`ios/project.yml` is the source of truth; the `.xcodeproj` bundle is
generated and gitignored (also gitignore `ios/build/` and the generated
`Info.plist`). Regenerate after adding or removing files — xcodegen
globs the sources dir, so edits to existing files need nothing.

- Install xcodegen from the GitHub release zip (`PREFIX=~/.local
  ./install.sh`) — no Homebrew required, works on a locked-down Mac.
- Unit-test targets need `GENERATE_INFOPLIST_FILE: YES` or the test
  bundle fails to sign.
- Define the scheme in `project.yml` with `gatherCoverageData: true` and
  the test target attached, or `xcodebuild test` has nothing to run.

## The dev loop — CLI-only, simulator by default

Every iOS project gets these Makefile targets. Parameterize the sim
name, device id, and team id as `?=` variables so nothing
machine-specific is hardcoded:

| Target | What |
|--------|------|
| `ios-project` | xcodegen generate + open in Xcode |
| `ios-sim` | build unsigned for simulator, boot sim, install, launch — **the default loop; zero signing, zero password prompts** |
| `ios-sim-seed` | ios-sim, then relaunch with `--seed-demo` |
| `ios-test` | `xcodebuild test` in the simulator + `xccov` per-target coverage report |
| `ios-deploy` | signed build for the plugged-in phone + `devicectl` install + launch |

`ios-test` prints coverage on every run — the house rule is that
`make test` always shows the number, with no hard threshold attached.

Simulator builds: `CODE_SIGNING_ALLOWED=NO` is fine for
`generic/platform=iOS` compile checks; simulator destinations need no
signing at all. Remember sim and phone have **separate databases**.

## Debug conveniences that always pay off

- **Sample-data seeder** (`DemoData.swift`, `#if DEBUG`): curated
  dataset with expiry/date fields computed relative to `.now`,
  idempotent (re-run adds nothing), exposed three ways — a Settings
  "Developer" section button, a `--seed-demo` launch argument checked
  via `ProcessInfo.processInfo.arguments`, and the `ios-sim-seed`
  target. Add a test asserting the seed exercises the app's main states
  so it can't silently rot.
- **Searchable pickers over wheel Pickers** once a list can exceed a
  couple dozen entries (imported datasets guarantee this).
- Camera/scanner features: check `DataScannerViewController.isSupported
  && .isAvailable` and ship a manual-entry fallback — it degrades
  gracefully in the simulator too.

## Testing

- XCTest in the simulator; when porting logic from another stack, port
  the reference test suite too so semantics are pinned cross-language.
- Shared helper: `makeInMemoryContext()` building a ModelContainer with
  `isStoredInMemoryOnly: true`; `@MainActor` test classes with
  `setUp() async throws`.
- Read pass/fail from the result bundle:
  `xcrun xcresulttool get test-results summary --path build/test.xcresult`.

## Signing & deploy — the gotchas (each cost real time)

- **The team ID is NOT the parenthetical in `security find-identity`** —
  that's the certificate's own ID. Get the real team ID from a
  provisioning profile (`security cms -D -i <profile>` → the
  `application-identifier` prefix) or Xcode's Accounts pane. The wrong
  ID gives a misleading "No Account for Team" error.
- **`-allowProvisioningUpdates` cannot see the Apple ID session from a
  plain terminal.** Create the provisioning profile once in the Xcode
  GUI (Signing & Capabilities → set Team → Try Again); after that,
  pure-CLI builds work with `DEVELOPMENT_TEAM=<id>
  CODE_SIGN_STYLE=Automatic`.
- **First-device dance**: enable Developer Mode on the phone (Settings →
  Privacy & Security; the toggle appears only after Xcode has talked to
  the device; reboots) → trust the computer → after first install, trust
  the developer cert (Settings → General → VPN & Device Management).
  Xcode's "your team has no devices" error usually means the
  registration attempt happened while the phone was rebooting or locked
  — select the phone as run destination and retry.
- **Keychain prompts on every build**: click **Always Allow** on the
  codesign keychain dialog once; the per-build password prompts stop.
- **Free personal team limits**: installs expire after **7 days**
  (redeploy to re-sign), max 3 sideloaded apps, no push notifications,
  no App Groups. A paid team lifts all four — if the app needs push or
  App Groups, that's the trigger to get one.
- Deploy/launch CLI: `xcrun devicectl list devices`,
  `xcrun devicectl device install app --device <id> <path>.app`,
  `… device process launch --device <id> <bundle-id>`.

## Fresh-Mac bootstrap (headless-friendly)

1. Xcode from the App Store (open
   `macappstore://apps.apple.com/app/id497799835`).
2. `sudo xcode-select -s /Applications/Xcode.app && sudo xcodebuild
   -license accept && sudo xcodebuild -runFirstLaunch` — needs an admin;
   on a standard account, fix admin membership first
   (`su admin -c 'sudo dseditgroup -o edit -a <user> -t user admin'`).
3. `xcodebuild -downloadPlatform iOS` — the iOS SDK + simulator runtime
   is a separate multi-GB download since Xcode 15; builds fail with
   "iOS X.Y is not installed" without it.
4. `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer` works as a
   per-command override before xcode-select is fixed.
5. xcodegen from the release zip into `~/.local` (see above).

## Companion clients for an existing server app — sync tiers

When the iPhone app joins an app that already has a server + webapp (the
server stays the single source of truth), pick the cheapest tier the
phone's use case allows. Ask "does the phone need to work away from the
server's network?" and "capture or full editing?" first:

1. **Thin client** — no local store; phone calls the existing API. Right
   when connectivity is a given.
2. **Snapshot mirror** — SwiftData cache rebuilt **wholesale** from a
   full JSON pull (single-user DBs are small). No merging → no
   tombstones, no per-row timestamps, no conflict engine; server-side
   deletes/edits arrive for free. Photos lazy-load via URL cache.
3. **Scoped outbox** — offline *creation* limited to **children of
   records that already exist** (logs, notes, photos against known
   parents). Parents carry real server IDs, so replay is plain API calls
   — integer-PK schemas need **no UUID migration**. Sync event = push
   outbox, then pull snapshot (own captures come back confirmed).
   Surface 404-orphans ("N couldn't sync"), never drop silently. Show a
   pending badge + last-synced timestamp. Auto-sync only when the user's
   pinned server address answers — never probe foreign networks.
4. **Full bidirectional sync** (UUIDs, tombstones, delta endpoints,
   LWW/CRDT) — only with evidence tiers 1–3 don't suffice; the rework
   starts server-side (client-mintable UUIDs, idempotent upserts,
   `changes?since=` + tombstones) before any Swift is written.

Tiers 2 and 3 compose well: a snapshot mirror for reads plus a scoped
outbox for captures covers "works at home, captures anywhere" without
any of tier 4's machinery.

## Project hygiene

- Machine- and device-specific facts (team ID, device UDID, sim name) go
  in Makefile `?=` vars or the project's own notes — never hardcoded in
  scripts, and never in this skill.
- Keep a `plans/<topic>.md` document for each in-flight slice of work
  and a top-level `CHANGELOG.md` for what shipped; move a plan to
  `plans/done/<topic>.md` with a "Delivered in \<version\> — \<date\>"
  header once it lands.
- Size each plan slice so it ends with **something runnable on a phone
  or simulator**. An iOS slice that can't be launched can't be reviewed.
