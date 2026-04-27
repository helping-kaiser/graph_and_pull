# Data Model — PostgreSQL

This document covers the **PostgreSQL schema** — the metadata/display layer.

For the graph model (nodes, edges, tensor dimensions, append-only layers),
see [Graph Model](graph-model.md).

> **Note:** This schema is a starting point. The production Peer Network
> backend has an existing Postgres schema with additional display data tables
> that will need to be reviewed and integrated. See:
> https://github.com/peer-network/peer_backend/tree/main/sql_files_for_import

## The Boundary Rule

> If the data is needed to **navigate or weight** the graph -> Memgraph.
> If the data is needed to **display** something -> Postgres.

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
    name          TEXT        NOT NULL,
    handle        TEXT        NOT NULL UNIQUE,
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
    id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name       TEXT,       -- null for 1:1 chats
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
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
CREATE TABLE hashtags (
    id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name       TEXT        NOT NULL UNIQUE,  -- stored lowercase, no '#'
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## What is intentionally NOT in Postgres

- **Edge data** (sentiment, relevance, closeness, layers) — graph-only
- **Feed ordering / ranking** — graph-only
- **Interaction history** (who liked what, who interacted with whom) —
  graph-only (encoded in tensor edges)
- **Counts** (followers, likes, comments) — derived from graph edges at query
  time, not materialized
- **Membership / ownership state** — graph-only (junction nodes: ChatMember,
  CollectiveMember, ItemOwnership)

---

## Notes

### author_id is a cached derivation

The `author_id` columns on `posts`, `comments`, and `chat_messages` are
**caches** of a fact that lives in the graph. The true author is the actor
whose incoming edge to the node has the earliest layer 1 timestamp (see
[authorship.md](authorship.md)). The
Postgres column exists because "who wrote this?" is asked on every render and
scanning all incoming edges every time would be expensive.

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
4. Memgraph nodes store the UUID as a `String` property named `id`.
5. Memgraph indexes: `CREATE INDEX ON :User(id)`, `CREATE INDEX ON :Post(id)`,
   etc. for all node types.
