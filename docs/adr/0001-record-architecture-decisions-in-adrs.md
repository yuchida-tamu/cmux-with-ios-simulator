# 1. Record architecture decisions in ADRs

Date: 2026-04-26

## Status

Accepted

## Context

We need a lightweight, version-controlled way to record significant architectural decisions so future contributors (human or agent) can understand the reasoning behind the current shape of the codebase.

## Decision

We will record architecture decisions here using the ADR format (Markdown Architectural Decision Records). Each ADR lives in `docs/adr/` and is numbered sequentially.

## Consequences

- Every non-obvious architectural decision gets a short written rationale.
- Agents and reviewers can grep `docs/adr/` before proposing divergent patterns.
- New decisions require a new ADR; supersessions are recorded explicitly.
