# CoGra

A spike for **Peer Network** — exploring how a graph-driven social media
platform can replace AI content algorithms with transparent, user-controlled
feed ranking based on the social graph.

No AI algorithms. No push marketing. No black boxes. Every user's feed is
computed from their own position in the graph and the weighted edges they
create through explicit interactions.

See [CLAUDE.md](CLAUDE.md) for the full set of project principles.

## Architecture

Two databases, each doing what it does best:

- **Memgraph** — the social graph: nodes (users, companies, posts, comments,
  chats, items, hashtags, junction nodes), directional tensor edges, and all
  traversal/ranking queries in Cypher.
- **PostgreSQL** — metadata: profiles, post content, media URLs, display data.
  Everything needed to render a page, nothing needed to weight the graph.

```
┌─────────────┐     GraphQL      ┌────────────────────────────────────────┐
│   Client    │ ──────────────── │             API  (Axum)                │
└─────────────┘                  └──────────────┬─────────────────────────┘
                                                │
                              ┌─────────────────┴──────────────────┐
                              │                                     │
                   ┌──────────▼──────────┐             ┌───────────▼───────────┐
                   │   graph-engine      │             │   postgres-store      │
                   │  (Cypher / bolt)    │             │      (SQLx)           │
                   └──────────┬──────────┘             └───────────┬───────────┘
                              │                                     │
                   ┌──────────▼──────────┐             ┌───────────▼───────────┐
                   │     Memgraph        │             │      PostgreSQL        │
                   │   (graph layer)     │             │   (metadata layer)    │
                   └─────────────────────┘             └───────────────────────┘
```

The shared key between both databases is the **UUID** assigned at creation
time. Memgraph stores graph topology (nodes + tensor edges). PostgreSQL stores
everything needed to display content.

## Crate Structure

| Crate | Role |
|---|---|
| `api` | Axum HTTP server, async-graphql schema, request handlers |
| `graph-engine` | Cypher queries against Memgraph via bolt protocol |
| `postgres-store` | SQLx queries, migrations, metadata CRUD |
| `common` | Shared domain types, error types, UUIDs |

## Quick Start

```bash
make run          # first-time: init + start DBs + migrate + start API
make dev          # returning: start DBs + migrate + start API
make api          # just the API (if DBs already running)
```

Memgraph Lab (visual graph browser): http://localhost:3000

## Make Commands

```
make init         first-time setup: copy .env, check/install dependencies
make run          full start: init + dev (first-time friendly)
make dev          start DBs + migrate + start API
make api          start the API server
make up           start all services (Postgres + Memgraph)
make down         stop all services
make reset-db     wipe all data and re-migrate
make migrate      run pending Postgres migrations
make ci           full CI pipeline (lint + test)
make lint         clippy + fmt check
make fmt          format all code
make test         cargo test --all
make logs         follow docker compose logs
```

## Documentation

### Design

- [Graph Model](docs/graph-model.md) — the core: node categories, edge categories, dimensions, directionality, append-only, junction approval pattern
- [Nodes](docs/nodes.md) — full node catalog: what each type is, its graph-side properties, and where display content lives
- [Edges](docs/edges.md) — full edge catalog and the relationship-label scheme at the graph layer
- [Layers](docs/layers.md) — append-only principle across edges, node properties, and Postgres-side display content
- [Feed Ranking](docs/feed-ranking.md) — ranking algorithm for ordering target nodes from a root node's perspective
- [Chats](docs/chats.md) — chats and messages as first-class public content; privacy via end-to-end encryption of content only
- [Authorship](docs/authorship.md) — how authorship is derived from the earliest incoming edge
- [Invitations](docs/invitations.md) — the two-edge onboarding pattern for new actors
- [Companies](docs/companies.md) — companies as actors; CompanyMember flow; economic role (not preferential)
- [Items](docs/items.md) — items as content nodes; ItemOwnership transfer flow
- [Architecture](docs/architecture.md) — system design, dual-database split, data flow
- [Data Model](docs/data-model.md) — PostgreSQL schema (display metadata)
- [Graph DB Decision Record](docs/graph-db-options.md) — why Memgraph, alternatives considered
- [Open Questions](docs/open-questions.md) — consolidated index of unresolved design calls

### API

- [API Spec](docs/api-spec.md) — **outdated, pending redesign** to align with tensor model

### Development

- [Development Guide](docs/development.md) — local setup, tools, workflows

## Tech Stack

| Concern | Choice |
|---|---|
| Language | Rust 2021 |
| API | Axum + async-graphql |
| Graph DB | Memgraph (openCypher, bolt protocol) |
| Metadata DB | PostgreSQL 16 (SQLx) |
| Local dev | Docker Compose |
| CI | GitHub Actions |
