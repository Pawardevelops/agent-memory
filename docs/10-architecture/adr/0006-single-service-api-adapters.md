---
title: "ADR 0006: One service API; MCP and REST are adapters"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0006: One service API; MCP and REST are adapters

## Context
The memory must work for MCP clients (via stdio or HTTP) and non-MCP callers (scripts, n8n), and the behaviour must be identical everywhere.

## Decision
The application services (remember, revise, recall, forget, history, list, brief) are the only API. REST `/v1` maps to them directly. The stdio shim is an MCP server that forwards to REST (Phase 1). A native MCP Streamable HTTP endpoint `/mcp` is added in Phase 4. Adapters hold no rules.

## Alternatives considered
- **MCP-only:** excludes scripts.
- **Separate MCP and REST implementations:** they would drift apart.

## Consequences
- One extra local HTTP hop for stdio clients, which costs sub-millisecond time.
- A parity test runs the same calls through the shim and REST (and later `/mcp`) and expects identical results.
