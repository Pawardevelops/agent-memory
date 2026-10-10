---
title: "ADR 0004: Topics with versions; forget is a hard delete that reverts"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0004: Topics with versions; forget is a hard delete that reverts

## Context
Sachin changes his mind over time. The latest conclusion must win, but 'forget the current discussion and keep the old one' must restore the previous conclusion.

## Decision
Memories are topics, each holding a version chain. Revising adds version n+1 as current and marks the previous one superseded. Forgetting an entry deletes its text, quote and every index row, and if it was current, promotes the highest remaining version. Forget can target an entry, a topic, a conversation, or {since, agent}. The database enforces at most one current version per topic.

## Alternatives considered
- **Overwrite in place:** loses 'what did I think before' and can't revert.
- **Soft delete (a hidden flag):** text survives, so 'forget' isn't honest.
- **Event-sourced log with replay:** more machinery than one user needs.

## Consequences
- History is available with `include_history` and `history`.
- Forgotten text survives only in backups until rotation (Phase 3). The user-facing wording says 'forgotten from active memory'.
