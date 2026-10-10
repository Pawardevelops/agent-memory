---
title: "DESIGN.md (pointer)"
owner: Architect
status: superseded
applies_to: all-phases
last_reviewed: 2026-10-10
---

# agent-memory design: where it lives now

The single-file spec (v1–v3) has been split into the documents below. The full v3 text is still in git history at commit `434aa4b` (`git show 434aa4b:docs/DESIGN.md`).

| You're looking for | Go to |
|---|---|
| Requirements, users, success | [00-vision/END_GOAL.md](00-vision/END_GOAL.md) |
| Target architecture, flows, phase map, cache layer, Buzz sub-identity, rollout | [10-architecture/OVERVIEW.md](10-architecture/OVERVIEW.md) |
| What Phase 1 builds | [10-architecture/phase-1/ARCHITECTURE.md](10-architecture/phase-1/ARCHITECTURE.md), [OUTPUT.md](10-architecture/phase-1/OUTPUT.md) |
| Schema and invariants | [10-architecture/DATA_MODEL.md](10-architecture/DATA_MODEL.md) |
| Decisions D1–D6 | [10-architecture/adr/](10-architecture/adr/README.md) (D7, the inference contract, is in RETRIEVAL.md) |
| Retrieval, confidence, budget, evals | [20-inference/RETRIEVAL.md](20-inference/RETRIEVAL.md) |
| Trust model, tokens, secrets, forget/backup | [30-security/THREAT_MODEL.md](30-security/THREAT_MODEL.md) |
| Tool contracts, errors, CLI | [40-interfaces/AGENT_UX.md](40-interfaces/AGENT_UX.md) |
| MCP server instructions (canonical text) | [10-architecture/OVERVIEW.md §6.4](10-architecture/OVERVIEW.md) |
| Stack and skills | [50-tech/STACK_AND_SKILLS.md](50-tech/STACK_AND_SKILLS.md) |
| Phases, PR flow, review gates | [ROADMAP.md](ROADMAP.md), [60-process/WORKFLOW.md](60-process/WORKFLOW.md) |
