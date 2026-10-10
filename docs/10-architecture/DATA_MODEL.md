---
title: "Data model"
owner: Architect
status: active
applies_to: all-phases
last_reviewed: 2026-10-10
---

# Data model

**SQLite is the single source of truth.** All indexes and caches can be rebuilt from it, except forgotten content, which is gone by design.

## Entity relationships

```mermaid
erDiagram
  AGENTS ||--o{ TOPICS : creates
  AGENTS ||--o{ ENTRIES : writes
  TOPICS ||--|{ ENTRIES : "has versions"
  ENTRIES ||--o| ENTRIES_FTS : "indexed by (P1)"
  ENTRIES ||--o| ENTRIES_VEC : "embedded as (P2)"
  AGENTS ||--o{ AUDIT_LOG : "acts in"

  AGENTS {
    text id PK "ulid"
    text name UK
    text client "claude-code|codex|buzz|cursor|claude-desktop|vscode|script"
    text token_hash "sha256, plaintext never stored"
    text created_at
    text revoked_at
  }
  TOPICS {
    text id PK
    text title
    text project
    text status "active|forgotten"
    text created_at
    text created_by_agent FK
  }
  ENTRIES {
    text id PK
    text topic_id FK
    int version "UNIQUE(topic_id, version)"
    text content "NULL once forgotten"
    text kind "free text"
    text tags "JSON array"
    text status "current|superseded|forgotten"
    text agent_id FK
    text agent_sub "P4 Buzz sub-identity"
    text provenance "attested (P1) | verified (P4)"
    text request_quote "NULL once forgotten"
    text conversation_id
    text buzz_channel_id "P4"
    text buzz_event_id "P4"
    text created_at
  }
  AUDIT_LOG {
    int id PK
    text ts
    text action "remember|revise|forget|agent_add|agent_revoke"
    text topic_id
    text entry_id
    text agent_id
    text provenance
  }
```

There are also virtual and auxiliary tables:
- `entries_fts`: FTS5 over `(content, tags, title)`, with `entry_id UNINDEXED`. Added in P1.
- `entries_vec`: sqlite-vec `vec0(entry_id, embedding float[D])`. `D` comes from the configured model. Added in P2.
- `embedding_cache`: `hash(model ‖ text) → vector`. No plaintext is stored. Added in P3.
- `meta`: `schema_version`, `write_generation` and `embed_model`.

## Invariants (enforced by the database, not by convention)

| # | Invariant | Enforcement |
|---|---|---|
| I1 | At most one `current` entry per topic | `CREATE UNIQUE INDEX one_current_per_topic ON entries(topic_id) WHERE status='current'` |
| I2 | Versions are unique and increase | `UNIQUE(topic_id, version)`, and the service computes `max+1` inside the writer transaction |
| I3 | Forgotten means no text | `CHECK (status <> 'forgotten' OR (content IS NULL AND request_quote IS NULL))` |
| I4 | Non-forgotten entries have text and provenance | `CHECK (status = 'forgotten' OR content IS NOT NULL)`, `CHECK (provenance IN ('attested','verified'))` |
| I5 | Status values are closed sets | `CHECK (status IN ('current','superseded','forgotten'))` on entries, and `('active','forgotten')` on topics |
| I6 | Audit never holds content | No content columns exist in `audit_log` |
| I7 | Index consistency | A forgotten entry has no FTS, vec or C2 row. This is asserted by tests and by `agent-memory doctor --deep` |

## Entry lifecycle

```mermaid
stateDiagram-v2
  [*] --> current: remember (v1) / revise (v n+1)
  current --> superseded: revise
  superseded --> current: forget of the current version<br/>promotes the highest remaining
  current --> forgotten: forget
  superseded --> forgotten: forget
  forgotten --> [*]: tombstone (IDs only)
```

## Time queries (`as_of`)
"What was true at time T" is the latest **non-forgotten** version per topic with `created_at <= T`. A forgotten version is never returned, even when it was current at T, because forgetting is retroactive by design.

## Migrations by phase

| File | Phase | Adds |
|---|---|---|
| `0001_init.sql` | P1 | agents, topics, entries (+ I1–I5), entries_fts, audit_log, meta |
| `0002_vectors.sql` | P2 | entries_vec, `meta.embed_model`, backfill queue table `embed_jobs` |
| `0003_cache.sql` | P3 | embedding_cache, recall statistics columns used by the brief |
| `0004_buzz.sql` | P4 | provenance `verified` path, agent_sub/buzz columns become used (they already exist from 0001) |

**Rules:**
- Migrations are forward-only and numbered.
- Never edit a migration once it has been applied.
- The runner records `schema_version` in `meta` and refuses to start against a newer schema than it knows.
- A changed embedding model needs `agent-memory reembed`, which rebuilds `entries_vec` from `content`.

## PRAGMAs
`journal_mode=WAL`, `synchronous=NORMAL`, `foreign_keys=ON`, `busy_timeout=5000`, `temp_store=MEMORY`. P3 adds `cache_size`/`mmap_size` tuning.
