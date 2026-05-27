# CLAUDE.md

This file is loaded into every Claude Code conversation on this
repo. **The rules below are operative, not background reading.**
Re-read this file at the start of every task.

**Audience split.** CLAUDE.md is AI-facing;
[CONTRIBUTING.md](CONTRIBUTING.md) is human-facing. Shared rules
(mission, core principles, hard design rules, workflow basics)
live in both; audience-specific rules (session hygiene, the
Commit + Push + PR cycle, autonomous-decision guardrails) live in
just one. Drift is caught by author vigilance, not tooling.

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
4. **Verify claims against the docs, not recall.** Open the
   relevant section before claiming how the system works — but
   don't re-read what's already in conversation context.
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
**PostgreSQL** (display content + operational metadata). See
[docs/implementation/architecture.md](docs/implementation/architecture.md).

Crates:

| Crate | Role |
|---|---|
| `api` | Axum HTTP server, async-graphql schema |
| `graph-engine` | Cypher queries against Memgraph via bolt protocol |
| `postgres-store` | SQLx queries, migrations, display-content CRUD |
| `common` | Shared domain types, error types |

Docs are layered:

- **`docs/primitive/`** — what the graph IS and how it BEHAVES.
- **`docs/instances/`** — concrete applications of the primitive.
- **`docs/implementation/`** — system and code-level concerns.

See [docs/README.md](docs/README.md) for the full index.
Cross-cutting design questions live in
[docs/open-questions.md](docs/open-questions.md).

---

## Hard rules — design

### Never

- **Never introduce AI into ranking, recommendations, or
  economics.** Feed ranking and ad-revenue distribution are
  driven only by the graph and its weights. AI as a
  frontend/UI helper is open — that boundary is intentionally
  permissive — but it must not touch the graph's signal or the
  economics computation.
- **Never delete graph structure.** Nodes, edges, and layer stacks
  are never removed. State transitions are always layered, never
  destructive. The only permitted "deletion" on the graph is
  in-place redaction per
  [docs/primitive/layers.md §5](docs/primitive/layers.md#5-deletion-policy). The
  same spirit applies to Postgres-side display content.
- **Never erase silently.** Any redaction — graph-side or
  Postgres-side — must leave a visible mark.
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
- **Never deviate silently.** If you have reason to break a rule
  in this file, name the rule and the reason — let the human
  accept or reject. Silent deviations look identical to violations
  from the outside; announced ones can be evaluated.
- **Never skip tests.** Linting, unit tests, and integration tests
  are created alongside the code, not after.

### Always

- **Explain why.** This is a learning project as much as a
  building project. Explain the reasoning behind choices, not just
  the implementation.
- **Move slowly and correctly.** Quality over speed. No
  rushing, no shortcuts.
- **Document decisions in the repo.** Any rule, principle, or
  agreement reached during discussion belongs in this file,
  [CONTRIBUTING.md](CONTRIBUTING.md), or a design doc — not in
  private notes, assistant memory, or anyone's head.

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

### Commit + Push + PR

When writing is done and no questions remain open in the
conversation — every unresolved item either decided or parked in
[docs/open-questions.md](docs/open-questions.md) — run the full
**commit + push + PR cycle in one motion, uninterrupted**. Don't
ask "want me to commit?", "should I push?", or propose a draft
commit message and wait for sign-off. The file edits were
reviewed one-by-one as they were proposed; that is the only
sign-off the workflow needs.

Task-completion framing — "resolve", "ship", "finalize", "let's
do X then resolve", or any phrasing that says the writing phase
is over — authorizes the whole cycle, commit step included. The
only legitimate stop is a genuine surprise in the diff (sensitive
files, an accidental edit), not a routine re-confirmation.

Stop and ask only **before** writing — to align on approach, pick
between options, or surface contradictions. Once the writing is
done, the workflow runs straight through to the PR.

---

## Hard rules — research and session hygiene

### Verify claims against the docs, not recall

The docs are the source of truth and grow long; recall is worse
than the file. Before making a claim about how the system works,
open the relevant section. The exception is files already in
conversation context — if a doc is loaded and hasn't been
edited, don't re-read it. Open what you need, skip what you
have. When making math-shaped claims (about ranking, weights,
dimensions), trace them back to the math in the docs — if you
can't, the claim is suspect.

### Flag contradictions inline

If a doc contradicts another, conflicts with the user's framing,
or seems out of place — flag it in the same response. Don't paper
over it; don't file it as a separate later task.

### Use a subagent for broad investigation

For investigations spanning more than a few files, spawn a
subagent. It does the heavy reading inside its own context,
returns a summary, and keeps the main thread lean — the cheapest
way to investigate without bloating the session.

### One session per task

Each PR merge is a natural session boundary. After a task closes,
**suggest a fresh session before starting the next task.** Long
sessions accumulate context that doesn't help: redundant doc
re-reads, resolved discussions, stale hypotheses. Fresh sessions
reload this file and start lean.

### Tightening passes: write current state, not change history

When fixing wrong, stale, or imprecise text in a docs pass:

- **Prefer deletion to rewriting.** If a sentence's only job was
  a comparison or restatement that turns out to be wrong, delete
  it. Don't replace a wrong sentence with a longer correct one
  whose only purpose is to explain the cut.
- **Never leave markers of what was removed.** No "previously X,
  now Y", no "the rule used to be Z", no "no longer stored" — the
  doc describes the current state; the change history lives in
  git.
- **Overly verbose is bad.** A reader wants the current rule in
  the fewest words that carry it. Trim, don't pad.

---

## Development commands

```bash
make run    # first-time: init + start DBs + migrate + start API
make dev    # returning: start DBs + migrate + start API
make api    # just the API (if DBs already running)
make ci     # lint + test + docs link check (run before pushing)
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
