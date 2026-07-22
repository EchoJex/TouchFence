# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## The one thing that changes everything: the source lives inside a zip

This repo tracks only three things: `README.md`, `.github/workflows/build.yml`, and
**`TouchFence.zip`**. The entire Android app — all Kotlin, the manifest, Gradle
files, and the committed `debug.keystore` — is packaged *inside* `TouchFence.zip`.
The CI workflow unzips it and builds from `TouchFence/` at the archive root.

A normal file scan of this repo will NOT show you the app. To see or change code you
must go through the zip.

### Read a file inside the zip
```bash
unzip -l TouchFence.zip                 # list entries
unzip -p TouchFence.zip TouchFence/app/src/main/java/dev/solo/touchfence/BlockerService.kt
```

### Edit code (extract → edit → repackage → commit the zip)
```bash
# 1. extract somewhere scratch
unzip -o -q TouchFence.zip -d /tmp/tf

# 2. edit files under /tmp/tf/TouchFence/...

# 3. repackage by UPDATING ONLY THE CHANGED ENTRIES into a copy of the original zip.
#    This preserves every other entry byte-for-byte (critically the binary
#    debug.keystore and gradle-wrapper.jar — a full re-zip can corrupt/alter them).
cp TouchFence.zip /tmp/work.zip
cd /tmp/tf && zip -q /tmp/work.zip TouchFence/app/src/main/java/dev/solo/touchfence/BlockerService.kt
cd - && cp /tmp/work.zip TouchFence.zip

# 4. VERIFY before committing: only the intended entries changed, and prior work survived
unzip -v TouchFence.zip | awk 'NF>=8{print $8}'    # compare entry set vs git HEAD's zip
unzip -p TouchFence.zip <path> | grep <expected-change>
```

### Hard-won rules when repackaging
- **Always rebuild from the committed `TouchFence.zip` (git HEAD), not a stale
  extraction.** This session's workspace can rewind between turns; a stale base
  silently *reverts* earlier fixes. After repackaging, grep the packaged file for
  markers of *previous* changes (not just your new one) to confirm they survived.
- Change only the entries you touched; confirm with a CRC/entry diff against
  `git show HEAD:TouchFence.zip`.

## Build / test / run — there is no local build

