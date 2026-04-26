# 3. Graft workflow scaffold onto the fork (single repo)

Date: 2026-04-26

## Status

Accepted (final — supersedes two earlier oscillations between this design and a meta-repo split)

## Context

This project produces two kinds of artifact:

1. **Project-coordination state** — PRD, ADRs, glossary, spike write-ups, per-task plans, GitHub issues, project board.
2. **Swift/Obj-C++ code changes** — additions to the cmux source tree (`Sources/Panels/`, `CmuxConfig`, `CLI/`, plus a new `Sources/SimulatorKit/`).

Two layouts are technically viable:

- **A. Two repos.** A meta repo holds (1); a separate fork of `manaflow-ai/cmux` holds (2). Issues live in the meta repo; PRs live in the fork.
- **B. One repo (graft).** One fork of `manaflow-ai/cmux`. Workflow scaffold sits next to the cmux source under `docs/`, `.memory/`, `CLAUDE.md` overlay, etc. Issues AND PRs live on the same repo.

We attempted A (meta repo) and reverted. We attempted B (graft) and reverted. We are now choosing B again, this time deliberately — and recording the reasoning so we don't oscillate again.

### Why B (graft) wins

1. **`@claude` PR review fires in the right place.** Claude Code's GitHub workflow (`.github/workflows/claude.yml`) is enabled per-repo and reviews PRs on that repo. With layout B, code PRs and `@claude` review live together. With layout A, the meta repo's `@claude` install is wasted (no code PRs there) and the fork repo needs its own install — fragmenting setup, secrets, and review history.
2. **One git history per change.** A spike write-up under `docs/spikes/03-iosurface-metal.md` and the Swift code it describes land in the same PR. Bisecting, blaming, and code review all see one diff. Layout A forces a cross-repo dance: open meta-repo issue → write fork-repo PR → write meta-repo PR for the spike doc → cross-link the three.
3. **Single working directory.** No "did you `cd` into the right repo" foot-gun. `exec-tasks` agents read the issue, the plan, and edit the code in one checkout.
4. **Simpler permissions and secrets.** One install of the GitHub App, one `CLAUDE_CODE_OAUTH_TOKEN`, one set of Actions secrets.

### Cost of B that A would have avoided

- The eventual upstream PR at M3 contains workflow-scaffold files (`docs/PRD.md`, `docs/adr/0001..0003`, `docs/glossary.md`, `.memory/`, the project-overlay section in `CLAUDE.md`, `.claude/settings.json` enabledPlugins, the workflow-only `.gitignore` lines) that upstream cmux maintainers don't want. We pay this cost as a **scripted cleanup step** in the M3 PR-prep flow, not as ongoing friction.
- PRD/ADR edits live alongside a large source tree (Swift, Ghostty submodule, marketing site). A bit noisier in the file tree, but cheap enough.

## Decision

We use layout B: a single fork of `manaflow-ai/cmux` with the workflow scaffold grafted on top. Concretely:

- **Repo:** `yuchida-tamu/cmux-with-ios-simulator` (fork of `manaflow-ai/cmux`).
- **Workflow-scaffold files in this fork** (the set that must be removed before the upstream PR):
  - `docs/PRD.md`
  - `docs/glossary.md`
  - `docs/adr/0001-record-architecture-decisions-in-adrs.md`
  - `docs/adr/0002-fork-then-upstream-pr.md`
  - `docs/adr/0003-graft-workflow-scaffold-on-fork.md` (this file)
  - `docs/spikes/` (entire directory)
  - `.memory/` (entire directory)
  - `.claude/settings.json` (only our additions — the `enabledPlugins` block; preserve any upstream additions)
  - The "Project: iOS Simulator Surface (this fork's overlay)" section appended to `CLAUDE.md` (and to the symlinked `AGENTS.md` it shadows)
  - The workflow-only block appended to `.gitignore` (`.init-project-state.json`, `.claude/settings.local.json`, `.memory/work/`)
- **Files we do NOT add or overwrite** (upstream owns them): `LICENSE` (GPL-3.0), `README.*`, `.github/workflows/claude.yml`, `.github/workflows/ci.yml`, `.github/workflows/*` in general. We rely on upstream's CI for the project check.
- **PostToolUse hook.** `.claude/settings.json` enables a `swift build` hook on `Edit|Write|MultiEdit` — fast subset of upstream's `xcodebuild test` check. Configurable: comment out or downgrade if it becomes painful (see PRD Open Questions).

### M3 cleanup script

Before the upstream PR is opened, the cleanup script (TBD: lives at `scripts/upstream-pr-prep.sh`, written when first needed) MUST:

1. Delete all paths in the "Workflow-scaffold files" list above.
2. Strip the project-overlay section from `CLAUDE.md` (everything below the `## Project: iOS Simulator Surface (this fork's overlay)` heading).
3. Strip the workflow-only `.gitignore` block.
4. Open a clean PR branch from the fork's `main` containing ONLY the cmux source changes.
5. Verify with `git diff manaflow-ai/cmux/main` that nothing scaffold-related remains.

The script is the contract. If it can't cleanly strip a file, that file shouldn't be added under workflow scaffolding in the first place.

## Consequences

- **Workflow plugins stay enabled in this repo.** `.claude/settings.json` lists `init-project`, `exec-tasks`, `post-session`. Anyone who clones the fork and opens it in Claude Code gets the workflow on first run.
- **`@claude` PR review available everywhere PRs happen.** Both feature-branch PRs against this fork's `main` and the eventual upstream PR (after cleanup) are reviewable by Claude in this single repo configuration.
- **No cross-repo bookkeeping.** Issues and code PRs share a tracker. Spike write-ups, plans, and code share one git history.
- **Cleanup discipline is mandatory.** Adding a workflow-scaffold file that the cleanup script can't reverse is a contract violation. Either extend the script or don't add the file.
- **Upstream PR diff stays clean.** When M3 opens, the diff against `manaflow-ai/cmux:main` contains only cmux source changes — no PRD, no ADRs, no `.memory/` artifacts, no project-overlay text in `CLAUDE.md`.
- **Meta-repo split remains an emergency option.** If the cleanup script becomes unmaintainable, we can split out the workflow scaffold into a separate meta repo and revisit. ADR-0003 is superseded only by an explicit replacement ADR — not by silent drift.
