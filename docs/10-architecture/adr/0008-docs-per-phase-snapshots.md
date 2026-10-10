---
title: "ADR 0008: Documentation: living target, per-phase snapshots, append-only ADRs"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0008: Documentation: living target, per-phase snapshots, append-only ADRs

## Context
Sachin expects the architecture to change between phases and during optimisation and security work, and wants the documents structured for that.

## Decision
`OVERVIEW.md` is the living target. `phase-N/` folders are as-built snapshots, frozen at ship, and each new one opens with 'Changes since phase N-1'. Decisions are ADRs, which are never deleted, only superseded. Diagrams are Mermaid in markdown. Every doc has frontmatter with owner, status, applies_to and last_reviewed.

## Alternatives considered
- **Edit one DESIGN.md in place:** loses what each phase actually looked like.
- **Wiki or external tool:** drifts from the code and isn't reviewed in PRs.

## Consequences
- There's a little duplication between snapshots. That's accepted for clarity.
- DESIGN.md stays authoritative until its content has moved into the new files, then it becomes a pointer.
