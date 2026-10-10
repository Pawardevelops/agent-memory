---
title: "Technology stack and skills"
owner: Tech Lead
status: active
applies_to: all-phases
last_reviewed: 2026-10-10
---

# Technology stack and skills

This project is a local inference infrastructure component. The stack is intentionally boring at the persistence layer and advanced at the retrieval/evaluation layer.

## Core stack
| Technology | Phase | Why |
|---|---:|---|
| Python 3.13 | 0 | Fast implementation, strong SQLite ecosystem, good MCP support |
| uv | 0 | Reproducible local Python workflow and fast dependency management |
| SQLite | 1 | One local user, single durable file, simple backup and migration story |
| SQLite WAL mode | 1 | Concurrent reads with controlled writes |
| FTS5 | 1 | Built-in keyword recall without external services |
| MCP SDK | 1 | Native agent tool interface |
| REST JSON | 1 | Simple non-MCP integration and test surface |
| Starlette/Uvicorn | 1 | Local daemon HTTP runtime |
| httpx | 1 | Local client calls from stdio shim and tests |
| pytest | 1 | Unit, integration and regression tests |
| sqlite-vec | 2 | Local vector search inside SQLite |
| Ollama | 2 | Local embedding runtime with no cloud dependency |
| nomic-embed-text | 2 | Default local embedding model |
| coincurve | 4 | Buzz/Nostr Schnorr signature verification |
| launchd | 3/4 | macOS service lifecycle |

## Advanced engineering techniques
- **Versioned memory:** memories are topic versions, not mutable blobs.
- **Hybrid retrieval:** FTS and vector candidates are merged with reciprocal rank fusion.
- **Confidence scoring:** recall returns quality signals so agents can avoid overclaiming.
- **Context budgeting:** long results are bounded without losing IDs or provenance.
- **Active deletion:** forget removes content from active rows, FTS, vector index and caches.
- **Generation invalidation:** every write invalidates result and brief caches.
- **Deterministic evals:** retrieval behaviour is tested with fixed fixtures.
- **Degraded mode:** keyword recall remains available when embedding is down.
- **Local provenance:** verified Buzz writes and attested local writes are labelled separately.

## Skills contributors need
| Skill | Needed for |
|---|---|
| SQLite schema design | topics, entries, FTS, migrations, indexes |
| Transaction design | revise/forget atomicity and one-current invariant |
| Local daemon design | lifecycle, localhost binding, operational safety |
| MCP tool design | agent-facing interface and stdio compatibility |
| REST API design | predictable request/response contracts |
| Retrieval engineering | ranking, confidence, evals, false-positive control |
| Embedding integration | Ollama, vector dimensions, backfill and fallback |
| Security engineering | tokens, sensitive data, logs, file permissions |
| macOS operations | launchd, local file paths, backups |
| Testing | persistence, concurrency, deletion and cross-transport parity |

## Phase-by-phase technology adoption
```mermaid
flowchart LR
  P0[Phase 0<br/>spikes] --> P1[Phase 1<br/>SQLite + FTS + REST/stdio]
  P1 --> P2[Phase 2<br/>Ollama + sqlite-vec + evals]
  P2 --> P3[Phase 3<br/>cache + metrics + backup]
  P3 --> P4[Phase 4<br/>client integrations + Buzz verification]
```

## What we avoid for now
| Avoided technology | Reason |
|---|---|
| Redis | Extra service and extra deletion surface with little local benefit |
| Postgres/pgvector | Too heavy for one local owner |
| Cloud embeddings | Text must stay local |
| Hosted memory services | Harder to enforce explicit writes and forget semantics locally |
| UI framework | No UI in the approved scope |
| Multi-user auth stack | Same-user local attribution is enough for v1 |

## Stack risks to prove early
- Python build can load SQLite extensions required by `sqlite-vec`.
- Ollama model dimension matches the declared vector table dimension.
- MCP stdio and HTTP support works in Codex and Claude Code.
- launchd can start the daemon reliably with the expected environment.
- Client config paths can be updated safely with dry-run output.
