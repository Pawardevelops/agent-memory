---
title: "Architecture decision records"
owner: Architect
status: active
applies_to: all-phases
last_reviewed: 2026-10-10
---

# Architecture decision records

ADRs are append-only:
- To change a decision, add a new ADR and set the old one's `status: superseded` and `superseded_by`.
- Use [TEMPLATE.md](TEMPLATE.md).
- Number ADRs sequentially.

| ADR | Decision | Status |
|---|---|---|
| [0001](0001-local-daemon-sqlite.md) | One local daemon, SQLite as source of truth | accepted |
| [0002](0002-any-agent-token-identity.md) | Any-agent access with token registry | accepted |
| [0003](0003-provenance-verified-attested.md) | Owner-only writes via verified/attested provenance | accepted |
| [0004](0004-topics-versions-hard-forget.md) | Topics with versions; forget = hard delete that reverts | accepted |
| [0005](0005-in-process-cache.md) | In-process cache with generation invalidation; no Redis | accepted |
| [0006](0006-single-service-api-adapters.md) | One service API; MCP and REST are adapters | accepted |
| [0007](0007-phased-delivery-fts-first.md) | Phased delivery, FTS first | accepted |
| [0008](0008-docs-per-phase-snapshots.md) | Docs: living target, phase snapshots, append-only ADRs | accepted |
