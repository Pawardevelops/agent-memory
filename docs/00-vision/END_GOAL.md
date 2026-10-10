---
title: "End goal"
owner: Engineering Guide
status: draft
applies_to: all-phases
last_reviewed: 2026-10-10
---

# End goal

Build a local memory system that any AI agent on Sachin's Mac can use to save, revise, recall and forget long-term context when Sachin explicitly asks it to remember something.

The system should make agents more useful without silently collecting everything. It must preserve provenance, support changed decisions over time, and make forgotten active memory unavailable to future recall.

## Problem
Current agent memory is fragmented:
- Short always-loaded memory is too small for long-term project facts.
- Cold memory is hard to discover because retrieval depends on exact names or manual context.
- Conversation history is not a reliable memory store because agents may not have the right thread, channel or session context.
- Different agents can make different assumptions because they do not share a consistent local memory layer.

The missing capability is **controlled retrieval**: agents need a local, searchable, versioned memory store that answers "what did Sachin ask us to remember?" without guessing.

## Product principles
1. **Explicit capture only.** Agents write memory only when Sachin says "remember", "keep in mind", "note this" or a similar instruction.
2. **Evidence before answer.** Agents call recall before answering questions about saved preferences, decisions or facts, then cite the saved memory or say nothing is saved.
3. **One current truth per topic.** A changed decision creates a new version and supersedes the previous one. History remains available unless forgotten.
4. **Forget must remove active memory.** Forget removes content from active rows, FTS, vector indexes and caches. Backups retain old content only until their configured rotation.
5. **Local-first.** The daemon, database and embeddings run locally on Sachin's Mac. No cloud embedding calls are part of the default design.
6. **Agent-agnostic.** Codex, Claude Code, Buzz agents and future clients use the same server contract.
7. **Inference-aware retrieval.** Recall returns ranked evidence with provenance, confidence and warnings, not final answer prose.
8. **Small, reviewable phases.** Correctness and deletion semantics come before embeddings, cache optimization and broad integrations.

## Primary users
- **Sachin:** controls what gets remembered and forgotten.
- **Local agents:** PA, Codex, Claude Code, Buzz agents and future local clients that need a shared memory interface.
- **Reviewers:** verify that recall, revise, forget, security and release behavior are correct before rollout.

## Phase 1 user outcome
After Phase 1, a local agent can:
1. Save an attested memory when Sachin explicitly asks.
2. Recall current memories using keyword search.
3. Revise a saved topic so the latest decision becomes current.
4. Forget a current version and restore the previous version when one exists.
5. Show history for a topic.
6. Prove through tests that active forgotten content does not appear in FTS recall.

Phase 1 intentionally does not optimize semantic retrieval. It proves that the memory state model is correct.

## Non-goals for the first release
- No UI.
- No mobile or remote access.
- No multi-user sharing.
- No automatic capture of conversations without explicit instruction.
- No cloud-only agent support.
- No external cache such as Redis.
- No password-manager behavior. Secrets should be rejected, not stored.
- No Phase 1 dependency on embeddings or `sqlite-vec`.

## Success measures
- PA can save a verified Buzz memory once verified provenance is implemented.
- Codex or Claude Code can recall the same memory with provenance.
- A later correction creates a new current version.
- Forgetting the current version restores the previous version when one exists.
- Forgotten active content does not appear in recall, FTS, vector indexes, memory brief or caches for the phases where those layers exist.
- Recall returns useful ranking and confidence signals.
- The local install can be diagnosed, backed up and rolled back.

## Product boundary
agent-memory is not a replacement for project management, source control, issue tracking, password storage or document storage. It is a local agent memory substrate: small facts, decisions, preferences and context that agents can retrieve as evidence.
