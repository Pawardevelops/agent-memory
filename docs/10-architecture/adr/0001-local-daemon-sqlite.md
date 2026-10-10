---
title: "ADR 0001: One local daemon with SQLite as the source of truth"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0001: One local daemon with SQLite as the source of truth

## Context
Sachin wants long-term memory for many agents on one Mac, fully local, fast, and with no UI. Expected volume is thousands to tens of thousands of short facts from one user.

## Decision
One always-on daemon process owns a single SQLite database (WAL) with FTS5, plus sqlite-vec from Phase 2. Clients never open the database directly.

## Alternatives considered
- **One process per agent (v1 design):** about 10 copies of the embed model and caches, and concurrent writers. Rejected once scope widened to any agent (see 0002).
- **Postgres + pgvector:** a server to run and back up, with no benefit at this scale.
- **A dedicated vector DB (Qdrant, Chroma):** a second store to keep consistent with forget, and still needs a keyword index.
- **Buy (mem0, basic-memory):** neither gives owner-only writes, versioning with forget-reverts and per-agent attribution together. Revisit if those needs drop.

## Consequences
- Simple backups (one file), zero ops, and reads in under a millisecond.
- A single writer limits write throughput. That's irrelevant for one human's memory.
- Moving to multi-user or remote access later means putting the same service API in front of Postgres. The schema carries over.
