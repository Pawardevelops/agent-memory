# agent-memory: design and build spec

Shared, local, long-term memory for the Buzz agents on Sachin's Mac. It's exposed as an MCP server.

Status: approved by Sachin (Buzz #osro platform, thread `16f5b1c8…`, 2026-10-09).

## Requirements (from Sachin)
1. Every agent can read and write. Each memory records **which agent** wrote it.
2. Agents write **only** when Sachin explicitly says "keep in mind", "remember" or similar.
3. Content and kind are free-form.
4. Runs on the local Mac only.
5. Embeddings are generated locally. No text leaves the Mac.
6. Time-versioned: a later change of mind supersedes the earlier conclusion. "Forget the current discussion, keep the old one" deletes the newest version and the previous one becomes current again.
7. The interface is MCP.
8. Use the fastest practical stack.
9. No UI.

## Architecture
```
Buzz agent session (Claude Code via claude-agent-acp, or Codex)
   │ MCP stdio (one server process per agent session)
   ▼
agent-memory server (Python 3.13, official `mcp` SDK)
   ├─ identity:   pubkey derived from env BUZZ_PRIVATE_KEY (64-char hex secp256k1 → x-only pubkey)
   │              display name from env BUZZ_ACP_DISPLAY_NAME
   ├─ write gate: fetch the source event from the relay, verify it, check author == owner pubkey
   ├─ store:      SQLite ~/.buzz/agent-memory/memory.db  (WAL, busy_timeout=5000, chmod 600)
   │              FTS5 keyword index + sqlite-vec vector table
   └─ embedder:   Ollama http://127.0.0.1:11434, model `nomic-embed-text` (configurable, e.g. bge-m3)
```
- **One process per agent:** identity comes from the agent's own environment. The model never asserts its own identity, and no tool takes an "author" argument.
- **No network listener.** The only outbound calls are to Ollama on localhost and to the Buzz relay (through the `buzz` CLI) for the write gate.
- **Config** (env vars or `~/.buzz/agent-memory/config.toml`): `OWNER_PUBKEY` (Sachin: `2b172f082d3bb2afcdb541e8a25e0504de6b305f01db1200d9eb0dfbcfffe202`), `DB_PATH`, `EMBED_MODEL`, `OLLAMA_URL`.

## Data model
```sql
CREATE TABLE topics (
  id                TEXT PRIMARY KEY,      -- ulid
  title             TEXT NOT NULL,
  project           TEXT,
  status            TEXT NOT NULL DEFAULT 'active',   -- active | forgotten
  created_at        TEXT NOT NULL,         -- ISO-8601 UTC
  created_by_pubkey TEXT NOT NULL,
  created_by_name   TEXT
);

CREATE TABLE entries (
  id                TEXT PRIMARY KEY,      -- ulid
  topic_id          TEXT NOT NULL REFERENCES topics(id),
  version           INTEGER NOT NULL,
  content           TEXT,                  -- NULL once forgotten
  kind              TEXT,                  -- free text
  tags              TEXT,                  -- JSON array
  status            TEXT NOT NULL,         -- current | superseded | forgotten
  author_pubkey     TEXT NOT NULL,         -- agent that wrote it
  author_name       TEXT,
  requested_by      TEXT NOT NULL,         -- owner pubkey (verified)
  source_event_id   TEXT NOT NULL,
  source_thread_id  TEXT,
  source_channel_id TEXT NOT NULL,
  created_at        TEXT NOT NULL,
  UNIQUE (topic_id, version)
);
-- invariant: at most one 'current' entry per topic
CREATE UNIQUE INDEX one_current_per_topic ON entries(topic_id) WHERE status = 'current';

CREATE VIRTUAL TABLE entries_fts USING fts5(content, tags, title, entry_id UNINDEXED);
CREATE VIRTUAL TABLE entries_vec USING vec0(entry_id TEXT PRIMARY KEY, embedding float[768]);

CREATE TABLE audit_log (            -- append-only, never stores content
  id INTEGER PRIMARY KEY, ts TEXT NOT NULL, action TEXT NOT NULL,  -- remember|revise|forget
  topic_id TEXT, entry_id TEXT, actor_pubkey TEXT NOT NULL, actor_name TEXT,
  source_event_id TEXT
);

CREATE TABLE schema_version (version INTEGER NOT NULL);
```
Size the vector dimension from the configured model. Re-embedding on a model change is a CLI command (`agent-memory reembed`).

## Write gate (enforced in the server, not in prompts)
Every write tool (`remember`, `revise`, `forget`) requires `source_channel_id` and `source_event_id`. The server:
1. Runs `buzz messages thread --channel <id> --event <id>` (using the agent's own env) and finds the event with `id == source_event_id`.
2. Verifies the event: the id is the NIP-01 sha256 of its serialized form, and the Schnorr `sig` is valid for its `pubkey`.
3. Requires `pubkey == OWNER_PUBKEY`. Otherwise it rejects with `not_requested_by_owner`.
4. Records `source_thread_id` (the root from the event's `e` reply tag, or the event itself).

The gate proves that Sachin authored the triggering message. Deciding *what* he asked to keep is still the agent's job, and that's why every write is audited and reversible.

Before any write, a secret filter rejects content that matches API key, token, password, OTP, card number (Luhn) or Aadhaar (12-digit Verhoeff) patterns. The rejection code is `looks_like_secret`.

## MCP tools
| Tool | Args | Behaviour |
|---|---|---|
| `remember` | `content, title?, kind?, tags?, project?, source_channel_id, source_event_id` | Gate. Search for similar current topics (hybrid score ≥ threshold). If found and `force_new` isn't set, return `possible_duplicate` with candidates and write nothing, so the agent asks Sachin. Otherwise create topic + entry v1 (`current`) |
| `revise` | `topic_id, content, kind?, tags?, source_*` | Gate. In one transaction: the current entry becomes `superseded`, and a new entry `version = max+1` becomes `current` |
| `recall` | `query, project?, as_of?, include_history=false, limit=8` | Hybrid search: FTS5 bm25 top-20 + vec KNN top-20, merged with Reciprocal Rank Fusion (k=60). Default: `current` entries only. `as_of`: the latest non-forgotten version per topic with `created_at <= as_of`. `include_history`: all non-forgotten versions in order. Each result: `topic_id, entry_id, title, version, content, kind, tags, author_name, author_pubkey, created_at, source_channel_id, source_event_id, status` |
| `forget` | exactly one of `entry_id`, `topic_id`, `thread_id`, plus `source_*` | Gate. For each target entry: `content=NULL`, delete its FTS and vec rows, `status='forgotten'`. If it was `current`, promote the highest remaining `superseded` version to `current`. If none remain, the topic becomes `forgotten`. Audit row with no content. Returns what was forgotten and what is now current |
| `history` | `topic_id` | All versions, with forgotten ones shown as tombstones (no content) |
| `list` | `project?, since?, author?, limit=50` | Current topics, newest first |

Errors are structured: `{error: code, message}`. The codes are `not_requested_by_owner`, `event_not_found`, `bad_signature`, `looks_like_secret`, `not_found` and `embedder_unavailable`. Recall degrades to keyword-only if Ollama is down, and says so in a `warnings` field.

## CLI (same core, for Sachin and ops)
`agent-memory serve` (MCP stdio) · `init` · `recall "<q>"` · `list` · `history <topic>` · `export --json` · `backup <dir>` · `reembed` · `doctor` (checks Ollama, sqlite-vec, DB permissions and the owner config).

## Acceptance tests (must pass, pytest)
1. Remember with an owner-authored event creates topic v1. The author fields come from the env-derived identity.
2. Remember with a non-owner event → `not_requested_by_owner`, and nothing is written.
3. Tampered content or sig on the fetched event → `bad_signature`.
4. Revise → v2 current and v1 superseded. Recall returns v2 only. `include_history` returns both.
5. `as_of` between v1 and v2 returns v1.
6. Forget v2 → v1 becomes current again. The v2 content is gone from `entries`, FTS and vec (assert by direct SQL).
7. Forget by `thread_id` removes only entries from that thread, and other topics are untouched.
8. Forget by `topic_id` → the topic is forgotten, and recall returns nothing for it.
9. The secret filter blocks a sample API key, a card number and an Aadhaar number.
10. Concurrency: 10 processes writing in parallel → no `database is locked` errors leak out, and the one-current invariant holds.
11. Ollama down → recall still works (keyword-only) with a warning, and remember still succeeds (embedding queued and backfilled by `reembed`).

Mock the relay fetch and Ollama in unit tests. Run one live smoke test manually (see Rollout).

## Stack and layout
Python 3.13, `uv`, `mcp` (official SDK, FastMCP), `sqlite-vec`, `coincurve` (or `secp256k1`) for key derivation and Schnorr verify, `httpx`, `python-ulid`, `pytest`.
```
src/agent_memory/{server.py, cli.py, store.py, search.py, gate.py, identity.py, secrets_filter.py, embed.py, config.py}
migrations/0001_init.sql
tests/
```
Risk to check first: Homebrew Python's `sqlite3` must allow `enable_load_extension` for sqlite-vec. If it doesn't, use `uv python` builds or the `pysqlite3` wheel.

## Rollout
1. `brew install ollama uv && ollama pull nomic-embed-text`. Then `agent-memory init` and `agent-memory doctor`.
2. Register for Claude agents: user-scope `mcpServers.agent-memory` in `~/.claude.json` (`claude mcp add -s user agent-memory -- uv run --directory <repo> agent-memory serve`). For Codex: `[mcp_servers.agent-memory]` in `~/.codex/config.toml`.
3. Add a "Shared memory" section *below the managed markers* of `~/.buzz/AGENTS.md`:
   - Write only when Sachin explicitly asks ("keep in mind", "remember", "note this"). Pass the triggering event and channel.
   - When he changes a saved conclusion, `recall` first, then `revise` the topic. Don't create a duplicate.
   - "Forget…" → `recall` and identify the target, then `forget` it and report what is now current.
   - Before answering questions about his past decisions or preferences, `recall`. Cite the author and date, or say nothing is saved.
4. Nightly backup via launchd (`agent-memory backup ~/.buzz/agent-memory/backups`, keep 14). Note: forgotten content survives in older backups until they rotate out.
5. Live smoke test: PA remembers a fact, Coder recalls it, Sachin changes his mind (revise), Sachin forgets the current version → the old version is back.

## Not in scope
UI, multi-user, sync across devices, and automatic capture without an explicit request.
