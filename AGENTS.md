# agent-memory

Local memory daemon (MCP + REST) shared by any AI agent on Sachin's Mac. Docs start at `docs/README.md`. Read `docs/10-architecture/phase-N/ARCHITECTURE.md` for the phase you're building, and the ADRs, before changing anything.

## Rules
- Enforce invariants in the server and the schema, not in prompts: the write gate, one current version per topic, hard delete on forget, and no forgotten content in any cache layer.
- Never take the agent identity from a tool argument. It comes from the bearer token (and the Buzz sub-identity proof).
- Never store or log memory content in `audit_log`, errors or debug logs.
- Every behaviour change needs a pytest test. Run the full suite before pushing.
- Schema changes go in a new numbered file in `migrations/`. Never edit an applied migration.

## Git
- Remote: `git@github-work:Pawardevelops/agent-memory.git` (Sachin's work account, not onro or personal).
- Commit identity: `Pawardevelops <pawardevelops@gmail.com>`. No signing unless Sachin names a key.
  The Buzz agent runtime injects its own `user.name`/`user.email` on the command line, which overrides repo config. Commit with
  `GIT_AUTHOR_NAME=Pawardevelops GIT_AUTHOR_EMAIL=pawardevelops@gmail.com GIT_COMMITTER_NAME=Pawardevelops GIT_COMMITTER_EMAIL=pawardevelops@gmail.com git commit ...`
  and check with `git log --format='%an <%ae> / %cn <%ce>'` before pushing.
- Code changes go on feature branches with PRs. Don't push code to `main` directly.
