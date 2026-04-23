# CLAUDE.md

This file is loaded at the start of every Claude Code conversation. It is the
single source of truth for how to work on this project.

---

## What This Project Is

A spike for **Peer Network** (Peer Network PSE GmbH) — a social media platform
that replaces AI-driven content algorithms with a transparent, graph-driven,
user-controlled system. The current Peer Network platform works like Instagram.
This repo explores the architecture for its next evolution: the graph network.

Project name: **CoGra** (Content Graph).

**Mission:** decentralize the power of social media. The goal is not to
become the next Instagram/X/TikTok with a graph bolted on — it is to shift
power from social-media companies to users, a massive network where the
weight and ranking are owned by users themselves. Every design decision
must resist re-centralization.

### Core Principles

These are non-negotiable. Every decision must be evaluated against them:

1. **No AI content algorithms.** Feed ranking is driven entirely by the social
   graph and direct edge weights. Every user gets a personalized view based on
   their own connections and explicit preferences.
2. **All edges are directional.** Nothing can push onto you. Inbound edges
   from others never affect your feed. Only your outgoing edges shape what you
   see.
3. **Append-only edges.** Edge history is immutable. You cannot delete or
   overwrite past interactions. New layers are added on top. Transparency and
   auditability over convenience.
4. **Fair economics.** Ad revenue distributes across the economic landscape of
   the graph. Bot clusters earn nothing because real users never point toward
   them. Pull marketing, not push marketing.
5. **User comes first.** No amount of money changes this. Users choose what
   they see, including ads. No one can force their way into another user's
   feed.
6. **Transparency over black boxes.** The system is a visible, auditable
   graph. Follow the principles of BTC: transparency, immutability, fairness.
7. **Fully open source.** The entire codebase is open source — a factual
   commitment, not a spirit. Forking, self-hosting, and running
   disconnected graphs are architecturally supported.
8. **Freedom of the mind.** No rewards for outrage, no manipulation, no dark
   patterns.

---

## Architecture

Dual-database: **Memgraph** (graph topology, edges, traversal) +
**PostgreSQL** (metadata, display content). See [docs/architecture.md](docs/architecture.md).

Crate structure:

| Crate | Role |
|---|---|
| `api` | Axum HTTP server, async-graphql schema |
| `graph-engine` | Cypher queries against Memgraph via bolt protocol |
| `postgres-store` | SQLx queries, migrations, metadata CRUD |
| `common` | Shared domain types, error types |

### Key Design Documents

Read these before making changes to data models or algorithms:

- [Edge Tensor Model](docs/edge-tensor-model.md) — the edge/node system. All
  edges are 2-dimensional directional tensors with append-only layers.
- [Feed Ranking](docs/feed-ranking.md) — how target nodes are ranked from a
  root node's perspective.
- [Data Model](docs/data-model.md) — Postgres schema + graph definitions.

---

## Hard Rules

### Never do these:

- **Never introduce AI-based ranking or recommendations.** The graph and its
  weights are the only ranking mechanism.
- **Never allow edge deletion.** Edges are append-only. New layers on top,
  never remove or overwrite.
- **Never let inbound edges affect a user's feed.** Only outgoing edges from
  the viewing user shape their feed.
- **Never break edge tensor uniformity.** All edges (actor and structural)
  have the same shape: 2 dimensions + system dimensions.
- **Never store graph topology in Postgres or content in Memgraph.** Each
  database does what it's built for.
- **Never make design decisions autonomously.** Always ask. Suggest options,
  explain trade-offs, but let the human decide. Design reasoning often exists
  that isn't visible in the code.
- **Never skip tests.** Linting, unit tests, and integration tests are created
  alongside the code, not after.

### Always do these:

- **Explain why.** This is a learning project as much as a building project.
  Explain the reasoning behind choices, not just the implementation.
- **Move slowly and correctly.** Quality over speed. No rushing, no shortcuts.
- **Follow atomic commits.** One commit = one logical task. A commit can touch
  multiple files if all changes serve one purpose. Never mix unrelated changes.
- **Branch-per-task.** Create a branch from main, work in small commits, merge
  via PR. Keep branch lifetime short.
- **Test everything.** `cargo fmt`, `cargo clippy -D warnings`, unit tests,
  integration tests.
- **Document decisions in the repo.** Any rule, principle, or agreement
  reached during discussion belongs in this file or a design doc — not in
  private notes, assistant memory, or anyone's head. Other contributors,
  other devices, and future sessions need to see the same truth.

---

## Development

### Quick Start

```bash
make run    # first-time: init + start DBs + migrate + start API
make dev    # returning: start DBs + migrate + start API
make api    # just the API (if DBs already running)
```

### Common Commands

```bash
make init       # copy .env, check/install dependencies
make up         # start Postgres + Memgraph
make down       # stop services
make migrate    # run pending Postgres migrations
make reset-db   # wipe all data, re-migrate
make lint       # clippy + fmt check
make fmt        # format code
make test       # cargo test --all
make ci         # lint + test
make logs       # follow docker logs
```

### Environment

All config in `.env` (copied from `.env.example`). Key vars:
- `DATABASE_URL` — Postgres connection
- `MEMGRAPH_HOST` / `MEMGRAPH_PORT` — Memgraph bolt connection
- `API_HOST` / `API_PORT` — API bind address
- `RUST_LOG` — log level (trace/debug/info/warn/error)

### Code Style

- `cargo fmt` enforced
- `clippy -D warnings` enforced
- No `unwrap()` in library code — use `thiserror` / `anyhow`
- Cypher queries only in `graph-engine`, SQL only in `postgres-store`
- No comments on obvious code. Comments explain *why*, not *what*.

---

## Git Workflow

1. Create a branch from `main` for the task
2. Work in atomic commits (one logical change per commit)
3. Ensure `make ci` passes
4. Merge via PR back to `main`
5. Keep branches short-lived

Commit messages: concise, imperative mood, describe the *why* not just the
*what*. Example: "add ChatMember junction node to support role-based chat
membership" not "update data model".

**Short commits, long PRs.** Commit body is at most a few lines — subject
plus the minimum why. Option comparisons, section-by-section change lists,
and full design rationale belong in the **PR description**, not the
commit. Reviewers read PRs; `git log` stays readable.
