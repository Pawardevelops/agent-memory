---
title: "Target architecture (all phases)"
owner: Architect
status: active
applies_to: all-phases
last_reviewed: 2026-10-10
---

# Target architecture

This is the end-state design that all phases build towards. What each phase actually ships is in `phase-N/ARCHITECTURE.md`. This file changes only through an ADR.

## 1. System context

```mermaid
flowchart TB
  S(["Sachin"])
  subgraph Mac["Sachin's Mac (single macOS user)"]
    direction TB
    subgraph Clients["AI clients"]
      CC(["Claude Code"])
      CX(["Codex"])
      BZ(["Buzz agents<br/>PA · Architect · Coder …"])
      CU(["Cursor / Claude Desktop / VS Code"])
      SC(["Scripts · n8n"])
    end
    AM["agent-memory daemon<br/>127.0.0.1:7077"]
    OL{{"Ollama<br/>local embeddings"}}
    DB[(SQLite<br/>~/.buzz/agent-memory)]
    BK[(Backups<br/>14 days)]
  end
  RELAY{{"Buzz relay<br/>signed events"}}

  S -- "keep in mind…" --> Clients
  Clients -- MCP / REST + token --> AM
  AM --> DB
  AM --> OL
  AM -- verify owner event --> RELAY
  DB -. nightly .-> BK
```

**Boundaries**
- Nothing listens beyond `127.0.0.1`. The only outbound call is to the Buzz relay, and only to verify a Buzz request. Text never leaves the Mac.
- The daemon is the only process that opens the database.

## 2. Containers and components

```mermaid
flowchart LR
  subgraph Adapters["Transport adapters"]
    SHIM[stdio MCP shim<br/>agent-memory stdio]
    MCPH[/mcp<br/>MCP Streamable HTTP/]
    REST[/v1<br/>REST JSON/]
    CLI[CLI]
  end
  subgraph Daemon["agent-memory daemon (one asyncio process)"]
    AUTH[Auth<br/>token → agent]
    PROV[Provenance<br/>buzz verifier · attested]
    SEC[Secret filter]
    CORE[Memory core<br/>topics · versions · forget]
    RET[Retrieval<br/>FTS5 + vector + RRF<br/>confidence · budget]
    CACHE["Cache<br/>C1–C4 + generation"]:::cache
    EMB[Embedder client<br/>+ backfill queue]
    AUD[Audit log]
    OBS[Metrics · stats]
  end
  DB[(SQLite WAL<br/>FTS5 · sqlite-vec)]
  OL{{"Ollama"}}

  SHIM -->|HTTP| REST
  CLI --> REST
  MCPH --> AUTH
  REST --> AUTH
  AUTH --> PROV --> SEC --> CORE
  AUTH --> RET
  CORE --> AUD
  CORE -->|bump generation| CACHE
  RET <--> CACHE
  RET --> DB
  CORE --> DB
  CORE --> EMB
  RET --> EMB
  EMB --> OL
  RET --> OBS
  classDef cache stroke-dasharray: 4 3
```

**Layering rule:** adapters → application services → core → storage. Adapters only translate, and all rules live in the core. MCP and REST are two views of one service API, so they can't drift apart (see ADR 0006).

## 3. Key flows

### 3.1 Remember (write)

```mermaid
sequenceDiagram
  autonumber
  actor S as Sachin
  participant A as Agent
  participant D as Daemon
  participant R as Buzz relay
  participant DB as SQLite
  S->>A: "Keep in mind: X"
  A->>D: remember(content, request{quote, conversation_id, buzz?}) + token
  D->>D: token → agent identity
  alt request.buzz present
    D->>R: fetch event
    R-->>D: signed event
    D->>D: verify id, sig, author == owner → verified
  else
    D->>D: quote required → attested
  end
  D->>D: secret filter (content + quote)
  D->>DB: duplicate check (search current topics)
  alt similar topic exists and not force_new
    D-->>A: possible_duplicate [candidates]
    A->>S: "Update existing or keep both?"
  else
    D->>DB: one transaction: topic + entry v1 current, FTS, audit, generation++
    D-->>A: saved {topic_id, entry_id, provenance}
    A->>S: "Saved: X"
  end
```

### 3.2 Recall (read)

