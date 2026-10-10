---
title: "ADR 0002: Any-agent access with a token registry"
owner: Architect
status: accepted
date: 2026-10-09
supersedes: none
superseded_by: none
---

# ADR 0002: Any-agent access with a token registry

## Context
Scope widened (2026-10-09) from Buzz agents to any AI client: Claude Code, Codex, Cursor, Claude Desktop, VS Code and scripts. Buzz key-derived identity doesn't exist in those tools.

## Decision
Each client is registered with `agent-memory agent add`, which issues a bearer token that is stored hashed. Every write records the agent. Buzz agents share one Claude Code config, so from Phase 4 they add a signed sub-identity (display name plus pubkey proof).

## Alternatives considered
- **Self-reported agent name in tool arguments:** trivially wrong or spoofable by the model.
- **Separate macOS users per agent:** real isolation, but heavy setup for one person's tools.

## Consequences
- Attribution is **reliable labelling, not a security boundary**: all clients run as one macOS user and could read each other's token files. See THREAT_MODEL.
- Tokens can be revoked and rotated per agent.
