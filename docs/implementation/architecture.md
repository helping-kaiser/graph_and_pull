# Architecture

## Overview

CoGra is the **graph-architecture exploration** for **Peer Network**'s
next evolution — a social media platform where feed ranking is driven
entirely by the social graph and explicit user interactions, not AI
algorithms.

The system uses a dual-database architecture. A social network has two
fundamentally different data access patterns that map poorly to a single
database:

1. **Traversal queries** — "what should I see next?", "how is this person
   connected to me?", "what's happening in my part of the graph?" These are
   graph problems. They perform well when the database understands
   relationships natively.

2. **Lookup queries** — "give me the profile for user X", "give me the
   content of post Y". These are key-value / relational lookups. They perform
   well in a traditional RDBMS.

Storing everything in one database forces a compromise. Storing graph topology
in a graph DB and metadata in Postgres lets each database do what it was built
for.

---

## Design Principles

### 1. Graph DB owns topology, Postgres owns content

If a piece of data is needed to **navigate or weight** the graph, it goes in
Memgraph. If it is needed to **display** something, it goes in Postgres.

| Data | Where | Why |
|---|---|---|
| Node ID (UUID) | Both | Shared key between databases |
| User bio, avatar, display name | Postgres | Display only |
| Post content, media URLs | Postgres | Display only |
| Actor edges (dim1, dim2 — see [edges.md](../primitive/edges.md) for per-edge-type labels) | Memgraph | Graph topology + ranking |
| Structural edges (containment, tagging) | Memgraph | Graph topology |
| Cached author_id on nodes | Both | Derived from earliest incoming edge, cached for fast lookup |

See [Graph Model](../primitive/graph-model.md) for the full node/edge
specification.

### 2. UUIDs as the shared key

Every entity gets a UUID at creation time. This UUID is stored in both
databases and is the only way they reference each other. The graph engine
never needs to know a username; the Postgres store never needs to know the
graph topology.

### 3. All ranking comes from the graph

There are no materialized counters, no popularity scores, no
algorithm-driven signals stored as node properties. Feed ranking
is computed at query time from the
[edge tensor model](../primitive/graph-model.md) using the
[feed ranking algorithm](../primitive/feed-ranking.md). The
algorithm itself — its parameters, sort order, and tie-breaker
chain — lives in `feed-ranking.md`; this doc covers only how the
system runs it (per-viewer, off the central hot path).

### 4. Edges are the source of truth

All graph state lives in edges. Edges are:
- **Directional** — A → B and B → A are independent
- **Multi-dimensional** — 2 user dimensions + system dimensions
- **Append-only** — new layers on top, never delete or overwrite
- **Uniform** — actor and structural edges have the same tensor shape

There are no per-action relationship types like FOLLOWS, LIKED, or CREATED.
Actor edges share one `:ACTOR` label and structural edges have a small fixed
sub-label set (see [edges.md §3](../primitive/edges.md)). The meaning of any
single edge is derived from the node types at each end and the dimension
values, not from a per-action relationship name. See
[Graph Model](../primitive/graph-model.md).

### 5. Writes are dual (content + topology)

When a user creates a post:
- Postgres: insert row into `posts` table (content + metadata)
- Memgraph: create Post node + actor edge from User to Post (layer 1 =
  authorship, with dim1/dim2 values)

When a user reacts to a post:
- Memgraph: create actor edge from User to Post (or add a layer if the edge
  exists) with the user's dim1/dim2 values for that node type
- Postgres: nothing

When a user expresses a stance toward another user:
- Memgraph: create/update actor edge from User to User with dim1/dim2 values
  (per [edges.md](../primitive/edges.md): sentiment + interest for this edge
  type)
- Postgres: nothing (unless profile display data changes)

---

## Components

### `crates/api`

The public-facing binary. Responsibilities:
- Starts the Axum HTTP server
- Hosts the async-graphql schema at `/graphql`
- Hosts the GraphQL playground at `/playground` (dev only)
- Holds connection pools for both databases
- Calls `graph-engine` and `postgres-store` to fulfill resolvers
- No business logic — it orchestrates, it does not decide

### `crates/graph-engine`

The Memgraph access layer. Responsibilities:
- Owns the `neo4rs::Graph` connection pool
- Exposes typed Rust functions for every Cypher query
- All Cypher strings live here, nowhere else
- Returns domain types from `common`, not raw graph results

### `crates/postgres-store`

The PostgreSQL access layer. Responsibilities:
- Owns the `sqlx::PgPool`
- Exposes typed Rust functions for every SQL query
- All SQL strings live here, nowhere else
- Manages migrations via SQLx

### `crates/common`

Shared types with no external dependencies. Responsibilities:
- Domain model structs (node types, edge types)
- Shared error types
- No database or HTTP logic

---

## Request Lifecycle: Feed Query

A personalized feed splits across two locations: the central backend
serves the **data**; the viewer's device computes the **ranking**.
This split is structural, not an optimization — per-actor ranking
cannot run on the central hot path at any real user count. See
[feed-ranking.md §9](../primitive/feed-ranking.md) for the full
reasoning and the math/deployment separation.

```
Phase 1 — central backend serves subgraph + seen-list

1. Client → POST /graphql to fetch the viewer's relevant graph
   slice.
2. API calls graph-engine: traverse N hops outward from the
   viewing user; return the relevant subgraph (nodes + their
   incident actor and structural edges, with top-layer tensor
   values intact).
3. API calls postgres-store: fetch the viewer's seen-list from
   user_view_log — a per-viewer set of already-shown content
   UUIDs. See feed-ranking.md §8.
4. API returns subgraph + seen-list to the client.

Phase 2 — viewer-side ranking and filtering

5. Client filters the subgraph by node type (per
   feed-ranking.md §9, "Filtering sits alongside ranking") and
   removes seen content from the candidate set (pre-rank
   exclusion per feed-ranking.md §8).
6. Client runs the feed-ranking algorithm (feed-ranking.md
   §1–§5) over the remaining candidates: a single sort by h(t)
   with cumulative tie-breakers (h, h+i, h+i+j, h+i+j+k), and
   S(t) as the final fallback. Output: an ordered list of
   node IDs.

Phase 3 — display-content fetch and render

7. Client requests display metadata for the top-N items it is
   about to render (post bodies, profile fields, media URLs)
   from postgres-store via the API.
8. Client renders. As items pass through the viewport, the
   client batches their IDs and POSTs them back to
   user_view_log on natural checkpoints (batch-fill, scroll
   pause, app close). See feed-ranking.md §8.
```

The central backend serves graph slices, seen-lists, and display
content; it does **not** rank. Ranking and filtering live on the
viewer's side — client by default, an optional delegate "miner" in
the future, both running the same algorithm (feed-ranking.md §9).
That is what keeps per-actor compute off the central hot path.

---

## Infrastructure

```
Local dev (Docker Compose):
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ┌─────────────────┐      ┌─────────────────────────┐   │
│  │  gnp_postgres   │      │      gnp_memgraph        │   │
│  │  postgres:16    │      │  memgraph-platform:latest│   │
│  │  port 5432      │      │  bolt: 7687              │   │
│  └─────────────────┘      │  lab:  3000              │   │
│                           └─────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

Volumes are named (`postgres_data`, `memgraph_data`) so data persists across
`make down` / `make up`. Use `make reset-db` to wipe everything.
