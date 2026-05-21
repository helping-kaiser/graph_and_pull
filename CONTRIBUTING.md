# Contributing to CoGra

This guide is for human contributors. [CLAUDE.md](CLAUDE.md) is
the AI-facing equivalent. Rules that apply to both audiences are
kept in both files — the shared subset is mission, core
principles, hard design rules, and workflow basics. Rules that
apply to only one audience (session hygiene, AI-specific
guardrails) live only in CLAUDE.md. Drift between the two is
caught by author vigilance, not tooling.

---

## What CoGra is

**CoGra** (Content Graph) is the **graph-architecture exploration**
for **Peer Network**'s next evolution (Peer Network PSE GmbH) — a
social media platform that replaces AI-driven content algorithms
with a transparent, graph-driven, user-controlled system. The
current Peer Network platform works like Instagram. This repo
branches off from main Peer Network development to design and
prototype the graph network that will succeed it; it is a
multi-year effort, not a short throwaway exploration.

**Mission:** decentralize the power of social media. The goal is
not to become the next Instagram/X/TikTok with a graph bolted on —
it is to shift power from social-media companies to users, a
massive network where the weight and ranking are owned by users
themselves. Every design decision must resist re-centralization.

---

## Core Principles

These are non-negotiable. Every decision must be evaluated against
them:

1. **No AI content algorithms.** Feed ranking is driven entirely by
   the social graph and direct edge weights. Every user gets a
   personalized view based on their own connections and explicit
   preferences.
2. **All edges are directional.** Nothing can push onto you.
   Inbound edges from others never affect your feed. Only your
   outgoing edges shape what you see.
3. **[Append-only](docs/primitive/layers.md#append-only-vocabulary)
   on the graph.** Graph state (edges and node properties) is
   immutable — you cannot delete or overwrite past interactions
   or values. New layers are added on top. The principle extends
   to Postgres-side display content, which uses versioned rows
   rather than overwrites. Transparency and auditability over
   convenience. See
   [docs/primitive/layers.md](docs/primitive/layers.md) for the
   full rule.
4. **Fair economics.** Ad revenue distributes across the economic
   landscape of the graph. Bot clusters earn nothing because real
   users never point toward them. Pull marketing, not push
   marketing.
5. **User comes first.** No amount of money changes this. Users
   choose what they see, including ads. No one can force their way
   into another user's feed.
6. **Governance, not admin escape hatches.** Redactions and
   policy changes run through community votes on the graph. There is no admin override — even court orders
   prompt a moderator to file
   [the same Proposal](docs/instances/moderation.md#2-reports--proposals-on-the-graph)
   anyone else would, leaving an auditable trail rather than a
   silent edit.
7. **Community choices stay local.** What a community decides —
   including severing ties — affects only the severing
   community's own outbound paths. The severance does not
   propagate forward to the severed party's other neighbours;
   viewing users whose own paths pass through the severing community
   see their feeds reshaped, but each of them chooses whether
   to cascade the severance further.
8. **Transparency over black boxes.** The system is a visible,
   auditable graph. Follow the principles of BTC: transparency,
   immutability, fairness.
9. **Fully public graph; no account needed to read.** Everything
   in the graph is visible without signing in. Accounts gate
   participation, not visibility.
10. **Privacy is per-content, not per-topology.** Chats and
    messages can be end-to-end encrypted; the social fabric —
    who is connected to whom — is intentionally public. CoGra
    protects what travels along the graph, not the graph itself.
11. **Fully open source.** The entire codebase is open source — a
    factual commitment, not a spirit. Forking, self-hosting, and
    running disconnected graphs are architecturally supported.
12. **Freedom of the mind.** No rewards for outrage, no
    manipulation, no dark patterns.

---

## Hard Rules

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
- **Never skip tests.** Linting, unit tests, and integration tests
  are created alongside the code, not after.

### Always

- **Explain why.** This is a learning project as much as a building
  project. Explain the reasoning behind choices, not just the
  implementation.
- **Move slowly and correctly.** Quality over speed. No rushing, no
  shortcuts.
- **Document decisions in the repo.** Any rule, principle, or
  agreement reached during discussion belongs in this file,
  [CLAUDE.md](CLAUDE.md), or a design doc — not in private notes,
  assistant memory, or anyone's head.

---

## Workflow

### Branches

Branch names follow `user/type/topic`. Examples:

- `jakob/primitive/network-node`
- `jakob/docs/extract-graph-schema`
- `jakob/cleanup/rehoming-and-nits`

Common type segments seen in the repo: `primitive`, `instances`,
`implementation`, `docs`, `cleanup`, `process`. Use a sensible new
type segment when none of the existing ones fits.

### Commits

**Atomic.** One commit = one logical task. A commit can touch
multiple files if all changes serve one purpose. Never mix
unrelated changes.

**Short.** Subject + at most 2-3 short body lines. Imperative mood;
describe the *why* not just the *what*.

Example: `add ChatMember junction node to support role-based chat
membership` — not `update data model`.

Section-by-section change lists, option comparisons, and full
design rationale belong in the **PR description**, not the commit
body. Reviewers read PRs; `git log` stays readable.

### Pull requests

PR body scaffold:

- `## Summary` — 1-3 sentences framing what this PR does.
- `## Reasoning` — the *why* behind the major decisions.
  **2-4 sentences per point.** Tradeoffs and what was rejected,
  not a re-derivation of the doc itself. Reviewers click through
  to the doc for the full detail.
- `## Commits` — compact list, one line per commit. The commit
  subject usually carries enough; add at most a short clarifying
  clause.
- `## Scope discipline` (optional) — only when there's a real scope
  question to flag (e.g. "this PR intentionally doesn't tackle X,
  that's PR Y").

Skip:

- Test-plan checklists.
- Filler subsections.
- Length-for-length's-sake.
- Per-commit prose that duplicates the commit body.

### Tests

Run `make ci` before pushing. `cargo fmt`, `cargo clippy -D
warnings`, unit tests, integration tests, and the docs
link-check (`make docs-link-check`, requires `lychee` —
`cargo install lychee`) must all pass.

---

## Code Style

- `cargo fmt` enforced.
- `clippy -D warnings` enforced.
- No `unwrap()` in library code — use `thiserror` / `anyhow`.
- Cypher queries only in `graph-engine`, SQL only in
  `postgres-store`.
- No comments on obvious code. Comments explain *why*, not *what*.

---

## Where to find things

- **Project spirit + workflow** — this file.
- **AI-assistant-specific rules** — [CLAUDE.md](CLAUDE.md).
- **Design docs** — [docs/](docs/), organized by primitive
  ([docs/primitive/](docs/primitive/)), instances
  ([docs/instances/](docs/instances/)), and implementation
  ([docs/implementation/](docs/implementation/)). See
  [docs/README.md](docs/README.md) for the full index.
- **Open design questions** —
  [docs/open-questions.md](docs/open-questions.md).
- **Development commands** —
  [docs/implementation/development.md](docs/implementation/development.md).
