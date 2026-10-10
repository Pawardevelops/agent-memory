---
title: "ADR 0003: Owner-only writes through provenance levels"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0003: Owner-only writes through provenance levels

## Context
Agents may write only when Sachin explicitly asks. A Buzz request is a signed Nostr event that can be verified. A request typed in Claude Code, Codex or Cursor has no verifiable proof.

## Decision
Every write carries `request{quote, conversation_id, buzz?}`. If `buzz` is present, the daemon fetches the event and verifies its id, its signature and that the author is the owner's pubkey; the entry is stored as **verified** (Phase 4). Otherwise a non-empty quote is required and the entry is stored as **attested** (Phase 1). Recall always shows the provenance. Sachin chose to save attested writes immediately, with no approval queue.

## Alternatives considered
- **Verified only:** would block every non-Buzz tool.
- **Pending queue with manual approval:** safer, but adds friction where Sachin himself is the one typing.
- **Trust prompts only:** not enforceable.

## Consequences
- Attested writes depend on the agent behaving honestly. They are mitigated by labelling, the audit log, reversibility and the server instructions.
- The verified path needs relay access at write time. If the relay is down, the write fails with an error rather than being silently downgraded.
