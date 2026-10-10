---
title: "ADR 0005: In-process cache layer with generation invalidation; no Redis"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0005: In-process cache layer with generation invalidation; no Redis

## Context
Sachin asked for a cache layer. SQLite reads are already under a millisecond, and the real costs are embedding latency (20–60 ms, or 1–3 s cold) and repeated identical recalls.

## Decision
Use caches inside the daemon:
- C0: keep the Ollama model warm.
- C1: an in-memory LRU for query embeddings.
- C2: a persistent embedding cache keyed by hash.
- C3: a recall-result cache whose keys include a global write generation.
- C4: a scoped memory brief, explicit only.
- C5: SQLite pragmas.

Every write bumps the generation. Forget also deletes the C2 rows for its content.

## Alternatives considered
- **Redis or memcached:** a second service and a second copy of personal data that forget must purge, with no speed gain on one machine.
- **TTL-only invalidation:** would serve stale facts after a revise or forget, which is unacceptable.

## Consequences
- The cache is optional and must give identical results when off. This is tested.
- Correctness rules: no stale read after a write is acknowledged, and nothing forgotten survives in any layer.