There is no Android SDK or Gradle in this environment, and **no test suite**.
You cannot compile here. Verification happens in CI on push. Before pushing,
self-check Kotlin by eye: brace/paren balance and that every API you used is
imported (this is the project's stated off-device QA gate). The on-device crash
reporter (separate `:crash` process, `CrashActivity`) catches runtime API misuse.

CI (`.github/workflows/build.yml`) on every push and `workflow_dispatch`:
1. checkout → `unzip TouchFence.zip` → JDK 17 → `./gradlew assembleDebug` with
   `-PversionCodeOverride=$((1000 + GITHUB_RUN_NUMBER))`.
2. uploads the APK as artifact `TouchFence-apk`, and
3. (re)publishes a single **rolling `latest` GitHub Release** carrying
   `TouchFence-<code>.apk` plus a `versionCode=<code>` note.

Getting the APK: repo **Releases** page (plain APK, not a zip) or the in-app
**Check for updates** button. Actions *artifacts* need a login; Releases don't
(the repo is public), which is why the in-app updater fetches from Releases.

## Versioning

`app/build.gradle.kts` pins `versionCode = 13` for local/manual builds but honors
`-PversionCodeOverride`, so every CI build gets a unique, increasing code
(`1000 + run_number`). The in-app updater compares that code against the installed
`longVersionCode` to decide "newer". The committed `debug.keystore` means any build
installs over any other — no uninstall needed.

## App architecture (package `dev.solo.touchfence`, ~10 Kotlin files)

Personal utility for a Pixel with a cracked digitizer: block touch input over
damaged screen regions. Framework APIs only — **zero third-party dependencies**
(the `dependencies {}` block is intentionally empty). The only network use is the
self-updater.

The data flow that ties the app together (read these together to understand it):

- **`DrawMaskActivity`** — two-phase mask editor. DRAW: freeform brush strokes
  (below a guard line) on a white canvas. Then rasterizes strokes → coarse grid →
  merges covered cells into `≤ MAX_RECTS` rectangles. POSITION: a D-pad nudges the
  whole rect set; the offset persists. The *draft* (strokes + offset) is the source
  of truth, not the rects.
- **`MaskStore`** (also defines the `Stroke` class) — persistence. Two things:
  the final blocked **rects** (what the blocker reads, `l,t,r,b` per line in absolute
  portrait px) and the **draft** (`offset:x,y` + `width;x,y;...` per stroke). Also
  `MANUAL_OVERRIDE` (hardcode rects, bypass UI) and last-mask backup/`recallLast`.
- **`BlockerService`** — foreground service. Loads rects, runs `consolidate()` to
  cap window count (`MAX_WINDOWS`, guarded by `MAX_MERGE_WASTE` so merges never
  bridge a gap and swallow clear screen), and puts **one touch-eating overlay window
  per rect** (`TYPE_APPLICATION_OVERLAY`, `NOT_FOCUSABLE`). Each `WallView` fills its
  window with the faint `ZONE_COLOR` tint plus a solid `ZONE_EDGE_COLOR` outline —
  **fill and blocked region are the same window**, so the visible red *is* exactly
  what blocks. Notification carries Pause/Stop escape hatches.
- **`CalibrateActivity`** ("Auto test") — characterizes ghost touches with the
  blocker stopped. Point model: every reported sample becomes an independent mask
  dot at its float coord (no swipes, no shape fill, panel contact-size is NOT
  trusted). Contacts are kept only as timing envelopes (nanosecond clock via
  reflection). Feeds dots back into the draft (`Add to mask` / auto-apply). Forces
  max refresh rate, partial wake lock, urgent-display thread priority, and
  unbuffered dispatch during a run. "Safe"/pressure-mute zones let you press the
  glass in a rotating spot that's excluded from detection.
- **`MainActivity`** — control panel (anchored to the lower, trustworthy half of the
  screen). Start/stop, permissions, export/import, Check for updates. **Auto-starts
  blocking on launch** when a mask exists and overlays are permitted. First-run
  onboarding overlay uses a human-style tap gate (duration + travel) to launch the
  auto-test-and-apply flow.
- **`QuickApply`** — volume-up+down chord (in-app only) applies/recalls the mask.
- **`LogStore`** — persistent CSV of every auto-test observation + active mask;
  export to Downloads via MediaStore (no storage permission) or clipboard.
- **`Updater`** — reads the rolling `latest` Release, and on a newer `versionCode`
  downloads the APK and drives a `PackageInstaller` session to the system install
  prompt. `INTERNET` + `REQUEST_INSTALL_PACKAGES`; all network/IO off the main thread.
- **`App`** — installs the uncaught-exception handler that opens `CrashActivity`.

### Key invariants
- **Coordinates are absolute portrait pixels** end to end: the editor captures
  `rawX/rawY` against full-screen `currentWindowMetrics.bounds`, and the blocker
  places overlays at those same coords with `FLAG_LAYOUT_IN_SCREEN`. Keep any new
  geometry in this one space.
- **Portrait only.** Overlays cannot cover the lock screen, status bar, or shade —
  don't design features that assume they can. Real-time per-touch filtering across
  *other* apps is impossible unrooted (an overlay is all-or-nothing per region);
  don't promise it.

## Design source of truth

The authoritative spec is the external **project schema** (a handoff doc the user
maintains, delivered as a PDF/markdown, sections A–H plus a "Tunables quick
reference"). When code and schema disagree, the schema wins — but it can contain
internal contradictions (prose vs. the tunables table); surface those rather than
guessing. Keep the schema's file count, tunables table, and feature list in sync
when you change behavior.

## GitHub workflow notes

- CI publishes Releases, so `permissions: contents: write` and the preinstalled
  `gh` CLI are used (no third-party actions).
- The session git proxy blocks branch **deletion** (`git push --delete` → 403);
  delete merged branches from the GitHub UI instead.
