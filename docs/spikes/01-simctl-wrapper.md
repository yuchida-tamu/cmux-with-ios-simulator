# Spike 1 — `simctl` wrapper CLI (sanity baseline)

**Issue:** [#1](../../issues/1) — _Spike 1: simctl wrapper CLI (sanity baseline)_
**Status:** Landed (M1)
**Date:** 2026-04-26

## Goal

Establish that the cmux CLI binary can lifecycle a named iOS Simulator (`boot`, `shutdown`, `install`) without using any Apple private SPI. This is the bottom rung of the iOS-Simulator-surface effort: it produces the UDID that Spike #2 (FBSimulatorControl framebuffer dump) will attach to.

## What shipped

A new `cmux sim` subcommand in `CLI/cmux.swift` with four operations:

| Subcommand                         | What it does                                                            |
| ---------------------------------- | ----------------------------------------------------------------------- |
| `cmux sim list`                    | Lists available simulator devices (name, state, UDID, runtime).         |
| `cmux sim boot <name\|udid>`       | Boots a device; prints the UDID on success. Idempotent (already-booted is success). |
| `cmux sim shutdown <name\|udid>`   | Shuts down a booted device.                                             |
| `cmux sim install <name\|udid> <app-path>` | Installs an `.app` bundle on the device.                        |

Implementation lives entirely inside `CLI/cmux.swift`:

- A pre-socket dispatch block (next to `feed clear` and `opencode install-hooks`) so `cmux sim ...` never opens the Unix socket.
- A small `SimctlDevice` projection of the `xcrun simctl list devices --json` output.
- Reuses the existing `CLIProcessRunner.runProcess` helper for shelling out — no new abstractions (per ADR-0002).
- Error reporting via the existing `CLIError(message:)` convention.

## What worked

- `xcrun simctl list devices --json` is a stable, JSON-parseable enumeration of every simulator device known to `CoreSimulatorService`. Parsing it with `JSONSerialization` is straightforward and the schema (`devices` map keyed by runtime identifier, each value an array of device dicts with `udid` / `name` / `state` / `isAvailable` / `deviceTypeIdentifier`) has been stable across Xcode 14 → 16.
- `simctl boot <udid>` is idempotent on recent Xcode (returns 0 when already booted). On older Xcode it returned non-zero with stderr `Unable to boot device in current state: Booted`. The wrapper treats both as success so the caller never has to care.
- The pre-socket dispatch pattern (used by `opencode install-hooks`, `feed clear`, `cursor install-hooks`, and now `sim`) means we get `cmux sim --help` working without a running cmux app, and we don't pay socket-connect latency for a subcommand that doesn't need a server.

## Limitations that motivate Spike #2

This spike does **not** produce a true headless `SimDevice`.

- `simctl boot` boots the device through `com.apple.CoreSimulatorService`, which spawns the standard `CoreSimulator` process tree. The `Simulator.app` UI does **not** auto-launch (it only appears if the user runs `open -a Simulator` separately), but the device is owned by the system-wide `CoreSimulatorService` instance, not by cmux.
- A truly headless `SimDevice` — the kind `idb` and `FBSimulatorControl` use to attach a framebuffer over `IOSurface` — requires constructing a custom `SimDeviceSet` from a private SPI. That's the boundary Spike #2 has to cross.
- Consequence for Spike #2: it will need to either (a) attach to the running `simctl`-booted device by UDID via FBSimulatorControl, or (b) bring up its own private-SPI-managed `SimDeviceSet`. Spike #2 picks the path; Spike #1's UDID remains useful in either case.

## Design choices recorded for review

### Device-name tiebreaker rule

Multiple simulator devices commonly share a name across runtimes (e.g., "iPhone 15 Pro" exists for iOS 17.x and iOS 18.x). When the user passes a name, the wrapper picks the device whose runtime identifier sorts last lexicographically — which, for Apple's `com.apple.CoreSimulator.SimRuntime.iOS-<MAJOR>-<MINOR>` naming, equals the latest iOS runtime.

This is a stable, no-network tiebreaker that does not require parsing version strings. If a user wants a different device, they can pass the UDID directly (the wrapper UDID-fast-paths the input).

`isAvailable: false` devices (deprecated runtime, missing data files) are filtered out before the tiebreaker.

### "Wired through cmux's existing CLI socket" — interpretation

The acceptance criterion's phrasing was ambiguous. Two readings:

1. The `sim` command itself routes over the cmux Unix socket.
2. The `sim` subcommand is added to the `cmux` CLI binary (which uses a socket for OTHER commands), but `sim` itself runs locally.

This spike implements reading **(2)**, matching the precedent of `opencode install-hooks`, `feed clear`, `feedback`, and `setup-hooks`, all of which run before `SocketClient(...)` is constructed (at `CLI/cmux.swift:~1984`). Reading (2) is also consistent with the acceptance test (`cmux sim boot 'iPhone 15 Pro'` returns a UDID), which doesn't need the cmux app to be running. If Manaflow upstream prefers reading (1) at PR review time, that's a follow-up — likely sized alongside the eventual `Sources/SimulatorKit/` module boundary in M2.

### No new abstractions

Per ADR-0002, no new helper module or abstraction was added. The wrapper:

- Reuses the existing `CLIProcessRunner.runProcess(executablePath:arguments:)` helper (defined at `CLI/cmux.swift:~1500`).
- Uses the existing `CLIError(message:)` error type (`CLI/cmux.swift:14`).
- Lives next to `runFeedClear`, matching the naming and placement of peer pre-socket subcommand handlers.

### No tests

Per CLAUDE.md "Test quality policy," no automated tests were added. Rationale:

- The CLI's only meaningful behavioral surface is "shells out to `xcrun simctl` and propagates results." A unit test that mocks `CLIProcessRunner.runProcess` would either re-test `JSONSerialization` (low value) or only assert dispatch-table shape (forbidden by the policy). 
- A real integration test would require an iOS Simulator runtime on the CI runner, which upstream cmux's CI doesn't provide.

The acceptance criterion explicitly calls for a manual end-to-end check, which is documented below.

## Manual verification

Run on a macOS host with Xcode 15+ installed and at least one iOS Simulator runtime downloaded.

```bash
# 0. Build the CLI binary
swift build

# Path to the built CLI:
CMUX_CLI=".build/debug/cmux"

# 1. Help works without a running cmux app
"$CMUX_CLI" sim --help

# 2. List available devices (sanity: should print a table with UDIDs)
"$CMUX_CLI" sim list

# 3. Boot a named device — should print a UDID
"$CMUX_CLI" sim boot 'iPhone 15 Pro'

# 4. Verify the device shows as Booted in Apple's own listing
xcrun simctl list devices | grep -i 'iPhone 15 Pro'
# Expected: a line containing "(Booted)"

# 5. Shut it back down
"$CMUX_CLI" sim shutdown 'iPhone 15 Pro'

# 6. Verify shutdown took effect
xcrun simctl list devices | grep -i 'iPhone 15 Pro'
# Expected: a line containing "(Shutdown)"

# 7. Idempotency check: booting twice in a row should both succeed
"$CMUX_CLI" sim boot 'iPhone 15 Pro'
"$CMUX_CLI" sim boot 'iPhone 15 Pro'
# Expected: both print the same UDID

# 8. Bad name produces a friendly error
"$CMUX_CLI" sim boot 'iPhone 999 Notreal'
# Expected: "No available simulator named 'iPhone 999 Notreal'. Run 'cmux sim list' to see available devices."
```

## Tested against

- macOS host with Xcode 15.x command-line tools (`xcrun simctl` from the Xcode 15 toolchain). The JSON schema and `simctl boot` idempotency behavior have been stable since Xcode 14, so this should also work on Xcode 14.x.
- Swift toolchain shipped with the same Xcode (Swift 5.9+).

## Code references to copy forward

For Spike #2:

- `resolveSimulatorDevice(nameOrUDID:)` in `CLI/cmux.swift` — name → UDID resolution with the latest-runtime tiebreaker. The Spike #2 framebuffer attach path will accept the same `<name|udid>` shape, so this resolver should be reused (move it into `Sources/SimulatorKit/` when that module is created in M2).
- `simctlListDevices()` in `CLI/cmux.swift` — JSON projection. Same; reuse from `SimulatorKit` once it exists.

## What to throw away

Nothing in this spike is throwaway. The CLI surface (`cmux sim list/boot/shutdown/install`) is the user-facing entry point we'll keep. When `Sources/SimulatorKit/` lands in M2 (Spike #2), the resolver and JSON projection should move into the module so the framebuffer code can call them, and `CLI/cmux.swift` should call into the module instead of duplicating the parser. Until then, keeping the wrapper inline in `CLI/cmux.swift` matches the upstream CLI's existing conventions.
