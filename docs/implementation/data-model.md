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
    avatar_id     UUID        REFERENCES media_attachments(id),
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
    avatar_id     UUID        REFERENCES media_attachments(id),
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

-- Media attachments: asset metadata only (URL, mime, size, alt text,
-- display options, uploader). Parents (posts, comments, chat messages,
-- items, users, collectives, chats) point at attachments via either a
-- junction table (1:N) or a direct FK column (1:1). The asset row
-- never points at a parent — see "Why parents point at attachments"
-- below.
--
-- options carries display hints the frontend reads to lay out the
-- container before the media finishes loading: aspect ratio,
-- autoplay/mute/loop flags, captions config, etc. JSONB so it can
-- grow without migrations as new hints are needed.
--
-- author_id + author_type identifies the uploader. Unlike posts.author_id
-- (which is a graph-derived cache), this column is Postgres-native source
-- of truth — Media is not a graph node, so there is no rebuild-from-graph
-- path. Used by the API to enforce that only the uploader's own parents
-- can reference an asset (anti-hijack), and to find an actor's media
-- when redacting their account (see instances/account-deletion.md).
CREATE TABLE media_attachments (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    author_id   UUID         NOT NULL,
    author_type TEXT         NOT NULL CHECK (author_type IN ('user', 'collective')),
    url         TEXT         NOT NULL,
    mime_type   TEXT         NOT NULL,
    size_bytes  BIGINT,
    alt_text    TEXT,
    options     JSONB        NOT NULL DEFAULT '{}'::jsonb,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
CREATE INDEX media_attachments_author_idx
    ON media_attachments (author_type, author_id);

-- Junction: posts → attachments (ordered, optionally a cover).
-- display_order and is_cover are parent-specific facts about the
-- relationship, not properties of the asset.
CREATE TABLE post_attachments (
    post_id       UUID     NOT NULL REFERENCES posts(id),
    attachment_id UUID     NOT NULL REFERENCES media_attachments(id),
    display_order SMALLINT NOT NULL DEFAULT 0,
    is_cover      BOOLEAN  NOT NULL DEFAULT FALSE,
    PRIMARY KEY (post_id, attachment_id)
);

-- Junction: comments → attachments (ordered).
CREATE TABLE comment_attachments (
    comment_id    UUID     NOT NULL REFERENCES comments(id),
    attachment_id UUID     NOT NULL REFERENCES media_attachments(id),
    display_order SMALLINT NOT NULL DEFAULT 0,
    PRIMARY KEY (comment_id, attachment_id)
);

-- Junction: chat messages → attachments (ordered).
CREATE TABLE chat_message_attachments (
    chat_message_id UUID     NOT NULL REFERENCES chat_messages(id),
    attachment_id   UUID     NOT NULL REFERENCES media_attachments(id),
    display_order   SMALLINT NOT NULL DEFAULT 0,
    PRIMARY KEY (chat_message_id, attachment_id)
);

-- Junction: items → attachments (ordered, optionally a cover).
CREATE TABLE item_attachments (
    item_id       UUID     NOT NULL REFERENCES items(id),
    attachment_id UUID     NOT NULL REFERENCES media_attachments(id),
    display_order SMALLINT NOT NULL DEFAULT 0,
    is_cover      BOOLEAN  NOT NULL DEFAULT FALSE,
    PRIMARY KEY (item_id, attachment_id)
);

-- Comments: responses to any commentable content node.
-- Comments are full nodes in the graph (can be liked, replied to).
-- target_id + target_type identify the parent — Post, Comment, Chat,
-- ChatMessage, or Item per edges.md §2 Containment. See
-- "target_id + target_type — discriminator, not foreign key" below for
-- why there is no SQL FK on this column.
CREATE TABLE comments (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    target_id   UUID        NOT NULL,
    target_type TEXT        NOT NULL CHECK (target_type IN
                            ('post', 'comment', 'chat', 'chat_message', 'item')),
    author_id   UUID        NOT NULL,
    author_type TEXT        NOT NULL CHECK (author_type IN ('user', 'collective')),
    content     TEXT        NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Chats: conversation containers.
-- Privacy is per-message (chat_messages.content_privacy), not per-chat —
-- a single chat can carry both plaintext and encrypted messages. See
-- chats.md §5.
CREATE TABLE chats (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT,       -- null for 1:1 chats
    description TEXT,
    image_id    UUID        REFERENCES media_attachments(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Chat messages: individual messages within a chat.
-- content_privacy is per-message (see chats.md §5): 'plaintext' bodies are
-- readable text; 'encrypted' bodies are ciphertext under the chat's
-- member-derived symmetric key for the epoch the message was authored in.
-- A chat can carry both freely.
--
-- epoch records which key the ciphertext is under (see chats.md §5: chat
-- keys are organized in epochs, advanced on membership change and on
-- passing mid-epoch rotation Proposals). NULL for plaintext rows; NOT NULL
-- for encrypted rows. The frontend uses it to pick the right key.
CREATE TABLE chat_messages (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    chat_id         UUID        NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
    author_id       UUID        NOT NULL,
    author_type     TEXT        NOT NULL CHECK (author_type IN ('user', 'collective')),
    content         TEXT        NOT NULL,
    content_privacy TEXT        NOT NULL DEFAULT 'plaintext'
                                CHECK (content_privacy IN ('plaintext', 'encrypted')),
    epoch           INTEGER     CHECK (
                                  (content_privacy = 'plaintext' AND epoch IS NULL) OR
                                  (content_privacy = 'encrypted' AND epoch >= 1)
                                ),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
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

### Personal frontend state

A category of per-viewer tables whose role is to feed the viewer's
**frontend** (or their delegated miner) — not the graph. They share
three properties:

- **Per-viewer.** Each row belongs to one user.
- **Storage-location-flexible.** This Postgres table is the
  backend-side default for the central frontend. Self-hosted
  clients, on-device caches, and miners can keep the same data
  locally and pass it to the calculator as a JSON array; the
  shape is the same regardless of where the data came from.
- **Operational, not graph history.** Exempt from the append-only
  rule that governs edges, node properties, and Postgres-side
  display content (see [layers.md](../primitive/layers.md)). These
  tables can be compacted, pruned, or replaced without leaving a
  visible trace.

Instances below: the seen-list (`user_view_log`), the hidden-actors
list (`user_hidden_actors`, frontend-side "don't show me Bob's
content" — see [feed-ranking.md §3.5](../primitive/feed-ranking.md)),
the chat-read pointer (`chat_read_state`), and bookmarks
(`user_bookmarks`). Further per-viewer state slots in here as it's
designed.

```sql
-- View log: per-viewer record of which content nodes have been seen.
-- Used by the feed-ranking computation as an exclusion set
-- (see feed-ranking.md §8).
CREATE TABLE user_view_log (
    user_id        UUID        NOT NULL,
    content_id     UUID        NOT NULL,
    first_seen_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, content_id)
);
CREATE INDEX user_view_log_recency_idx
    ON user_view_log (user_id, first_seen_at);
```

The seen-list's compaction policy (1-year default, ~7 MB/active-
user-year bound, trade-off, frontend tunability) lives with the
seen-list mechanism in
[feed-ranking.md §8.5](../primitive/feed-ranking.md).

```sql
-- Hidden actors: per-viewer list of users/collectives the viewer
-- doesn't want in their feed. Applied as a post-rank exclusion
-- filter on the viewer's side (see feed-ranking.md §3.5).
-- hidden_type disambiguates which table the hidden_id refers to,
-- same shape as author_type / target_type elsewhere.
CREATE TABLE user_hidden_actors (
    viewer_id   UUID        NOT NULL,
    hidden_id   UUID        NOT NULL,
    hidden_type TEXT        NOT NULL CHECK (hidden_type IN ('user', 'collective')),
    hidden_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (viewer_id, hidden_id, hidden_type)
);

-- Chat read state: per-user, per-chat 'last read' pointer.
-- ChatMessages are timestamp-ordered, so a single TIMESTAMPTZ marks
-- where the user has read up to. Unread = messages with created_at
-- > last_read_at. UPSERTed each time the user reads further; the
-- row's most recent update IS last_read_at, so no separate
-- updated_at column is needed.
CREATE TABLE chat_read_state (
    user_id      UUID        NOT NULL,
    chat_id      UUID        NOT NULL,
    last_read_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, chat_id)
);

-- User bookmarks: per-viewer "save this for later" list. Private
-- state, never visible to other actors and never an input to the
-- ranking math (see graph-model.md §3 — bookmarking is a frontend
-- event, not a stance). content_id can be any node UUID; a
-- discriminator is intentionally not stored, mirroring user_view_log.
CREATE TABLE user_bookmarks (
    user_id       UUID        NOT NULL,
    content_id    UUID        NOT NULL,
    bookmarked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, content_id)
);
CREATE INDEX user_bookmarks_recency_idx
    ON user_bookmarks (user_id, bookmarked_at DESC);
```

---

### User preferences

Per-user settings stored backend-side so they cross devices.
**Storage location is not flexible** for this category (unlike the
"Personal frontend state" tables above): iOS App Store rules forbid
in-app changes to mature-content settings, so users adjust them in
the web UI and the setting carries over to mobile clients — which
means the central backend has to be the source of truth.

```sql
-- User preferences: per-user frontend settings the backend persists
-- so they cross devices (see section intro for the App Store
-- rationale).
--
-- content_filtering_severity_level: how aggressive the viewer wants
-- the sensitive-content filter to be. 0 = show everything,
-- 10 = strictest. NULL = unset (frontend default applies).
-- Sensitive-content classification itself is community-moderated;
-- the moderation mechanism lives in instances/moderation.md.
CREATE TABLE user_preferences (
    user_id                          UUID     PRIMARY KEY,
    content_filtering_severity_level SMALLINT CHECK (
        content_filtering_severity_level IS NULL OR
        (content_filtering_severity_level BETWEEN 0 AND 10)
    )
);
```

---

### Application registry

```sql
-- Versions: one row per release per client component. Lets the API
-- answer "what's the current version of backend/iOS/Android/web?"
-- and "where are the patch notes for version X?". Append-only —
-- each release adds a row; previous rows stay so past patch-note
-- links remain resolvable.
CREATE TABLE versions (
    component       TEXT        NOT NULL CHECK (component IN
                                ('backend', 'ios', 'android', 'web')),
    version         TEXT        NOT NULL,
    patch_notes_url TEXT,
    released_at     TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (component, version)
);
CREATE INDEX versions_current_idx
    ON versions (component, released_at DESC);
```

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

### author_id is a cached derivation — except for media_attachments

The `author_id` columns on `posts`, `comments`, and `chat_messages` are
caches of the authorship derivation. The graph is the source of truth; see
[authorship.md](../primitive/authorship.md) for the rule and the cache-
rebuild semantics.

`media_attachments.author_id` is the **exception**: Media is not a graph
node, so there is no graph-side authorship derivation to cache. The
column is Postgres-native source of truth. If it gets corrupted, the
recovery path is object-storage ACLs / upload logs — not the graph.

### author_id + author_type — discriminator, not foreign key

`posts.author_id`, `comments.author_id`, `chat_messages.author_id`, and
`media_attachments.author_id` each reference either `users.id` or
`collectives.id`. A standard SQL foreign key can't point to two tables,
so each of these tables carries an `author_type` discriminator alongside
`author_id` with a `CHECK` restricting it to `'user'` or `'collective'`.

There is deliberately **no FK** from these columns to either parent
table. For posts/comments/chat_messages, the graph is the source of
truth for authorship; Postgres `author_id` is a cache. For
media_attachments, the column is Postgres-native (per the note above)
but uses the same shape for uniformity. A real FK would buy DB-level
referential integrity at the cost of schema churn every time a new
actor type is added (e.g. a future self-hosted instance introducing
its own actor kind). For the cached cases, integrity is guaranteed by
the cache-rebuild path: if Postgres ever disagrees with the graph,
rebuild from the graph.

Reads that need the parent row join on `author_type`:

```sql
SELECT p.*, COALESCE(u.display_name, c.name) AS author_name
FROM posts p
LEFT JOIN users     u ON p.author_type = 'user'    AND u.id = p.author_id
LEFT JOIN collectives c ON p.author_type = 'collective' AND c.id = p.author_id;
```

### target_id + target_type — same shape, different reason

`comments.target_id` references either `posts.id`, `comments.id`,
`chats.id`, `chat_messages.id`, or `items.id` — see
[edges.md §2 Containment](../primitive/edges.md). A standard SQL
foreign key can't point to five tables, so the table carries a
`target_type` discriminator with a `CHECK` on the same five values
the graph uses.

The graph is the source of truth here too: a comment's parent is
encoded in the `Comment → Target :CONTAINMENT` structural edge.
Postgres `target_id` is a cache. Same cache-rebuild rule as
`author_id`: if the cache disagrees with the graph, rebuild from the
graph.

This is also why old `posts(id) ON DELETE CASCADE` and a separate
`parent_comment_id` column are gone: posts and comments are graph
nodes that are never deleted (per [layers.md §5](../primitive/layers.md)),
and reply chains live on the graph as `Comment → Comment`
containment edges — Postgres doesn't need a parallel column.

### Why parents point at attachments

Many parent types attach media: posts (galleries), comments,
chat messages, items, plus 1:1 cases (user avatar, collective
avatar, chat picture). The natural query is always parent →
attachments ("show me the media for this post"), never the
reverse. So:

- `media_attachments` holds **asset metadata only** — no parent
  reference on the asset itself. The asset row is a pure asset,
  reusable across the uploader's own parents.
- 1:N parents reference attachments via per-parent **junction
  tables** (`post_attachments`, `comment_attachments`,
  `chat_message_attachments`, `item_attachments`). One row per
  attachment-on-parent. Per-relationship facts (`display_order`,
  `is_cover`) live on the junction, not on the asset.
- 1:1 parents reference attachments via a direct FK column
  (`users.avatar_id`, `collectives.avatar_id`, `chats.image_id`).

Junctions cost more rows than an array column would, but each
junction row is FK-enforced, supports per-relationship metadata
without table churn, and makes "find all parents using
attachment X" a normal indexed lookup (relevant for ownership
tracing on account redaction — see
[account-deletion.md](../instances/account-deletion.md)).

**Anti-hijack** is enforced at the API layer: when a parent
references an attachment, the API checks
`attachment.author_id == parent.author_id` (and
`author_type` matches) before writing the junction row or FK.
Cross-author re-use of media isn't supported through this path —
sharing someone else's content goes via linking to their post,
not by referencing their asset directly.

---

## ID Strategy

1. UUIDs are generated in the **API layer** (Rust), not by the database.
2. The same UUID is written to Postgres and Memgraph in the same request.
3. Postgres uses `UUID` as the primary key type with a `DEFAULT
   gen_random_uuid()` fallback, but the API always supplies it explicitly.
   (Exception: hashtags drop the DEFAULT — see "Node identity strategies"
   below.)

For how UUIDs are stored on the Memgraph side and the per-label index
declarations, see [graph-data-model.md](graph-data-model.md).

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
