# agent-memory

Local MCP memory server shared by the Buzz agents on Sachin's Mac. The spec is in `docs/DESIGN.md`; read it before changing anything.

## Rules
- Enforce invariants in the server and the schema, not in prompts: the write gate, one current version per topic, hard delete on forget.
- Never take the author identity from a tool argument. Derive it from the process environment.
- Never store or log memory content in `audit_log`, errors or debug logs.
- Every behaviour change needs a pytest test. Run the full suite before pushing.
- Schema changes go in a new numbered file in `migrations/`. Never edit an applied migration.

## Git
- Remote: `git@github-work:Pawardevelops/agent-memory.git` (Sachin's work account, not onro or personal).
- Commit identity: `Pawardevelops <pawardevelops@gmail.com>`. No signing unless Sachin names a key.
  The Buzz agent runtime injects its own `user.name`/`user.email` on the command line, which overrides repo config. Commit with
  `GIT_AUTHOR_NAME=Pawardevelops GIT_AUTHOR_EMAIL=pawardevelops@gmail.com GIT_COMMITTER_NAME=Pawardevelops GIT_COMMITTER_EMAIL=pawardevelops@gmail.com git commit ...`
  and check with `git log --format='%an <%ae> / %cn <%ce>'` before pushing.
- Work on feature branches and open PRs. Don't push to `main` directly after the initial scaffold.
