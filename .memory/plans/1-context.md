# Context package for issue #1: Spike 1: simctl wrapper CLI (sanity baseline)

**Generated:** 2026-04-26T00:00:00Z
**Contract version:** 2

## Issue summary
Establish a working baseline by shelling out to `xcrun simctl` from a tiny Swift CLI wired through cmux's existing CLI socket. Proves we can lifecycle a named iOS Simulator from cmux without any private SPI.

**Acceptance criteria (verbatim):**
- Tiny Swift CLI shells out to `simctl` for boot/shutdown/install of one named device.
- Exposes a `cmux sim ...` subcommand wired through cmux's existing CLI socket (`CLI/cmux.swift`).
- End-to-end manual test: `cmux sim boot 'iPhone 15 Pro'` returns a UDID and the simulator appears in `xcrun simctl list`.
- Spike write-up landed in `docs/spikes/01-simctl-wrapper.md` summarizing what worked and what limitations motivate Spike 2.
- All upstream cmux tests still pass (upstream's check command).

## Linked issues
- #2 Spike 2: FBSimulatorControl framebuffer dump — open — downstream consumer; this spike must produce a UDID it can attach to.
- #3 Spike 3: render IOSurface to a Metal view — open — depends transitively on #2.
- #4 Wire SurfaceType.simulator into cmux fork — open — M1 acceptance moment that uses the device CLI bring-up flow.

## Relevant ADRs

### ADR-0002: Fork-then-upstream-PR (never permanent fork)
- All architectural choices in this fork must be defensible against upstream maintainers.
- Prefer upstream conventions (`Panel` protocol shape, file layout under `Sources/Panels/`, naming consistent with `terminal` / `browser` / `markdown`).
- New abstractions only when an upstream review would also accept them — no internal-only patterns.
- Spike code that is not upstream-quality lives in `docs/spikes/` and may not leak into mainline directories.
- Apple private SPI is firewalled — but this spike uses **`xcrun simctl`** (public CLI), so no SPI concerns yet. SPI starts in #2.

### ADR-0003: Graft workflow scaffold on the fork
- `docs/spikes/` is workflow-scaffold and stripped at M3 by `scripts/upstream-pr-prep.sh`.
- `.memory/plans/` is workflow-scaffold and stripped at M3.
- The CLI changes (`CLI/cmux.swift`) are NOT scaffold — they are upstream-bound code. Treat them with upstream-review-quality care.
- Adding a workflow-scaffold file the cleanup script can't reverse is a contract violation. `docs/spikes/01-simctl-wrapper.md` falls under the existing `docs/spikes/` directory cleanup rule, so no script change needed.

## Relevant PRD sections

From `docs/PRD.md` Goals/Milestones:
- "Boot and drive a headless `SimDevice` from cmux without spawning `Simulator.app`." (Goal — this spike does NOT achieve this; `simctl boot` spawns `com.apple.CoreSimulator.CoreSimulatorService` and an OS process tree, but `Simulator.app` UI does not auto-launch unless the user runs `open -a Simulator`. Document this distinction in the write-up.)
- M1: "sim-server Swift daemon links FBSimulatorControl, exposes IOSurface over Unix socket; cmux fork renders one device's framebuffer in a simulator panel; mouse/keyboard input forwarded; manual `cmux new-pane --type simulator --device 'iPhone 15 Pro'` works end-to-end." (This spike is the bottom rung — sanity-check that we can list/boot/shutdown a named device from a cmux-owned binary at all.)

## Glossary terms
- **Headless `SimDevice`** — A booted iOS Simulator instance that does NOT spawn `Simulator.app`. Same mechanism `idb` and `FBSimulatorControl` use. (This spike does NOT yet produce a true headless device — `simctl boot` boots through `CoreSimulatorService`. True headless via custom `SimDeviceSet` is a #2 concern. Note this in the spike write-up.)

## Symbol references in codebase

### `simctl` / `SimDevice`
- No existing references in Swift source — this is greenfield.
- `docs/glossary.md`, `docs/PRD.md` mention them in spec text only.

### `dispatchSubcommandHelp` (pre-socket help dispatch pattern)
- `CLI/cmux.swift:1756` — the gate that runs subcommand help WITHOUT connecting to the socket. Local subcommands (`feed`, `opencode install-hooks`, etc.) bypass the socket using this guard.

### Local-only subcommand handling pattern (no socket required)
- `CLI/cmux.swift:1855-1980` — `command == "opencode"` (with `install-hooks` / `uninstall-hooks` subs) is handled BEFORE `let client = SocketClient(...)` is constructed at line ~1985. This is the pattern `sim` should follow: the subcommand handler runs locally and never requires a running cmux app.
- `CLI/cmux.swift:1979` — `setup-hooks` / `uninstall-hooks` follow the same pre-socket pattern.

### Help registration (`usage()` and `dispatchSubcommandHelp`)
- `CLI/cmux.swift:17252` — top-level `usage()` text. New `sim` command must appear in the alphabetized command list here.
- `CLI/cmux.swift:7100-7200` — per-subcommand help blocks returned by `dispatchSubcommandHelp`. New `sim` block goes here so `cmux sim --help` works without a running cmux.

### `CLIError`
- `CLI/cmux.swift:14` — the conventional error type. `throw CLIError(message: "...")` for user-facing errors.

### Process-spawning prior art (for `xcrun simctl` shell-out)
- `CLI/cmux.swift` is ~17,400 lines; grep for `Process()` / `launchPath` / `executableURL` to find the existing pattern. The agent should reuse the same pattern (likely a small `runProcess(...)` helper or inline `Process` setup) rather than introducing a new helper module. See ADR-0002 — no new abstractions unless upstream-acceptable.

## Recent related PRs
- None — this fork has no merged PRs touching `CLI/cmux.swift` for simulator work yet. Issue #5 (`Add init-project workflow scaffold`) is the only merged change and is workflow-scaffold only.

## Issue comments
None.

## Library documentation

### `xcrun simctl` (Apple-bundled, no fetch needed)
The relevant subcommands are public and stable across recent Xcode versions:
- `xcrun simctl list devices --json` — enumerate devices; output is JSON with `devices` map keyed by runtime, each value an array of `{ udid, name, state, isAvailable, deviceTypeIdentifier, ... }`.
- `xcrun simctl boot <udid>` — boot a device (idempotent; returns 0 on already-booted in recent Xcode, but errors on older versions — handle both).
- `xcrun simctl shutdown <udid>` — shutdown a device.
- `xcrun simctl install <udid> <app-path>` — install an `.app` bundle.
- Lookup by name → UDID requires parsing `simctl list devices --json` and matching `name` (and optionally preferring `state == "Booted"` or the latest runtime).

Edge cases the spike should document:
- Multiple devices share a name across runtimes (e.g., "iPhone 15 Pro" exists for iOS 17.x and iOS 18.x). The acceptance test pins to the name only — pick latest runtime by default; document the tiebreaker rule.
- `isAvailable: false` devices (deprecated runtime) must be skipped.
- `simctl boot` can fail if `CoreSimulatorService` isn't running (rare on a developer machine; `xcrun simctl list` warm-starts it).

### `Foundation.Process` (Swift)
Standard library, no fetch needed. Pattern: set `executableURL = URL(fileURLWithPath: "/usr/bin/xcrun")`, `arguments = ["simctl", ...]`, capture stdout via `Pipe`, run synchronously with `run()` + `waitUntilExit()`. Check `terminationStatus`. Existing `CLI/cmux.swift` already spawns subprocesses elsewhere — match its pattern.

## Gathering notes
- The phrase "wired through cmux's existing CLI socket" in the issue is **ambiguous**. Two readings:
  1. The `sim` command itself routes over the cmux Unix socket (would require a backend handler in `Sources/`).
  2. The `sim` subcommand is added to the `cmux` CLI binary (which happens to use a socket for OTHER commands), but `sim` itself is local.
  Reading (2) matches the precedent (`opencode install-hooks`, `feed clear`, `feedback`, `themes` partially) and is consistent with the acceptance criterion that `cmux sim boot ...` returns a UDID — which doesn't need cmux running. **Recommend reading (2)** in the plan; if the agent disagrees, the plan-task skill should record the decision under "Open questions" and proceed.
- The PRD note that this spike does NOT produce a true headless device (since `simctl boot` boots via `CoreSimulatorService`, not a custom `SimDeviceSet`) is critical for the spike write-up — it's literally the limitation that motivates Spike #2.
- CLAUDE.md's "Test quality policy" forbids tests that only verify source text or grep patterns. For a CLI subcommand, the right test layer is either (a) a unit test that exercises argument parsing through a runtime seam, or (b) skip tests entirely and rely on the manual end-to-end check in the acceptance criterion. The plan-task skill must pick one and justify it.
- Project check command: per CLAUDE.md, full check is `xcodebuild -project GhosttyTabs.xcodeproj test`, but **CLAUDE.md explicitly says "Never run tests locally"** — they run via GitHub Actions / VM. The PostToolUse hook runs `swift build` for fast feedback. The agent should:
  1. Run `swift build` (fast, hook already does this).
  2. Optionally `xcodebuild -scheme cmux-unit ... build` to verify the unit-test target compiles (CLAUDE.md says this is "safe — no app launch").
  3. NOT run the full `xcodebuild ... test` locally.
  Document this in the plan's test-plan section.
- The CLI subcommand list in `usage()` is alphabetized by command group but not strictly. New entries seem to land near related commands. Place `sim` in a location that's defensible at upstream review time — likely a new line near `markdown` or `browser` (peer surface CLIs).
