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
in a graph DB and display content in Postgres lets each database do what it
was built for.

### Vocabulary: display content vs metadata

These describe a value's *purpose*, not its storage location, and
the two categories overlap:

- **Display content** — data that UIs render to the viewing user
  (post bodies, message bodies, profile text, attachment URLs,
  display names). Mostly Postgres rows.
- **Metadata** — data that drives flows rather than being rendered:
  edge weights and layer history, junction approval state,
  moderation flags, retention bookkeeping. Mostly graph state.

A single value can be both — a `ChatMember.role` lives on the
graph (where it weights governance tallies) and a UI may also
display it next to the member's name. The labels say *what the
value is used for*, not which database it sits in.

---

## At a glance

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
                   │     Memgraph        │             │      PostgreSQL       │
                   │   (graph layer)     │             │ (display-content layer)│
                   └─────────────────────┘             └───────────────────────┘
```

The shared key between both databases is the **UUID** assigned at creation
time. Memgraph stores graph topology (nodes + tensor edges). PostgreSQL stores
everything needed to display content. See the
[Components](#components) section below for what each crate does.

| Concern | Choice |
|---|---|
| Language | Rust 2021 |
| API | Axum + async-graphql |
| Graph DB | Memgraph (openCypher, bolt protocol) |
| Display-content DB | PostgreSQL 16 (SQLx) |
| Local dev | Docker Compose |
| CI | GitHub Actions |

---

## Design Principles

### 1. Graph DB owns topology, Postgres owns content

**Invariant:** Memgraph owns graph topology; Postgres owns display
content; UUIDs are the shared key. No content in Memgraph; no
topology in Postgres.

If a piece of data is needed to **navigate or weight** the graph, it goes in
Memgraph. If it is needed to **display** something, it goes in Postgres.

| Data | Where | Why |
|---|---|---|
| Node ID (UUID) | Both | Shared key between databases |
| User bio, avatar, display name | Postgres | Display only |
| Post content, media URLs | Postgres | Display only |
| Actor edges (dim1, dim2 — see [edges.md](../primitive/edges.md) for per-edge-type labels) | Memgraph | Graph topology + ranking |
| Structural edges (containment, tagging) | Memgraph | Graph topology |
| Authorship | Memgraph (`:AUTHOR` sub-label on the authoring actor edge); Postgres (`author_id` column on `posts` / `comments` / `chat_messages` for display) | Derived from the earliest incoming layer-1 edge; see [authorship.md](../primitive/authorship.md) |

See [Graph Model](../primitive/graph-model.md) for the full node/edge
specification.

### 2. UUIDs as the shared key

Every entity gets a UUID at creation time. This UUID is stored in both
databases and is the only way they reference each other. The graph engine
never needs to know a username; the Postgres store never needs to know the
graph topology.

### 3. All ranking comes from the graph

**Invariant:** All feed ranking is computed at query time from
the edge tensor. There are no materialized counters, popularity
scores, or algorithm-driven signals stored as node properties.
The algorithm itself — its parameters, sort order, and
tie-breaker chain — lives in
[feed-ranking.md](../primitive/feed-ranking.md); this doc covers
only how the system runs it (per-viewer, off the central hot
path).

### 4. Edges are the source of truth

All graph state lives in edges. Edges are:
- **Directional** — A → B and B → A are independent
- **Multi-dimensional** — 2 user dimensions + system dimensions
- **Append-only** — new layers on top, never delete or overwrite
- **Uniform in shape, not in meaning** — actor and structural
  edges share the same tensor shape (so the ranking algorithm
  never branches on edge category) but their dimensions carry
  completely different semantics. Actor-edge dimensions are
  signed valence and connection-weight a user expressed
  toward a target; structural-edge dimensions are typically
  `0` or carry approval-pair state. Same struct, different
  reading. See [graph-model.md §3](../primitive/graph-model.md#3-edge-categories).

There are no per-action relationship types like FOLLOWS, LIKED, or CREATED.
Actor edges share one `:ACTOR` label and structural edges have a small fixed
sub-label set (see [edges.md §3](../primitive/edges.md#3-edge-labels-at-the-graph-layer)). The meaning of any
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

## Service-layer transactions

Every write that touches more than one store, or more than one
node in Memgraph that must succeed or fail together, runs inside
a **single service-layer transaction** wrapped by the API. The
service layer is the only place with handles on both pools, so it
is the only place that can hold a Memgraph transaction and a
Postgres transaction open simultaneously and commit them as one
logical unit.

### Partial-failure handling

Two engines have two commit boundaries; an inter-commit window
exists in which the first store has committed and the second
has not. The pattern that closes the window is:

- **Hold both transactions open** through every write. Any
  error before either commit aborts both with a `ROLLBACK`; the
  graph and the display-content row stay in pre-write state.
- **Choose an order so the first-committed side is
  idempotent on retry.** Graph writes use Cypher `MERGE` keyed
  on the node UUID rather than `CREATE` — a retried
  service-layer transaction collapses a duplicate Memgraph
  commit into a no-op. Postgres inserts are paired with
  `ON CONFLICT DO NOTHING` (or equivalent) on the relevant
  primary key.
- **Place the lower-risk commit last.** The order is chosen so
  the more failure-prone engine commits first; if it fails,
  rollback is clean. The second commit is then close to
  guaranteed; if it does fail, the idempotency above lets the
  caller retry safely.

A two-phase commit primitive across the two engines is **not**
in scope: implementation cost outweighs the gain at our scale.
Idempotent retry + the cache-rebuild path
([data-model.md](data-model.md#author_id-is-a-cached-derivation--except-for-media_attachments))
together cover the residual inconsistency surface.

Every dual-store write follows this shape: User registration
(below), Post / Comment / ChatMessage authoring (graph node +
Postgres body row), the redaction cascade (graph layer +
Postgres archive row — and archive-first per
[governance.md §6 "Cascade dispatch"](../primitive/governance.md#6-when-outcomes-take-effect)),
account deletion (graph redactions + Postgres row clears).

### Genesis bootstrap

The instance bootstrap migration is the system's clearest example
of the pattern. It writes three nodes — the `:Network` singleton,
the genesis User, and the `bot-defense` Hashtag — and all three
go in one transaction. See
[network.md §2](../primitive/network.md#2-creation) for the
primitive-side framing of what the migration produces.

Because no graph exists until the transaction commits, no hostile
Proposal can race the bootstrap: there is no target to file
against, no Network singleton to scope to, no eligibility set to
vote from. The pre-graph window is fully closed. After commit, the
graph is in a complete state — singleton + bootstrap moderator +
bot-defense Hashtag — and ordinary governance applies from there.

The migration is the **only** writer of these three nodes; no
runtime path produces a second `:Network` or a second genesis
User. It is also the only step in the system that escapes the
actor-gesture-or-governance rule (per
[graph-model.md §1](../primitive/graph-model.md#1-core-principles)),
and that escape is confined to the migration.

### Cascade handler

Every governance threshold-cross — moderation classification,
member disavowal, eligibility dropout, Chat epoch rotation, Item
ownership transfer — fans out through the **cascade handler**: a
single dispatch module in the API service layer that sequences
the derived writes for each cascade type. See
[governance.md §6](../primitive/governance.md#6-when-outcomes-take-effect)
for the mechanism.

The handler runs **synchronously** inside the same service-layer
transaction as the triggering vote layer. Per cascade type it
knows the fan-out order; **archive writes precede graph
mutations** (so a failed archive never leaves a redacted layer
without an archive copy); any step failure **rolls back the
whole transaction**, including the triggering write.

The handler lives in `api/` (orchestration). It calls into
`graph-engine` for the Cypher writes and `postgres-store` for
the archive rows; per the code-style rules in CLAUDE.md, the
cascade module sequences the calls but holds no DB-specific code
itself.

### User registration (invitation acceptance)

Email verification creates the User node, its invitation edges,
and the first session in one service-layer transaction. The
trigger is the invitee clicking the verification link with the
single-use token written to the `auth_pending_registrations`
row at registration submit (see
[auth.md "Invitation acceptance"](auth.md#invitation-acceptance-the-default-path)).

Inside the transaction:

1. Validate the verification token (Postgres read).
2. Read the inviter's pre-committed `(dim1, dim2)` from the
   linked `auth_invitations` row (Postgres read).
3. Create the User node in Memgraph with `network_role =
   'member'` and layered properties initialized.
4. Write the two invitation actor edges per
   [invitations.md](../primitive/invitations.md) — inviter
   value outward, invitee value back.
5. Insert the first `auth_refresh_tokens` row (Postgres).
6. Delete the `auth_pending_registrations` row (Postgres).

The order makes each step rollback-safe. If any step fails the
service layer rolls back both pools' transactions; the pending
registration row survives so the invitee can retry.

There is **no observable window** in which the User node exists
without its invitation edges or in which a session token is
issued before the User. The
[no-User-node-before-verification invariant in
user.md §2](../primitive/user.md#2-creation) holds at the
implementation level because of this ordering.

---

## Request Lifecycle: Feed Query

A personalized feed splits across two locations: the central backend
serves the **data**; the viewing user's device computes the **ranking**.
This split is structural, not an optimization — per-actor ranking
cannot run on the central hot path at any real user count. See
[feed-ranking.md §9](../primitive/feed-ranking.md#9-where-ranking-and-filtering-live) for the full
reasoning and the math/deployment separation.

```
Phase 1 — central backend serves subgraph + seen-list