```mermaid
sequenceDiagram
  autonumber
  participant A as Agent
  participant D as Daemon
  participant C as Cache
  participant DB as SQLite
  participant O as Ollama
  A->>D: recall(query, project?, as_of?, max_chars?)
  D->>C: C3 lookup (key includes generation)
  alt hit
    C-->>D: results
  else miss
    par keyword
      D->>DB: FTS5 bm25 top-20
    and meaning
      D->>C: C1/C2 query vector?
      opt miss
        D->>O: embed(query)
      end
      D->>DB: sqlite-vec KNN top-20
    end
    D->>D: RRF merge → filter status/as_of → collapse versions → confidence → budget
    D->>C: store C3
  end
  D-->>A: evidence[] + confidence + warnings
  A-->>A: answer with citation, or "nothing saved"
```

### 3.3 Change of mind, then forget (versions)

```mermaid
sequenceDiagram
  autonumber
  actor S as Sachin
  participant A as Agent
  participant D as Daemon
  S->>A: Mon: "Keep in mind: launch on 1 Nov"
  A->>D: remember → topic T, v1 current
  S->>A: Tue: "Changed my mind: 15 Nov"
  A->>D: recall("launch") → T v1
  A->>D: revise(T, "15 Nov") → v2 current, v1 superseded
  S->>A: Wed: "Forget today's change, keep the old one"
  A->>D: forget(entry v2)
  D->>D: hard-delete v2 text/index/vectors, promote v1 → current, generation++
  D-->>A: forgotten v2, current = v1 ("1 Nov")
```

## 4. Quality attributes and how the design meets them

| Attribute | Mechanism |
|---|---|
| Correctness of "what is true now" | DB-enforced one-current-per-topic, transactional revise and forget |
| Owner-only writes | Provenance check in the daemon. Buzz-signed requests are verified. Requests from other tools are labelled `attested` (ADR 0003) |
| Real forgetting | Hard delete of content, quote, FTS, vector and C2 rows, plus a generation bump that invalidates C3 and C4 |
| Low latency | One process, a warm model, the cache layer, and SQLite read in-process (ADR 0005) |
| Any client | One service API exposed as MCP stdio, MCP HTTP and REST, with per-client config presets (ADR 0002, 0006) |
| Degrades safely | Ollama down → keyword-only recall with a warning, and writes still succeed with the embedding backfilled |
| Changeable | Ports for embedder, provenance verifier and transport. Numbered migrations. Phase snapshots and ADRs |

## 5. Phase map

```mermaid
flowchart LR
  P0[Phase 0<br/>Spike<br/>sqlite-vec · Ollama · MCP · launchd] --> P1[Phase 1<br/>Core memory<br/>daemon · REST · shim · FTS<br/>versions · forget · tokens]
  P1 --> P2[Phase 2<br/>Inference quality<br/>embeddings · hybrid RRF<br/>dedupe · confidence · evals]
  P2 --> P3[Phase 3<br/>Cache & ops<br/>C0–C4 · stats · backups<br/>export · reembed]
  P3 --> P4[Phase 4<br/>Integrations<br/>Claude Code · Codex · Buzz<br/>verified provenance · /mcp HTTP]
  P4 --> L[Later<br/>Cursor · Desktop · VS Code<br/>UI? · remote access?]
```

| Component | P1 | P2 | P3 | P4 |
|---|---|---|---|---|
| Daemon + REST `/v1` | ✅ | | | |
| stdio MCP shim | ✅ | | | |
| MCP Streamable HTTP `/mcp` | | | | ✅ |
| Token registry and attribution | ✅ | | | Buzz sub-identity |
| Provenance | attested | | | + verified (Buzz) |
| Topics, versions, forget, audit | ✅ | | | |
| FTS5 recall + confidence | ✅ | | | |
| Embeddings + hybrid RRF + dedupe | | ✅ | | |
| Evals (`agent-memory eval`) | FTS cases | ✅ hybrid | | |
| Cache C0–C4, stats | | | ✅ | |
| Backups, export, reembed | | | ✅ | |
| Client presets and snippets | | | | ✅ |

## 6. Target details not yet in a phase snapshot

These parts arrive in Phases 3–4. Each one moves into `phase-3/` or `phase-4/ARCHITECTURE.md` when that phase is built. Until then, this section is the reference. It was migrated from DESIGN.md v3.

### 6.1 Cache layer (Phase 3, ADR 0005)

| Layer | What | Where | Invalidation |
|---|---|---|---|
| C0 Model warm | Embed model kept loaded (`keep_alive: -1`), warm-up at daemon start | Ollama | n/a |
| C1 Query-embedding LRU | `hash(model, normalized query)` → vector, 2,000 entries | daemon memory | model change |
| C2 Embedding cache | `hash(model, text)` → vector for content and queries, survives restarts | `embedding_cache` table, pruned to 50k rows / 30 days unused | model change; **content rows deleted on forget** |
| C3 Recall results | `(query, filters, as_of, include_history, limit, max_chars, generation)` → results, 500 entries, TTL 10 min | daemon memory | any write bumps `write_generation` |
| C4 Memory brief | Scoped summary ≤ 2 KB per project, **explicit `brief` call only** | memory, lazily rebuilt | generation change |
| C5 SQLite | `cache_size=-65536`, `mmap_size=256MB`, `temp_store=MEMORY` | SQLite | automatic |

