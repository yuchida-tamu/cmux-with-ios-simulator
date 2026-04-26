# 2. Fork-then-upstream-PR — never permanent fork

Date: 2026-04-26

## Status

Accepted

## Context

cmux (`manaflow-ai/cmux`) is a native Swift/AppKit application. Inspection of `Sources/Panels/Panel.swift` shows `enum PanelType { terminal, browser, markdown }` — a closed Swift enum — and a `@MainActor` `internal` `Panel` protocol. A mirror `enum CmuxSurfaceType` lives in `Sources/CmuxConfig.swift`. There is **no plugin/extension API**: no surface registry, no dylib loading, no JS scripting hook, no `plugins/` directory. The closest precedent (`AppIconDockTilePlugin.swift`) is an `NSDockTilePlugIn`, unrelated to surface types.

Adding a `SurfaceType.simulator` therefore requires editing the closed enum, the surface-type mirror, the `PanelContentView` SwiftUI dispatcher, the CLI argument parser, and the config schema — all in-tree changes to upstream cmux.

cmux is licensed GPL-3.0-or-later with a dual commercial license; `CONTRIBUTING.md` requires a CLA-like grant to Manaflow that allows commercial relicensing of contributions. The codebase is friendly to additive panel work because the `Panel` protocol cleanly abstracts panel behavior. There is no upstream issue requesting an iOS Simulator surface today (issue #2770 is unrelated CLI ergonomics work).

We considered three options:

1. **Permanent fork** — minimum upfront friction, maximum maintenance cost (must rebase against every upstream change indefinitely).
2. **Plugin/extension** — impossible: no extension API exists, and adding one is itself an upstream change requiring maintainer buy-in.
3. **Fork-then-upstream-PR** — prototype on a fork, propose the merged feature back to upstream. Slightly higher coordination cost (file an issue first, sign the CLA, follow upstream conventions), but the only path that yields a stable shipping artifact without forever rebasing.

## Decision

We will use the fork-then-upstream-PR model. Concretely:

- This repository (`yuchida-tamu/cmux-with-ios-simulator`) is a fork of `manaflow-ai/cmux` and is treated as a **prototype branch**, not a long-lived product.
- All architectural choices made in this fork must be defensible against upstream maintainers. When a decision could go two ways, prefer the option closer to upstream conventions (`Panel` protocol shape, file layout under `Sources/Panels/`, naming consistent with `terminal` / `browser` / `markdown`).
- New abstractions are added only when an upstream review would also accept them. No "internal-only" patterns that would have to be redone before the PR.
- Spike code that is not upstream-quality lives in clearly marked locations (e.g., `docs/spikes/`, separate non-merged branches) and is not allowed to leak into mainline directories.
- The workflow scaffold itself (`docs/PRD.md`, `docs/adr/`, `docs/glossary.md`, `.memory/`, project-overlay section in `CLAUDE.md`, `.claude/settings.json`) is project coordination state. It is NOT shipped to upstream — see [ADR-0003](./0003-graft-workflow-scaffold-on-fork.md) for the cleanup contract.
- M3 is "land an upstream PR." Until then, M1 and M2 are explicitly graded against the question: "Could this be reviewed by a Manaflow maintainer without major rework?"

## Consequences

- **Single source of truth on style.** When upstream cmux disagrees with our preference, upstream wins. We follow `Sources/Panels/Panel.swift` conventions even when ours feel cleaner, because the cost of arguing the change in PR review exceeds the local ergonomics gain.
- **Apple private SPI is firewalled.** All `CoreSimulator` / `FBSimulatorControl` / `IOSurface` private-header usage lives behind a single Swift module boundary (e.g., `Sources/SimulatorKit/`) so it can be feature-flagged off, swapped to a public API if Apple ever ships one, or stripped from the upstream PR if maintainers ask.
- **No cmux-core rewrites.** A change is only proposed if the same value couldn't be obtained additively. Refactors of unrelated panel code are out of scope.
- **Upstream feature-request issue is mandatory.** Before merging into the fork's `main`, an upstream issue describing the design (Option A → Option B), prior art (Radon IDE), and license/CLA acknowledgement is filed in `manaflow-ai/cmux`. The issue URL is recorded in `docs/PRD.md` under "Open Questions."
- **CI artifacts must stay clean.** Distributable builds may not link Apple private frameworks. Private SPI is local-dev only until M3 lands and maintainers decide how to gate it (compile flag, separate target, etc.).
- **Maintenance burden is bounded.** If upstream rejects the PR at M3, this ADR is superseded by an explicit "permanent fork" ADR with revised milestones; we do not silently drift into a permanent fork.
