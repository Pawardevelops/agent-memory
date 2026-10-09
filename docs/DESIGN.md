# agent-memory: design and build spec (v2)

Shared, local, long-term memory for **any AI agent** on Sachin's Mac: Claude Code, Codex, Buzz agents, Cursor, Claude Desktop, VS Code and custom scripts.

Status: approved by Sachin (Buzz #osro platform, thread `16f5b1c8…`, 2026-10-09).
- v1 covered Buzz agents only.
- v2 opens it to any agent and adds the cache layer. Sachin accepted Architect's defaults for the open decisions.

## Requirements (from Sachin)
1. Every agent can read and write, and each memory records **which agent** wrote it. "Agent" means any AI client, not only Buzz.
2. Agents write **only** when Sachin explicitly says "keep in mind", "remember" or similar.
3. Content and kind are free-form.
4. Runs on the local Mac only.
5. Embeddings are generated locally, so no text leaves the Mac.
6. Time-versioned: a later change of mind supersedes the earlier conclusion. "Forget the current discussion, keep the old one" deletes the newest version, and the previous one becomes current again.
7. The interface for agents is MCP. Non-MCP callers get REST.
8. Use the fastest practical stack.
9. No UI.
10. Include a cache layer.

## Decisions
| # | Decision | Choice |
|---|---|---|
| D1 | Process model | One local daemon owns the DB, the embedder and the caches. Clients connect to it, and none of them open the DB directly |
| D2 | Identity | Agent registry with per-agent bearer tokens. Buzz key identity is kept as an optional verifier |
| D3 | Writes from non-Buzz clients | Saved immediately with provenance `attested` and the owner's quoted request. No approval queue |
| D4 | First clients at go-live | Claude Code, Codex and Buzz agents (which run on Claude Code via claude-agent-acp). Cursor, Claude Desktop and VS Code come next |
| D5 | Cache | In-process caches inside the daemon, with generation-based invalidation. No Redis or other external cache (see Cache layer) |
| D6 | Network exposure | Bind to 127.0.0.1 only. Cloud agents (ChatGPT web, claude.ai) are out of scope and would need a tunnel plus OAuth later |

## Architecture
```
 Claude Code ──┐ MCP stdio ──► `agent-memory stdio --token-file …` (thin shim, no DB access)
 Codex ────────┤                         │
 Buzz agents ──┤                         ▼
 Cursor ───────┤ MCP Streamable HTTP ─► agent-memory daemon  127.0.0.1:7077  (launchd, KeepAlive)
 Claude Desktop┤                         ├─ auth: bearer token → agent registry
 scripts/n8n ──┘ REST JSON /v1/* ──────► ├─ core: topics · versions · recall · forget
                                         ├─ provenance: buzz verifier | attested
                                         ├─ cache layer (see below)
                                         ├─ SQLite ~/.buzz/agent-memory/memory.db (WAL, FTS5, sqlite-vec)
                                         └─ embedder: Ollama 127.0.0.1:11434 (nomic-embed-text, keep_alive=-1)
```
- **Daemon:** Python 3.13 + `uv`. Starlette/uvicorn serves both the MCP Streamable HTTP endpoint (`/mcp`, official `mcp` SDK) and the REST API (`/v1`). One asyncio process, with a single writer connection plus a reader pool.
- **stdio shim:** for clients that only spawn stdio MCP servers. It forwards JSON-RPC to the daemon over HTTP and holds no state. It starts the daemon if it isn't running.
- **Single writer:** all writes go through one connection and one asyncio lock, so `database is locked` errors can't happen.
- **Files:** everything lives in `~/.buzz/agent-memory/` with mode `700`. The DB is `600`.

## Identity: agent registry
```sql
CREATE TABLE agents (
  id          TEXT PRIMARY KEY,   -- ulid
  name        TEXT NOT NULL UNIQUE,   -- "Claude Code", "PA (Buzz)", "Codex"
  client      TEXT NOT NULL,      -- claude-code | codex | buzz | cursor | claude-desktop | vscode | script
  token_hash  TEXT NOT NULL,      -- sha256 of token; plaintext shown once
  created_at  TEXT NOT NULL,
  revoked_at  TEXT
);
```
- `agent-memory agent add <name> --client <preset>` creates the agent, generates a token, stores it in `~/.buzz/agent-memory/tokens/<name>` (mode 600) and **writes the client's MCP config**. The presets are:
  - `claude-code`: `claude mcp add -s user`
  - `codex`: `~/.codex/config.toml [mcp_servers.agent-memory]`
  - `cursor`: `~/.cursor/mcp.json`
  - `claude-desktop`: `claude_desktop_config.json`
  - `vscode`: user `mcp.json`
- Other commands: `agent list`, `agent revoke <name>`, `agent rotate <name>`.
- **Buzz agents** share one Claude Code MCP config, so a static token can't tell PA from Coder. The shim instead reads `BUZZ_ACP_DISPLAY_NAME` and `BUZZ_PRIVATE_KEY` from its environment and sends `X-Agent-Sub` (the display name plus a pubkey proof: a Schnorr signature over a daemon-issued nonce). The daemon records the Buzz agent as the sub-identity under the `buzz` agent.
- **Known limit:** all clients run as the same macOS user, so one agent could read another's token. Agent attribution is reliable labelling, not a security boundary. Real isolation would need separate OS users. That's out of scope.

## Provenance (owner-only writes)
Every write (`remember`, `revise`, `forget`) must carry `request`:
```json
{ "quote": "<Sachin's exact words that asked for this>",
  "conversation_id": "<Buzz thread root | Claude/Codex session id | chat id>",
  "buzz": { "channel_id": "…", "event_id": "…" }   // optional
}
```
- **verified:** when `request.buzz` is present, the daemon fetches the event with `buzz messages thread --channel … --event …`. It checks that the id is the NIP-01 hash, that the Schnorr sig is valid and that `pubkey == OWNER_PUBKEY` (`2b172f082d3bb2afcdb541e8a25e0504de6b305f01db1200d9eb0dfbcfffe202`). The memory is stored as `verified`. Any failure rejects the write (`not_requested_by_owner`, `bad_signature`, `event_not_found`).
- **attested:** with no `request.buzz`, `quote` is required and must be non-empty. The memory is stored as `attested` with the quote. It is not verifiable, but it's labelled, audited and reversible.
- `recall` always returns `provenance`.

A secret filter runs before any write. It rejects API keys, tokens, passwords, OTPs, card numbers (Luhn) and Aadhaar numbers (Verhoeff) with `looks_like_secret`. It also scans `quote`.

## Data model
```sql
CREATE TABLE topics (
  id TEXT PRIMARY KEY, title TEXT NOT NULL, project TEXT,
  status TEXT NOT NULL DEFAULT 'active',             -- active | forgotten
  created_at TEXT NOT NULL, created_by_agent TEXT NOT NULL REFERENCES agents(id)
);

CREATE TABLE entries (
  id               TEXT PRIMARY KEY,                 -- ulid
  topic_id         TEXT NOT NULL REFERENCES topics(id),
  version          INTEGER NOT NULL,
  content          TEXT,                             -- NULL once forgotten
  kind             TEXT,                             -- free text
  tags             TEXT,                             -- JSON array
  status           TEXT NOT NULL,                    -- current | superseded | forgotten
  agent_id         TEXT NOT NULL REFERENCES agents(id),
  agent_sub        TEXT,                             -- e.g. Buzz display name + pubkey
  provenance       TEXT NOT NULL,                    -- verified | attested
  request_quote    TEXT,                             -- NULL once forgotten
  conversation_id  TEXT,
  buzz_channel_id  TEXT, buzz_event_id TEXT,
  created_at       TEXT NOT NULL,
  UNIQUE (topic_id, version)
);
CREATE UNIQUE INDEX one_current_per_topic ON entries(topic_id) WHERE status = 'current';
CREATE INDEX entries_conv ON entries(conversation_id);

CREATE VIRTUAL TABLE entries_fts USING fts5(content, tags, title, entry_id UNINDEXED);
CREATE VIRTUAL TABLE entries_vec USING vec0(entry_id TEXT PRIMARY KEY, embedding float[768]);

CREATE TABLE embedding_cache (            -- persistent L2, see Cache layer
  key TEXT PRIMARY KEY,                   -- sha256(model || 0x00 || text); no plaintext stored
  model TEXT NOT NULL, embedding BLOB NOT NULL,
  kind TEXT NOT NULL,                     -- content | query
  last_used_at TEXT NOT NULL
);

CREATE TABLE audit_log (                  -- append-only, never stores content or quotes
  id INTEGER PRIMARY KEY, ts TEXT NOT NULL,
  action TEXT NOT NULL,                   -- remember | revise | forget | agent_add | agent_revoke
  topic_id TEXT, entry_id TEXT, agent_id TEXT, agent_sub TEXT,
  provenance TEXT, buzz_event_id TEXT
);

CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT);   -- schema_version, write_generation
```

## Cache layer
At this scale (one user, up to tens of thousands of facts), SQLite reads take well under 1 ms. The latency is in **embedding** (Ollama 20–60 ms per query, plus 1–3 s if the model is cold) and in **repeated identical recalls**. The cache targets exactly those costs. **No Redis:** it would be a second service to run and a second copy of personal data that forget would have to purge, with no speed gain on one machine.

| Layer | What | Where | Invalidation |
|---|---|---|---|
| C0 Model warm | Keep the embed model loaded (`keep_alive: -1`), and warm it up at daemon start | Ollama | n/a |
| C1 Query-embedding LRU | `hash(model, normalized query)` → vector. 2,000 entries | daemon memory | Model change |
| C2 Embedding cache (persistent) | `hash(model, text)` → vector, for both content and queries, so restarts and `reembed` skip known text | `embedding_cache` table, pruned to 50k rows / 30 days unused | Model change; **content rows deleted on forget** |
| C3 Recall result cache | `(query, filters, as_of, include_history, limit)` → result list. 500 entries, TTL 10 min | daemon memory | **Any write bumps `write_generation`, which drops all entries.** The key includes the generation |
| C4 Memory brief | A precomputed compact summary: the most-recalled and most recent current topics per project (≤ 2 KB), for agents to load at session start | memory, recomputed lazily after a generation change | Generation change |
| C5 SQLite | `cache_size=-65536` (64 MB), `mmap_size=256MB`, `temp_store=MEMORY` | SQLite | automatic |

**Cache invariants** (each has a test):
1. **Forget is total.** After `forget` returns, the forgotten content can't come back from any layer: C2 content rows are deleted, C3 and C4 are invalidated by the generation bump, and the FTS and vec rows are deleted. Nothing stores query text in plaintext.
2. **No stale reads after a write.** A `recall` that starts after a `remember`, `revise` or `forget` returns observes it. The generation is bumped inside the write transaction's commit path, before the response.
3. **The cache is optional.** If caching is disabled (`AGENT_MEMORY_CACHE=off`), results are identical, only slower.

Track the metrics `cache_hits` and `cache_misses` per layer, plus recall p50 and p95. `agent-memory stats` shows them.

## Tools (MCP) and endpoints (REST)
Each MCP tool has a matching REST endpoint at `/v1/<tool>` (POST JSON, bearer auth).

| Tool | Args | Behaviour |
|---|---|---|
| `remember` | `content, title?, kind?, tags?, project?, request, force_new?` | Provenance check, then secret filter. Searches for similar current topics, and if any match (and `force_new` isn't set), returns `possible_duplicate` with candidates and writes nothing. Otherwise creates the topic and entry v1 as `current` |
| `revise` | `topic_id, content, kind?, tags?, request` | One transaction: the current entry becomes `superseded`, and the new entry (`version = max+1`) becomes `current` |
| `recall` | `query, project?, as_of?, include_history=false, limit=8` | Hybrid: FTS5 bm25 top-20 + vec KNN top-20, merged by RRF (k=60). The default returns `current` only. `as_of` returns the latest non-forgotten version per topic with `created_at <= as_of`. Each result carries `topic_id, entry_id, title, version, content, kind, tags, agent, agent_sub, provenance, created_at, conversation_id, buzz_event_id, status` |
| `forget` | exactly one of `entry_id`, `topic_id`, `conversation_id`, or `{since, agent?}`, plus `request` | For each target: set `content` and `request_quote` to NULL, delete the FTS, vec and C2 rows, set `status='forgotten'`, and bump the generation. If the target was current, promote the highest remaining superseded version, or mark the topic forgotten if none remain. Writes an audit row. Returns what was forgotten and what is current now |
| `history` | `topic_id` | All versions. Forgotten ones appear as tombstones |
| `list` | `project?, since?, agent?, limit=50` | Current topics, newest first |
| `brief` | `project?` | C4 memory brief. Also exposed as the MCP resource `memory://brief` |

Errors use the shape `{error: code, message}`. Recall degrades to keyword-only with `warnings: ["embedder_unavailable"]`. Remember still succeeds and queues the embedding for backfill.

**MCP server `instructions`** are sent to every client on connect. They're the portable usage contract:
- Write only when the user explicitly asks to remember or keep something in mind. Include the user's exact words as `request.quote`, and include `request.buzz` when the request came from Buzz.
- Before answering questions about the user's past decisions, preferences or facts they gave you, call `recall`.
- When the user changes a saved conclusion, `recall` first and then `revise`. Never create a duplicate.
- "Forget…": identify the target with `recall` or `history`, call `forget`, then tell the user what is current now.
- Cite the agent and date for every recalled fact, or say nothing is saved. Never guess.

## CLI
- `agent-memory daemon` (run in the foreground) and `install-launchd`
- `stdio --token-file <f>`
- `agent add|list|revoke|rotate`
- `recall "<q>"`, `list`, `history <topic>`, `brief`
- `export --json`, `backup <dir>`, `reembed`, `stats`
- `doctor`: checks the daemon, Ollama and the model, sqlite-vec loading, file modes, owner config and client registrations

## Acceptance tests (pytest; mock the relay and Ollama)
1. Remember with a valid owner Buzz event is stored as `verified`. Remember with only a quote is stored as `attested`. Agent attribution comes from the token, not from args.
2. Buzz event by a non-owner → `not_requested_by_owner`. A tampered event → `bad_signature`. Nothing is written.
3. An attested write without a quote is rejected.
4. Revise → v2 current, v1 superseded. Recall returns v2 only, and `include_history` returns both.
5. `as_of` between v1 and v2 returns v1.
6. Forget v2 → v1 is current again. The v2 content is absent from `entries`, FTS, vec and `embedding_cache` (assert by direct SQL).
7. Forget by `conversation_id` touches only that conversation. Forget by `{since, agent}` touches only that agent's entries since then.
8. Forget by `topic_id` → the topic is forgotten, and recall returns nothing for it.
9. The secret filter blocks an API key, a card number and an Aadhaar number, in both content and quote.
10. **Cache:** recall twice → the second is a C3 hit. Then remember → the next recall misses and includes the new fact. Then forget → the next recall excludes it, with the cache on and with it off.
11. **Cache:** the same query embedding is computed once across recalls (C1), and survives a daemon restart (C2).
12. Concurrency: 20 parallel clients mixing writes and recalls → no errors, the one-current invariant holds, and no stale reads after a write ack.
13. A revoked token → 401. A token never appears in logs.
14. Ollama down → recall works keyword-only with a warning, and remember succeeds with the embedding backfilled later.
15. The stdio shim and the HTTP and REST paths return identical results for the same calls.

## Layout
```
src/agent_memory/
  daemon.py      # starlette app: /mcp (MCP Streamable HTTP) + /v1 (REST)
  shim.py        # stdio → HTTP forwarder
  core/          # store.py, versions.py, search.py, forget.py
  cache.py       # C1–C4, generation counter
  provenance/    # buzz.py (event fetch + NIP-01 verify), attested.py
  identity.py    # tokens, registry, buzz sub-identity
  embed.py       # ollama client + C2
  secrets_filter.py
  presets/       # claude_code.py, codex.py, cursor.py, claude_desktop.py, vscode.py
  cli.py
migrations/0001_init.sql
tests/
```
The dependencies are `mcp`, `starlette`, `uvicorn`, `httpx`, `sqlite-vec`, `coincurve`, `python-ulid` and `pytest`. The risk to check first is that Homebrew Python's `sqlite3` allows `enable_load_extension`. If it doesn't, use a `uv python` build or `pysqlite3`.

## Rollout
1. `brew install ollama uv && ollama pull nomic-embed-text`, then `agent-memory install-launchd` and `agent-memory doctor`.
2. Register the D4 clients: `agent add "Claude Code" --client claude-code` (which also covers the Buzz agents, with sub-identity) and `agent add "Codex" --client codex`.
3. Add reinforcement snippets that mirror the server instructions: the user-level `~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, and a "Shared memory" section below the managed markers in `~/.buzz/AGENTS.md`.
4. Nightly backups via launchd, keeping 14. Note: forgotten content survives in older backups until they rotate out.
5. Live smoke test:
   - PA (Buzz) remembers → verified.
   - Codex recalls it.
   - Sachin revises in Claude Code → attested v2.
   - Sachin says "forget the current, keep the old" → v1 is back, with an immediate recall miss and no stale cache.
6. Next clients: Cursor, Claude Desktop, VS Code.

## Not in scope
UI, multi-user, sync across devices, cloud-agent access, automatic capture without an explicit request, and an external cache.