```mermaid
sequenceDiagram
  autonumber
  participant A as Agent
  participant D as Daemon
  participant C3 as C3 result cache
  participant DB as SQLite
  A->>D: recall("launch date")
  D->>C3: get(key + gen 41)
  C3-->>D: hit (v1 "1 Nov")
  A->>D: forget(v1)
  D->>DB: delete text, FTS, vec and C2 rows, gen = 42
  A->>D: recall("launch date")
  D->>C3: get(key + gen 42)
  C3-->>D: miss
  D->>DB: search → nothing current
  D-->>A: confidence none
```

**Cache invariants** (each is a release-blocking test):
1. Forget is total: nothing forgotten can come back from any layer.
2. No stale read after a write is acknowledged: the generation is bumped in the commit path, before the response.
3. Cache off (`AGENT_MEMORY_CACHE=off`) gives identical results.

Metrics: hits and misses per layer, plus recall p50/p95, shown by `agent-memory stats`.

### 6.2 Buzz sub-identity and verified provenance (Phase 4, ADR 0002/0003)
- **One token, many agents:** all Buzz agents run on Claude Code through `claude-agent-acp` and share one MCP config, so a single token can't tell PA from Coder.
- **How the shim identifies each agent:** it reads `BUZZ_ACP_DISPLAY_NAME` and `BUZZ_PRIVATE_KEY` from its environment and sends `X-Agent-Sub`. That header carries the display name, the pubkey, and a Schnorr signature over a nonce issued by the daemon. The daemon stores the result as `agent_sub` under the `buzz` agent.
- **Verified provenance:** when a request includes `request.buzz{channel_id, event_id}`, the daemon fetches the event (`buzz messages thread --channel … --event …`). It then verifies the NIP-01 id and the Schnorr signature, and checks that `pubkey == OWNER_PUBKEY`. On success the provenance is `verified`. On failure it returns `not_requested_by_owner`, `bad_signature` or `event_not_found`, and nothing is written.

### 6.3 Client registration and rollout (Phases 3–4)
1. Install the prerequisites: `brew install ollama uv && ollama pull nomic-embed-text`. Then run `agent-memory install-launchd` (KeepAlive) and `agent-memory doctor`.
2. Register each client with `agent-memory agent add <name> --client <preset> [--dry-run]`. The command shows a diff-style preview and then writes the config:

   | Preset | Config it writes |
   |---|---|
   | claude-code | `claude mcp add -s user …` |
   | codex | `~/.codex/config.toml [mcp_servers.agent-memory]` |
   | cursor | `~/.cursor/mcp.json` |
   | claude-desktop | `claude_desktop_config.json` |
   | vscode | user `mcp.json` |

   Go-live clients are Claude Code (which also covers the Buzz agents) and Codex.
3. Add reinforcement snippets that mirror the server instructions to `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, and `~/.buzz/AGENTS.md` (below the managed markers).
4. Set up nightly backups through launchd, keeping 14 days. Forgotten text survives in backups until it rotates out; see THREAT_MODEL.
5. Run the live smoke test:
   - PA (Buzz) remembers a fact, and it is stored as verified.
   - Codex recalls it.
   - Sachin revises it in Claude Code, which creates an attested v2.
   - Sachin forgets the current version. v1 comes back, and the next recall is fresh, not a stale cache hit.

### 6.4 MCP server instructions (canonical text)
The daemon sends this text to every client when it connects. It tells the agent *when* to call each tool. [AGENT_UX](../40-interfaces/AGENT_UX.md) defines *what to say* afterwards.
- Write only when the user explicitly asks you to remember or keep something in mind. Put the user's exact words in `request.quote`, and add `request.buzz` when the request came from Buzz.
- Before answering questions about the user's past decisions or preferences, or facts they gave you, call `recall`.
- When the user changes a saved conclusion, call `recall` first and then `revise`. Never create a duplicate.
- For "forget…", identify the target with `recall` or `history`, call `forget`, then tell the user what is current now.
- Cite the agent and date for every recalled fact, or say that nothing is saved. Never guess.

## 7. Decisions
See [adr/README.md](adr/README.md).
