# CLAUDE.md

This file is loaded into every Claude Code conversation on this
repo. **The rules below are operative, not background reading.**
Re-read this file at the start of every task.

The project's mission, core principles, hard design rules, and
contribution workflow are mirrored in
[CONTRIBUTING.md](CONTRIBUTING.md) for human contributors. The
content overlaps deliberately: both audiences (Claude here,
humans there) need the same rules in their canonical doc, and
relying on Claude to navigate to a separate file before acting is
unreliable. **Updates to workflow rules must be made in both
files.**

---

## Critical reminders

If you only remember a handful of things from this file, remember
these — these are the rules most often violated:

1. **Never make design decisions autonomously.** Suggest options,
   explain trade-offs, let the human decide.
2. **Atomic commits.** One commit = one logical task. Never mix
   unrelated changes.
3. **Short commits, long PRs.** Commit body ≤ 2-3 lines. Full
   rationale goes in the PR description, never the commit body.
4. **Re-read the relevant docs before claiming.** The docs are the
   source of truth and grow long; recall is a worse source than
   the file itself.
5. **Flag contradictions inline.** If a doc contradicts another or
   the user's framing, raise it in the same response. Don't paper
   over it.
6. **One session per task.** After a PR is merged, suggest a fresh
   session before starting the next task. Long sessions
   accumulate context that doesn't help.
7. **Never deviate silently.** If you have reason to break a rule
   here, name the rule and the reason — let the human accept or
   reject. The rule is not "never deviate," it's "never deviate
   silently." Silent deviations look identical to violations from
   the outside; announced ones can be evaluated.

---

## Architecture (one-screen reference)

Dual-database: **Memgraph** (graph topology, edges, traversal) +
**PostgreSQL** (metadata, display content). See
[docs/implementation/architecture.md](docs/implementation/architecture.md).

Crates:

| Crate | Role |
|---|---|
| `api` | Axum HTTP server, async-graphql schema |
| `graph-engine` | Cypher queries against Memgraph via bolt protocol |
| `postgres-store` | SQLx queries, migrations, metadata CRUD |
| `common` | Shared domain types, error types |

Docs are layered:

- **`docs/primitive/`** — what the graph IS and how it BEHAVES
  (graph-model, nodes, edges, layers, retention-archive,
  governance, authorship, feed-ranking, invitations, network).
- **`docs/instances/`** — concrete applications of the primitive
  (chats, collectives, items, moderation, account-deletion).
- **`docs/implementation/`** — system and code-level concerns
  (architecture, data-model, graph-data-model, development,
  api-spec, graph-db-options).

See [docs/README.md](docs/README.md) for the full index.
Cross-cutting design questions live in
[docs/open-questions.md](docs/open-questions.md).

---

## Hard rules — design

### Never

- **Never introduce AI-based ranking or recommendations.** The
  graph and its weights are the only ranking mechanism.
- **Never delete graph structure.** Nodes, edges, and layer stacks
  are never removed. State transitions are always layered, never
  destructive. The only permitted "deletion" on the graph is
  in-place redaction per
  [docs/primitive/layers.md §5](docs/primitive/layers.md). The
  same spirit applies to Postgres-side display content.
- **Never erase silently.** Any redaction or takedown — graph-side
  or Postgres-side — must leave a visible mark.
- **Never let inbound edges affect a user's feed.** Only outgoing
  edges from the viewing user shape their feed.
- **Never break edge tensor uniformity.** All edges (actor and
  structural) have the same shape: 2 dimensions + system
  dimensions.
- **Never store graph topology in Postgres or content in
  Memgraph.** Each database does what it's built for.
- **Never make design decisions autonomously.** Always ask.
  Suggest options, explain trade-offs, but let the human decide.
  Design reasoning often exists that isn't visible in the code.
- **Never skip tests.** Linting, unit tests, and integration tests
  are created alongside the code, not after.

### Always

- **Explain why.** This is a learning project as much as a
  building project. Explain the reasoning behind choices, not just
  the implementation.
- **Move slowly and correctly.** Quality over speed.
- **Document decisions in the repo.** Any rule, principle, or
  agreement reached during discussion belongs in this file,
  [CONTRIBUTING.md](CONTRIBUTING.md), or a design doc — not in
  memory or anyone's head.

---

## Hard rules — workflow

### Branches

`user/type/topic`. Examples: `jakob/primitive/network-node`,
`jakob/docs/extract-graph-schema`. Common types: `primitive`,
`instances`, `implementation`, `docs`, `cleanup`, `process`. Use
a sensible new type segment when none of the existing ones fits.

### Commits

**Atomic** — one commit = one logical task; never mix unrelated
changes. **Short** — subject + at most 2-3 body lines, imperative
mood, describe the *why* not just the *what*. Section-by-section
change lists, option comparisons, and full design rationale
belong in the PR description, not the commit body.

### PR body scaffold

- `## Summary` — 1-3 sentences.
- `## Reasoning` — the *why* behind major decisions.
  **2-4 sentences per point.** Tradeoffs and what was rejected,
  not a re-derivation of the doc.
- `## Commits` — compact list, one line per commit.
- `## Scope discipline` (optional) — only when there's a real
  scope question to flag.

No test-plan checklist. No filler subsections. No per-commit prose
that duplicates the commit body.

### Push + PR

After the last planned commit, **push and open the PR directly**.
Don't ask "want me to push?". File edits were reviewed one-by-one
as they were proposed; the user doesn't need a re-confirm step.

---

## Hard rules — research and session hygiene

### Re-read docs before claiming

The docs are the source of truth and grow long. Before making a
claim about how the system works, open the relevant section and
re-read it. Recall is a worse source than the file. When making
math-shaped claims (about ranking, weights, dimensions), trace
them back to the math in the docs — if you can't, the claim is
suspect.

### Flag contradictions inline

If a doc contradicts another, conflicts with the user's framing,
or seems out of place — flag it in the same response. Don't paper
over it; don't file it as a separate later task.

### Don't re-read what's already in context

If a file is already in conversation context and hasn't been
edited, don't re-read it. Save the tokens.

### Use the Explore subagent for multi-file research

For broad investigations spanning more than a few files, spawn an
`Explore` subagent. It does the heavy reading inside its own
context, returns a summary, and keeps the main thread lean — this
is the cheapest way to investigate without bloating the session.

### One session per task

Each PR merge is a natural session boundary. After a task closes,
**suggest a fresh session before starting the next task.** Long
sessions accumulate context that doesn't help: redundant doc
re-reads, resolved discussions, stale hypotheses. Fresh sessions
reload this file and start lean.

---

## Development commands

```bash
make run    # first-time: init + start DBs + migrate + start API
make dev    # returning: start DBs + migrate + start API
make api    # just the API (if DBs already running)
make ci     # lint + test (run before pushing)
```

Full make-target list, env vars, and other dev guidance:
[docs/implementation/development.md](docs/implementation/development.md).

### Code style

- `cargo fmt` enforced.
- `clippy -D warnings` enforced.
- No `unwrap()` in library code — use `thiserror` / `anyhow`.
- Cypher queries only in `graph-engine`, SQL only in
  `postgres-store`.
- No comments on obvious code. Comments explain *why*, not *what*.
