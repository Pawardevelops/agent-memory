---
title: "ADR 0007: Phased delivery: correctness first, then intelligence, then speed"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0007: Phased delivery: correctness first, then intelligence, then speed

## Context
The full design (daemon, hybrid retrieval, cache, integrations) is too large to review safely as one PR. Tech Lead and Engineering Guide recommended splitting it.

## Decision
Phase 0 is a spike. Phase 1 is core memory with FTS only. Phase 2 adds embeddings, hybrid ranking and evals. Phase 3 adds cache and operations. Phase 4 adds integrations and verified provenance. A phase starts only when the previous phase's invariants pass.

## Alternatives considered
- **Big-bang build:** high review risk, and correctness bugs get hidden behind ranking noise.

## Consequences
- Phase 1 recall misses paraphrases, and that is surfaced as a `keyword_only` warning.
- Each phase gets its own architecture snapshot (see ADR 0008).
