# Architecture

## Overview

CoGra is a spike for **Peer Network** — a social media platform where
feed ranking is driven entirely by the social graph and explicit user
interactions, not AI algorithms.

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
| Actor edges (sentiment, relevance) | Memgraph | Graph topology + ranking |
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
- **Directional** — A -> B and B -> A are independent
- **Multi-dimensional** — 2 user dimensions + system dimensions
- **Append-only** — new layers on top, never delete or overwrite
- **Uniform** — actor and structural edges have the same tensor shape

There are no named relationship types like FOLLOWS, LIKED, or CREATED. All
edges are uniform tensors. The meaning is derived from the node types at each
end and the dimension values. See [Graph Model](../primitive/graph-model.md).

### 5. Writes are dual (content + topology)

When a user creates a post:
- Postgres: insert row into `posts` table (content + metadata)
- Memgraph: create Post node + actor edge from User to Post (layer 1 =
  authorship, with sentiment/relevance values)

When a user interacts with a post (e.g. likes it):
- Memgraph: create actor edge from User to Post (or add layer if edge exists)
  with the user's sentiment and relevance values
- Postgres: nothing

When a user interacts with another user:
- Memgraph: create/update actor edge from User to User with sentiment and
  interest values
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

A GraphQL query for a personalized feed shows how the two databases compose:

```
Client -> POST /graphql { feed(limit: 20) }

1. API receives query, calls postgres-store to fetch the viewer's
   seen-list (user_view_log) — a per-viewer set of already-shown
   content UUIDs. See feed-ranking.md §8.

2. API calls graph-engine, passing the seen-list as an exclusion
   parameter alongside R, decay shape, etc.

3. graph-engine: Cypher queries to Memgraph
   - Traverse outgoing edges from the viewing user
   - Exclude already-seen content nodes from the candidate set
     before computing metrics (pre-rank pruning)
   - For each remaining target node, compute ranking metrics
     (h, i, j, k) from the tensor edge dimensions
   - Sort by R (hops), then order by h -> h+i -> h+i+j -> h+i+j+k
   - Return ranked list of node IDs

4. API calls postgres-store with the ranked node IDs
5. postgres-store: SQL queries to Postgres
   - Fetch display metadata for the ranked nodes (post content,
     user profiles, media, etc.)

6. API merges results, preserving the graph-determined order
7. Returns JSON to client

8. Frontend renders, batches the IDs of items that pass through
   the viewport, and POSTs them back to user_view_log on natural
   checkpoints (batch-fill, scroll pause, app close).
```

The graph engine decides *which* nodes to show and in *what order* (topology
+ edge-weight-based ranking). Postgres tells us *what* those nodes contain
*and* tracks the per-viewer seen-list that prunes the candidate set up front.

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
