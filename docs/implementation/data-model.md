# Data Model — PostgreSQL

This document covers the **PostgreSQL schema** — the metadata/display layer.

For the graph model (nodes, edges, tensor dimensions, append-only layers),
see [Graph Model](../primitive/graph-model.md).

> **Note:** This schema is a starting point. The production Peer Network
> backend has an existing Postgres schema with additional display data tables
> that will need to be reviewed and integrated. See:
> https://github.com/peer-network/peer_backend/tree/main/sql_files_for_import

## The Boundary Rule

> If the data is needed to **navigate or weight** the graph → Memgraph.
> If the data is needed to **display** something → Postgres.

UUIDs are the shared key. Both databases store the same ID for the same
entity; neither database stores the other's fields.

---

## PostgreSQL Schema

Postgres holds all human-readable metadata. It knows nothing about the social
graph, edge weights, or feed ranking. Every table here exists to answer the
question: "given a UUID, what do I render on screen?"

### Actor metadata

```sql
-- Users: identity and profile display data
CREATE TABLE users (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    username      TEXT        NOT NULL UNIQUE,
    display_name  TEXT        NOT NULL,
    bio           TEXT,
    avatar_url    TEXT,
    website_url   TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Collectives: profiles for any collective actor (households, bands, co-ops, companies, ...)
CREATE TABLE collectives (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name          TEXT        NOT NULL UNIQUE,  -- handle for mentions/lookups, analogous to users.username
    display_name  TEXT        NOT NULL,
    description   TEXT,
    avatar_url    TEXT,
    website_url   TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Content metadata

```sql
-- Posts: content authored by users or collectives
CREATE TABLE posts (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    author_id   UUID        NOT NULL,
    author_type TEXT        NOT NULL CHECK (author_type IN ('user', 'collective')),
    content     TEXT        NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Media attached to posts (images, videos)
CREATE TABLE media_attachments (
    id            UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id       UUID         NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    url           TEXT         NOT NULL,
    mime_type     TEXT         NOT NULL,
    size_bytes    BIGINT,
    alt_text      TEXT,
    display_order SMALLINT     NOT NULL DEFAULT 0
);

-- Comments: responses to posts or other comments
-- Comments are full nodes in the graph (can be liked, replied to)
CREATE TABLE comments (
    id                UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id           UUID        NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id         UUID        NOT NULL,
    author_type       TEXT        NOT NULL CHECK (author_type IN ('user', 'collective')),
    parent_comment_id UUID        REFERENCES comments(id),
    content           TEXT        NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Chats: conversation containers
CREATE TABLE chats (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT,       -- null for 1:1 chats
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Chat messages: individual messages within a chat
CREATE TABLE chat_messages (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    chat_id     UUID        NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
    author_id   UUID        NOT NULL,
    author_type TEXT        NOT NULL CHECK (author_type IN ('user', 'collective')),
    content     TEXT        NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Items: physical or digital goods (future)
CREATE TABLE items (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT        NOT NULL,
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Hashtag registry (name lookup + metadata)
-- id is derived via UUIDv5 from the canonical name (see "Node identity
-- strategies" below). No DEFAULT — the API must always supply the
-- deterministic UUID; relying on a random fallback would break content-
-- addressing.
CREATE TABLE hashtags (
    id         UUID        PRIMARY KEY,
    name       TEXT        NOT NULL UNIQUE,  -- stored lowercase, no '#'
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Per-viewer ranking-input state

```sql
-- View log: per-viewer record of which content nodes have been seen.
-- Used by the feed-ranking computation as an exclusion set
-- (see feed-ranking.md §8).
--
-- Storage location is the viewer's choice — this table is the
-- backend-side default for the central frontend. Self-hosted
-- clients and miners can keep the same data locally and pass it
-- to the calculator as a JSON array; the math is the same.
CREATE TABLE user_view_log (
    user_id        UUID        NOT NULL,
    content_id     UUID        NOT NULL,
    first_seen_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, content_id)
);
CREATE INDEX user_view_log_recency_idx
    ON user_view_log (user_id, first_seen_at);
```

Unlike display content (which follows the append-only versioning
rule from [layers.md](../primitive/layers.md)), `user_view_log` is
**operational filter state**, not graph history. The full
compaction policy (1-year default, ~7 MB/active-user-year bound,
trade-off, frontend tunability) lives with the seen-list mechanism
in [feed-ranking.md §8.5](../primitive/feed-ranking.md).

---

## What is intentionally NOT in Postgres

- **Edge data** (sentiment, interest, relevance, layers) — graph-only
- **Feed ordering / ranking** — graph-only
- **Interaction history** (who reacted to what, who interacted with whom) —
  graph-only (encoded in tensor edges)
- **Counts** (inbound edges, reactions, comments) — derived from graph edges
  at query time, not materialized
- **Membership / ownership state** — graph-only (junction nodes: ChatMember,
  CollectiveMember, ItemOwnership)

---

## Notes

### author_id is a cached derivation

The `author_id` columns on `posts`, `comments`, and `chat_messages` are
caches of the authorship derivation. The graph is the source of truth; see
[authorship.md](../primitive/authorship.md) for the rule and the cache-
rebuild semantics.

### author_id + author_type — discriminator, not foreign key

`posts.author_id`, `comments.author_id`, and `chat_messages.author_id`
each reference either `users.id` or `collectives.id`. A standard SQL
foreign key can't point to two tables, so each of these tables carries
an `author_type` discriminator alongside `author_id` with a `CHECK`
restricting it to `'user'` or `'collective'`.

There is deliberately **no FK** from these columns to either parent
table. The graph is the source of truth for authorship; Postgres
`author_id` is a cache. A real FK would buy DB-level referential
integrity at the cost of schema churn every time a new actor type is
added (e.g. a future self-hosted instance introducing its own actor
kind). Integrity is guaranteed by the cache-rebuild path instead: if
Postgres ever disagrees with the graph, rebuild from the graph.

Reads that need the parent row join on `author_type`:

```sql
SELECT p.*, COALESCE(u.display_name, c.name) AS author_name
FROM posts p
LEFT JOIN users     u ON p.author_type = 'user'    AND u.id = p.author_id
LEFT JOIN collectives c ON p.author_type = 'collective' AND c.id = p.author_id;
```

---

## ID Strategy

1. UUIDs are generated in the **API layer** (Rust), not by the database.
2. The same UUID is written to Postgres and Memgraph in the same request.
3. Postgres uses `UUID` as the primary key type with a `DEFAULT
   gen_random_uuid()` fallback, but the API always supplies it explicitly.
   (Exception: hashtags drop the DEFAULT — see "Node identity strategies"
   below.)
4. Memgraph nodes store the UUID as a `String` property named `id`.
5. Memgraph indexes: `CREATE INDEX ON :User(id)`, `CREATE INDEX ON :Post(id)`,
   etc. for all node types.

---

## Node identity strategies

Different node types have different *kinds* of identity. The data model
uses three strategies, chosen per type based on what the node
fundamentally *is*. Stating the strategies explicitly here so future node
types are designed against a conscious choice rather than schema
intuition.

### Type 1 — Identity is a canonical string

A node whose existence *is* a string concept. Two creations of the same
string should converge on one node, no matter where in the graph (or
which forked instance) they happen.

- **Hashtag**: a hashtag is its name. `#bot-defense` is one concept; the
  Postgres table forbids two rows with the same canonical name.

For these types, the UUID is **content-addressed**: derived
deterministically from the canonical string via
`UUIDv5(HASHTAG_NAMESPACE, canonical_name)` with a fixed project-scoped
namespace UUID. Same name → same UUID *across any instance or fork*. The
UUID is mathematically redundant with the name (both encode the same
identity), but the UUID is still the database key and the bridge between
Postgres and Memgraph (per "The Boundary Rule" earlier in this doc).

The canonical-name normalization (currently for hashtags: lowercase, no
`#`) is **load-bearing**: it determines what counts as "the same"
hashtag. Changing the normalization later would invalidate previously-
minted UUIDs. Treat the normalization as part of the schema, not a UI
affordance.

The namespace UUID is fixed at the project level and **never changes**.
Changing it would break every previously-derived hashtag UUID.
Implementation MUST commit the namespace value to source so all
instances and forks compute identical UUIDs.

Federation across separated instances of these types requires **no
reconciliation** — instances independently compute the same UUIDs from
the same names by construction.

### Type 2 — Identity is a chosen handle (display label)

A node that has a UNIQUE display handle within an instance, but the
handle is a label, not the deep identity. Two separate humans named
"alice" are two different users; they should not collapse to one node
just because they picked the same handle.

- **User**: identified by `users.id` (UUID). `username` is UNIQUE per
  instance for cross-reference (`@alice`) but is not the user's
  identity.
- **Collective**: identified by `collectives.id` (UUID). `name` is
  UNIQUE per instance, same shape — analogous to `users.username`.

UUIDs for these types are **random** (`gen_random_uuid()`). The UNIQUE
constraint on the handle prevents within-instance collision.

Federation across separated instances requires explicit reconciliation
for the handle: instance A's `@alice` and instance B's `@alice` could be
the same person or two different people. A federation protocol must
decide. Tracked as a forward question in
[open-questions.md](../open-questions.md) (Q15).

### Type 3 — Identity is per-creation

A node that is a discrete thing brought into existence at a specific
moment. There is no canonical concept the node "represents"; every
creation is its own node.

- **Post, Comment, ChatMessage**: a piece of content authored at a
  specific time. Two posts with identical text by different authors are
  different posts.
- **Chat**: a conversation container. Two chats with the same title are
  different chats.
- **Item**: a goods entry.
- **Junction nodes** (ChatMember, CollectiveMember, ItemOwnership,
  Proposal, etc.): represent a relationship instance.

UUIDs for these types are **random**. There is no UNIQUE constraint on
any user-facing field; identity is the UUID alone.

Federation across separated instances requires reconciliation only for
*cross-references* (e.g. a post in instance A referenced by content in
instance B). Same Q15 as type 2.

### When adding a new node type

Decide which strategy applies first. The choice determines the schema
(UNIQUE constraint? content-addressed UUID? random UUID with no
constraint?) and the cross-instance behavior (free dedup vs.
reconciliation needed). Recording the strategy alongside the new node
type in [nodes.md](../primitive/nodes.md) keeps the conscious choice
visible to future readers.
