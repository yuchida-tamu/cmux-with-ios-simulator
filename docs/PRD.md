# cmux-with-ios-simulator — PRD

## Overview

A `SurfaceType.simulator` for cmux that renders a headless iOS Simulator framebuffer (`IOSurface` → `CAMetalLayer`) directly inside a cmux pane, alongside the existing terminal/browser/markdown surfaces. Built for cmux users who want native, low-latency iOS dev workflows without a separate `Simulator.app` window. Prototyped on this fork of `manaflow-ai/cmux`; the feature targets an upstream PR at M3 (see [ADR-0002](./adr/0002-fork-then-upstream-pr.md)). Workflow scaffolding (PRD, ADRs, etc.) lives in-tree on this fork and is removed before the upstream PR (see [ADR-0003](./adr/0003-graft-workflow-scaffold-on-fork.md)).

## Goals

- Boot and drive a headless `SimDevice` from cmux without spawning `Simulator.app`.
- Render the simulator framebuffer into a Metal-backed cmux pane at 60fps with no JPEG/HTTP indirection.
- Forward mouse, keyboard, and (eventually) multitouch + hardware buttons (Home, AppSwitcher) into the simulator.
- Land the feature upstream in `manaflow-ai/cmux` as `SurfaceType.simulator` (M3).

## Non-Goals

- Not a re-implementation of `Simulator.app`. We do not aim to replace Xcode's simulator UI for general developers.
- Not a cross-platform tool. macOS only — no Linux/Windows builds, no Android emulator support (Radon's Android path is out of scope).
- Not a JPEG/HTTP streaming pipeline. We explicitly skip Radon's MJPEG indirection because cmux is native.
- Not a public/stable API surface. We rely on Apple private headers (`CoreSimulator`, `FBSimulatorControl`) and may break with Xcode updates.
- Not a standalone product. Only ships as a feature of cmux; no separate distribution.

## Users

Existing cmux users who develop iOS apps. They already use cmux for terminal/browser/markdown panes; they want a simulator pane that fits the same pane/surface model rather than alt-tabbing to `Simulator.app`.

## Core Loop

User opens a cmux pane with `cmux new-pane --type simulator --device 'iPhone 15 Pro'`, sees the live booted iOS Simulator framebuffer rendered into the pane, and interacts with it via mouse/keyboard/multitouch the same way they'd use any other cmux surface — alongside their terminal, browser, and markdown panes.

## Milestones

- **M1 (MVP — validate the pipeline):** Option A working. `sim-server` Swift daemon links `FBSimulatorControl`, exposes `IOSurface` over Unix socket; `simulator` panel renders one device's framebuffer; mouse/keyboard input forwarded; manual `cmux new-pane --type simulator --device 'iPhone 15 Pro'` works end-to-end.
- **M2 (Polish — collapse to in-process):** Move `CoreSimulator` linkage in-process (Option B). Graceful `CoreSimulatorService` death recovery, multitouch + hardware buttons (Home/AppSwitcher), Xcode version sniffing, no-cmux integration test harness.
- **M3 (Ship — upstream):** Open upstream feature-request issue, address maintainer feedback, sign Manaflow CLA, run the workflow-scaffold cleanup script (ADR-0003), land PR into `manaflow-ai/cmux` so `SurfaceType.simulator` exists in mainline.

## Open Questions

- **Riskiest unknown:** Apple private SPI (`CoreSimulator`, `FBSimulatorControl`, `SimDisplayIOSurfaceRenderable`) breaking or being walled off in an Xcode update — the entire pipeline depends on it. Secondary: `IOSurface` IPC across the daemon ↔ cmux process boundary hitting sandbox/entitlement walls in Option A. Tertiary: codesigning / hardened-runtime rejecting linkage against private frameworks at distribution time.
- **Upstream signal:** No existing maintainer signal for a `simulator` surface. Issue #2770 in `manaflow-ai/cmux` is unrelated (CLI pane-reuse ergonomics). Whether maintainers will accept the upstream PR is unproven; record the upstream issue URL here once filed (M3).
- **Xcode version coverage:** Which Xcode versions must the SPI shim support? Initial target: latest stable + previous minor. Decide at M2.
- **Hook responsiveness:** PostToolUse hook runs `swift build`. If incremental builds become painful as `Sources/SimulatorKit/` grows, downgrade to `swift build --target SimulatorKit` or comment out the hook entirely. Decide at M2 review.

## Reference material

- cmux: <https://github.com/manaflow-ai/cmux>
- cmux feature request — pane-typed surfaces (CLI ergonomics, not surface API): <https://github.com/manaflow-ai/cmux/issues/2770>
- Radon IDE: <https://github.com/software-mansion/radon-ide>
- Radon IDE — pane mode docs: <https://ide.swmansion.com/docs/getting-started/panel-mode>
- idb (`FBSimulatorControl` source): <https://github.com/facebook/idb> (`fbsimctl/FBSimulatorControl/`)
- idb private CoreSimulator overview: <https://fbidb.io/docs/coresimulator/>
- Public Radon `sim-server` MJPEG architecture (prior art): Radon IDE issues #2770, #528, #645
