# FORK.md - Wasim's cmux fork

This is a personal fork of [manaflow-ai/cmux](https://github.com/manaflow-ai/cmux). It carries
custom features that do not exist upstream. This file documents what those features are, which files
they live in, and the exact loop to pull in a new upstream version without losing them.

Agent rule overrides for this fork live in `.claude/rules/wasim-overrides.md`.

Keep this file fork-only. Upstream never touches `FORK.md`, so it never causes a merge conflict.
That is the whole point: our fork notes live here, not in `CLAUDE.md` (which upstream owns and edits).

## Remotes

- `origin` -> `https://github.com/wasimjalali/cmux-fork.git` (our fork)
- `upstream` -> `https://github.com/manaflow-ai/cmux.git` (the real cmux)

A fresh `git clone` of the fork only sets up `origin`. Add `upstream` by hand or the update loop below
has nothing to fetch:

```bash
git remote add upstream https://github.com/manaflow-ai/cmux.git
```

## Our custom features

### 1. Sidebar folder drag-and-drop

Drop a folder onto the sidebar to open it as a new workspace. The workspace shows the folder name
and auto-runs `claude` in it.

Design and plan docs:

- `docs/superpowers/specs/2026-07-11-sidebar-folder-drop-design.md`
- `docs/superpowers/plans/2026-07-11-sidebar-folder-drop.md`
- `docs/superpowers/plans/2026-07-11-sidebar-folder-drop-v2.md`

### 2. App-wide zoom on Cmd+= / Cmd+-

Cmd+= zooms the entire app in (sidebar, tab bar, terminals, panels, chrome), Cmd+- zooms out.
It drives the existing `GlobalFontMagnification` percent (50-200%, steps of 10, persisted in
UserDefaults), so the zoom level survives restarts. The chords are routed before the per-pane
browser/markdown/terminal zooms; unbind `globalZoomIn` / `globalZoomOut` in Settings >
Keyboard Shortcuts to get upstream's per-pane behavior back. Also exposed in the command
palette ("App Zoom In" / "App Zoom Out").

New shortcut actions: `globalZoomIn`, `globalZoomOut` (in both `KeyboardShortcutSettings.Action`
and the `CmuxSettings` `ShortcutAction` mirror; the drift test keeps them aligned).

## Files the fork owns or changes

Knowing this list is what makes updates safe. When an upstream merge conflicts, it will only ever be
in the "modified upstream files" group below. The "fork-only" files never conflict.

**Fork-only new files** (upstream does not have these, so they never conflict):

- `Sources/DirectoryDropFilter.swift`
- `Sources/SidebarDropRegionProbe.swift`
- `Sources/SidebarDropRegionRegistry.swift`
- `cmuxTests/DirectoryDropFilterTests.swift`
- `cmuxTests/SidebarDropRegionRegistryTests.swift`
- `Packages/macOS/CmuxFoundation/Tests/CmuxFoundationTests/GlobalFontMagnificationSteppingTests.swift`
- the three docs listed above

**Modified upstream files for app-wide zoom** (conflict surface for that feature):

- `Packages/macOS/CmuxFoundation/Sources/CmuxFoundation/GlobalFontMagnification.swift` - step helpers
- `Sources/KeyboardShortcutSettings.swift`, `Sources/AppDelegate.swift`,
  `Sources/AppDelegate+CanvasShortcutRouting.swift` - `globalZoomIn`/`globalZoomOut` actions + routing
- `Packages/macOS/CmuxSettings/Sources/CmuxSettings/Values/ShortcutAction.swift` (+`+Defaults.swift`) - mirror enum
- `Sources/ContentView.swift`, `Sources/ContentView+ViewCommandPalette.swift` - command palette entries
- `Packages/macOS/CmuxSettingsUI/Sources/CmuxSettingsUI/Sections/AppSection.swift` - subtitle copy
- `Resources/Localizable.xcstrings`, `web/data/cmux-shortcuts.ts`, `web/data/cmux.schema.json`,
  `skills/cmux-settings/references/shortcut-actions.md` - strings, docs, schema
- `cmuxTests/KeyboardShortcutContextTests.swift` - default-chord regression test

**Modified upstream files** (this is the conflict surface, check these on every merge):

- `Sources/ContentView.swift` - wires in the drop overlay. Look for `onFoldersDroppedOnSidebar`
  and `.background(SidebarDropRegionProbe())`.
- `Sources/FileDropOverlayView.swift` - fork edits to the drop overlay.
- `Sources/FileDropOverlayViewHitTesting.swift` - fork edits to hit testing.
- `cmux.xcodeproj/project.pbxproj` - wires the new Swift files into the Xcode target.
- `.gitignore` - adds `build-release/`.

## The update loop (run this every time upstream ships a new version)

Follow these steps whenever you want to pull a new cmux version into the fork. This is the system
that keeps our features. Never click the in-app Sparkle "Update" button as your way of updating; that
only swaps the installed binary and does nothing for our source. Updates happen here, in git.

Replace `vX.Y.Z` with the target version (e.g. `v0.64.19`).

```bash
# 0. Start clean on the feature branch that holds our work.
git checkout feat/sidebar-folder-drop
git status   # must be clean

# 1. Get the new upstream version and tags.
git fetch upstream --tags

# 2. See what is coming and where it might conflict with us.
#    BASE is the merge-base; the last arg is the new tag.
BASE=$(git merge-base feat/sidebar-folder-drop vX.Y.Z)
comm -12 \
  <(git diff --name-only $BASE feat/sidebar-folder-drop | sort) \
  <(git diff --name-only $BASE vX.Y.Z | sort)
#    -> the printed files are the only possible conflicts. Expect them to be a
#       subset of the "modified upstream files" list above.

# 3. Do the merge on a dedicated integration branch (keeps the feature branch safe).
git checkout -b merge/upstream-vX.Y.Z feat/sidebar-folder-drop
git merge vX.Y.Z --no-edit
```

### Resolving conflicts

- **`Sources/*.swift`**: keep BOTH sides. Our drop code and upstream's new code should coexist.
  After resolving, confirm our hooks are still present:
  ```bash
  grep -n "onFoldersDroppedOnSidebar\|SidebarDropRegionProbe" Sources/ContentView.swift
  ```
- **`cmux.xcodeproj/project.pbxproj`**: this conflicts because both sides add file entries in the
  same sorted region. The fix is union (keep both), then let the normalizer re-sort:
  ```bash
  # strip only the conflict-marker lines (keeps both sides everywhere)
  grep -vE '^(<<<<<<< HEAD|=======|>>>>>>> vX\.Y\.Z)$' cmux.xcodeproj/project.pbxproj > pbx.tmp
  mv pbx.tmp cmux.xcodeproj/project.pbxproj
  python3 scripts/normalize-pbxproj.py
  bash scripts/check-pbxproj.sh                 # must exit 0
  bash scripts/lint-pbxproj-test-wiring.sh      # must say ok
  git add cmux.xcodeproj/project.pbxproj
  ```
  Then confirm every fork file is still wired (4 refs each is normal):
  ```bash
  for f in DirectoryDropFilter SidebarDropRegionProbe SidebarDropRegionRegistry \
           FileDropOverlayView FileDropOverlayViewHitTesting \
           DirectoryDropFilterTests SidebarDropRegionRegistryTests; do
    echo "$f: $(grep -c "$f.swift" cmux.xcodeproj/project.pbxproj)"
  done
  ```
- **`.gitignore`**: trivial, keep both sides.

### After conflicts are resolved

```bash
# 4. Sync submodules to the merged pointers (upstream often bumps ghostty / bonsplit).
git submodule update --init --recursive

# 5. Commit the merge.
git commit --no-edit

# 6. Build. reload.sh runs ensure-ghosttykit.sh, which repoints GhosttyKit.xcframework
#    to the new ghostty SHA automatically.
./scripts/reload.sh --tag merge-vX-Y-Z

# 7. Run our regression tests.
xcodebuild test -project cmux.xcodeproj -scheme cmux -destination 'platform=macOS' \
  -only-testing:cmuxTests/DirectoryDropFilterTests \
  -only-testing:cmuxTests/SidebarDropRegionRegistryTests

# 8. Push the merge branch to the fork, then dogfood the tagged build.
git push -u origin merge/upstream-vX.Y.Z
```

Dogfood the built app first (drag a folder onto the sidebar, confirm it opens a workspace and runs
claude). Only after it works, fast-forward the feature branch to the merge:

```bash
git checkout feat/sidebar-folder-drop
git merge --ff-only merge/upstream-vX.Y.Z
git push origin feat/sidebar-folder-drop
```

A PR on the fork is optional for a version sync. The real diff is ~1000+ files of already-reviewed
upstream code, so a bot review is mostly noise. The substance to check is the conflict resolution and
that the feature still works, which the build + regression tests already prove. Open a PR only if you
want the record.

## Build toolchain per machine (zig / Ghostty CLI helper)

The Xcode build has a "Build Ghostty CLI helper" run-script phase that needs a specific zig. The
gotchas turned out to differ per machine, so they are split below. Read the section for the machine
you are actually on, the two are not interchangeable.

### Company MacBook Pro (Mac17,9, Apple M5 Pro) - current machine

The Nature Heart company machine. macOS 26.5.2 (25F84), Xcode 26.0 (17A324), Apple Silicon.
Set up 2026-07-26. Everything below is verified on it by building v0.64.19 Release from `main`.

**Full setup from scratch.** In this order:

```bash
git clone --recursive https://github.com/wasimjalali/cmux.git
cd cmux
git remote add upstream https://github.com/manaflow-ai/cmux.git   # not created by the clone

# zig 0.15.2, isolated. Export both: ensure-ghosttykit.sh checks `command -v zig`,
# the Xcode run-script phase reads CMUX_ZIG.
ZIG_FORCE_LOCAL_INSTALL=1 ZIG_INSTALL_ROOT="$HOME/.cache/cmux/zig" bash scripts/install-zig-ci.sh
export CMUX_ZIG="$HOME/.cache/cmux/zig/zig-aarch64-macos-0.15.2/zig"
export PATH="$HOME/.cache/cmux/zig/zig-aarch64-macos-0.15.2:$PATH"

xcodebuild -downloadComponent MetalToolchain   # one time, 705 MB
./scripts/ensure-ghosttykit.sh
./scripts/install-git-hooks.sh
```

**Xcode has to come from `xcodes`, not the App Store.** This Mac is DEP-enrolled and MDM-managed
through Apple Business Manager, and the Apple ID signed in to it is a Managed Apple ID. Apple blocks
Managed Apple IDs from App Store downloads entirely, so the Xcode "Get" button is permanently greyed
out. The `xcodesorg/made` Homebrew formula is not a way out either: it builds from source and that
build needs Xcode's own `xcbuild`, so it cannot bootstrap. Use the prebuilt notarized `xcodes` binary
from the project's GitHub releases instead, with a personal (non-managed) Apple ID.

- `xcodes` binary lives at `~/.local/bin/xcodes`, it is not a Homebrew install
- the Apple ID needs a free Apple Developer account that has **accepted the developer agreement**
- Xcode installs to `/Applications/Xcode-26.0.0.app`, not `/Applications/Xcode.app`
- then `sudo xcode-select -s /Applications/Xcode-26.0.0.app`, then `sudo xcodebuild -license accept`,
  in that order. The license step cannot run before `xcode-select` repoints away from
  `/Library/Developer/CommandLineTools`.

If the developer agreement has not been accepted, `xcodes install` reports the download as complete
but writes Apple's `Unauthorized` HTML page to disk *as* the `.xip`, and the unarchive step then dies
with a `Unxip.swift` precondition failure. Sanity-check the archive size: an 84 KB `.xip` where
~15 GB belongs is that failure, not a corrupt download. Delete it before retrying, otherwise `xcodes`
reports "Found existing archive that will be used for installation" and reuses the error page.

**The macOS 26 SDK link failure does NOT reproduce here.** The older machine needed
`CMUX_SKIP_ZIG_BUILD=1` because zig 0.15.2 could not link libSystem against the macOS 26 SDK. Under
Xcode 26.0 on this machine the real helper builds cleanly and the bundled `ghostty` CLI comes out as
a real 12 MB binary rather than a stub. Do not reach for `CMUX_SKIP_ZIG_BUILD=1` here unless a build
genuinely fails.

**Xcode 26 ships the Metal toolchain as a separate component.** Without it the build fails with
`cannot execute tool 'metal' due to missing Metal Toolchain`. One-time 705 MB download, see the
setup block above.

**A Release build needs the entitlements overridden.** `Resources/cmux.entitlements` is wired into
the Release config only (Debug ships `CODE_SIGN_ENTITLEMENTS = ""` by design, which is why
`reload.sh` works for everyone). It sets
`keychain-access-groups = $(AppIdentifierPrefix)$(PRODUCT_BUNDLE_IDENTIFIER)`, a restricted
entitlement whose `$(AppIdentifierPrefix)` resolves to Manaflow's team `7WLXT3NR37`. We have no
certificate for that team, so the build fails with "has entitlements that require signing with a
development certificate". A free personal Apple Developer cert does not fix this, `com.cmuxterm.app`
is already claimed under Manaflow's team. Override on the command line so no repo file changes:

```bash
xcodebuild -project cmux.xcodeproj -scheme cmux -configuration Release -destination 'platform=macOS' \
  CODE_SIGN_ENTITLEMENTS="" CODE_SIGN_IDENTITY="-" CODE_SIGN_STYLE=Manual DEVELOPMENT_TEAM="" \
  build
```

`ENABLE_HARDENED_RUNTIME` is `NO` in this project, so dropping entitlements costs nothing beyond the
keychain access group. If cmux ever misbehaves around keychain-stored credentials, that is the cause,
and the Debug config (`./scripts/reload.sh --tag <tag>`) is the fallback.

The Release app lands in DerivedData, `reloadp.sh` launches it from there rather than installing it.
For a real installed app, copy it across:

```bash
cp -R ~/Library/Developer/Xcode/DerivedData/cmux-*/Build/Products/Release/cmux.app /Applications/cmux.app
```

### Personal MacBook (older machine)

Two gotchas hit that machine when building v0.64.19:

1. **Exact zig version.** The phase requires exactly `zig 0.15.2`. Homebrew there has `0.16.0`, which
   the strict check rejects. Zig 0.15.2 is installed isolated at
   `~/.cache/cmux/zig/zig-aarch64-macos-0.15.2/zig` (Homebrew's 0.16.0 untouched). Point the build
   at it with `CMUX_ZIG`:
   ```bash
   export CMUX_ZIG="$HOME/.cache/cmux/zig/zig-aarch64-macos-0.15.2/zig"
   ```
   To reinstall it later: `ZIG_FORCE_LOCAL_INSTALL=1 ZIG_INSTALL_ROOT="$HOME/.cache/cmux/zig" bash scripts/install-zig-ci.sh`

2. **zig 0.15.2 vs the macOS 26 SDK.** Even with the right zig, the helper's build-runner fails to
   link libSystem on macOS 26 (undefined `_sigaction`, `_waitpid`, etc.). This is a zig/SDK problem,
   not a cmux or fork problem. For local dogfood builds, skip the real helper and build a stub (the
   terminal still works via GhosttyKit; only the standalone bundled `ghostty` CLI is a stub):
   ```bash
   CMUX_SKIP_ZIG_BUILD=1 ./scripts/reload.sh --tag <tag>
   ```
   A proper release build needs the real helper, which CI builds in a controlled environment. Do not
   ship a `CMUX_SKIP_ZIG_BUILD=1` build as a release.

## Notes

- The `ghostty` and `vendor/bonsplit` submodules track upstream. Our fork does not modify them, so
  the merge always takes upstream's newer pointer. No submodule conflict expected.
- `GhosttyKit.xcframework` is a symlink into `~/.cache/cmux/ghosttykit/<ghostty-sha>-.../`. Do not
  edit it by hand. `scripts/ensure-ghosttykit.sh` (called by `reload.sh`) points it at the right SHA.
- If a future upstream version renames or heavily rewrites `ContentView.swift`,
  `FileDropOverlayView.swift`, or `FileDropOverlayViewHitTesting.swift`, the merge conflict there may
  need real reintegration, not just union. That is the one case to slow down and read both sides.