1. Client → POST /graphql to fetch the viewing user's relevant graph
   slice.
2. API calls graph-engine: traverse N hops outward from the
   viewing user; return the relevant subgraph (nodes + their
   incident actor and structural edges, with top-layer tensor
   values intact).
3. API calls postgres-store: fetch the viewing user's seen-list from
   user_view_log — a per-viewer set of already-shown content
   UUIDs. See feed-ranking.md §8.
4. API returns subgraph + seen-list to the client.

Phase 2 — viewer-side ranking and filtering

5. Client filters the subgraph by node type (per
   feed-ranking.md §9, "Filtering sits alongside ranking") and
   removes seen content from the candidate set (pre-rank
   exclusion per feed-ranking.md §8).
6. Client runs the feed-ranking algorithm (feed-ranking.md
   §1–§5) over the remaining candidates. Output: an ordered
   list of node IDs.

Phase 3 — display-content fetch and render

7. Client requests display content for the top-N items it is
   about to render (post bodies, profile fields, media URLs)
   from postgres-store via the API.
8. Client renders. As items pass through the viewport, the
   client batches their IDs and POSTs them back to
   user_view_log on natural checkpoints (batch-fill, scroll
   pause, app close). See feed-ranking.md §8.
```

The central backend serves graph slices, seen-lists, and display
content; it does **not** rank. Ranking and filtering live on the
viewing user's side — client by default, an optional delegate "miner" in
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
