# Post

The **Post** is a content node — a piece of authored content
(text and/or attached media) authored by a User or Collective.
Posts are the primary public-content surface of the platform: they
are the canonical target the [feed-ranking](../primitive/feed-ranking.md)
algorithm orders, and they are what most opinion-bearing actor
edges in a typical instance point at.

This doc is the per-node catalog for the Post: how it is created,
what it carries on the graph and in Postgres, what edges it can
participate in, and how it ends. The mechanics those topics depend
on stay in their topical docs — this doc links rather than
duplicates.

---

## 1. Creation

A Post is created by a single authoring gesture from one actor —
either a User or a Collective. There is **no approval flow**:
unlike junction nodes (see
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows)),
a Post requires no second-party affirmation to come into existence.
The author's outgoing edge is the only edge needed to bring the
node into the graph.

The gesture writes three records atomically:

- A new `:Post` node on the graph.
- The Postgres `posts` row carrying the body and any attachments
  (see [data-model.md](../implementation/data-model.md)).
- An actor edge from the authoring actor toward the new Post
  node — the **authorship edge** (§5). Its `(dim1, dim2)` values
  are the author's initial opinion of their own content,
  typically high positive sentiment and relevance.

A Collective authoring a Post is the same gesture: the graph
records the Post as the Collective's, and the off-graph
authentication that produced it traces — possibly through nested
CollectiveMember chains — back to one or more Users with active
sessions per [user.md §1](../primitive/user.md#1-user-vs-collective)
and [auth.md](../implementation/auth.md). Members of the
Collective do not individually sign or approve the post; the
Collective's social-contract governance defines whether and how
member consent is required, per
[collectives.md](collectives.md).

---

## 2. Graph-side properties

A Post node carries the minimum the graph needs to traverse,
filter, and rank. Substance lives in Postgres (§3).

- **`moderation_status`** — `'normal'` / `'sensitive'` /
  `'illegal'`, default `'normal'`, layered. Universal across all
  user-input-bearing nodes; the per-node mechanics — set by a
  passing `'sensitive'` Proposal, auto-flipped to `'illegal'` by
  the redaction cascade — are described in
  [nodes.md "Universal: moderation_status"](../primitive/nodes.md#universal-moderation_status)
  and §6 below.

The Post body, attachments, and any other display content do
**not** live on the graph. Concrete property types and indexes
for the graph-side node live in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 3. Postgres-side content

A Post's substance is its body and attached media — both live in
Postgres, linked to the graph node by UUID. Edits to display
content are append-only per
[layers.md §4](../primitive/layers.md#4-layers-on-postgres-side-display-content):
a new version row, no overwrite.

- **`content`** — the body text. Stored on the `posts` row;
  see [data-model.md](../implementation/data-model.md).
- **Attachments** — images, videos, and other media via the
  `post_attachments` junction table, which carries
  per-attachment `display_order` and an optional `is_cover`
  flag. Each row references one `media_attachments` asset, owned
  by the same author as the Post (anti-hijack rule per
  [data-model.md "Why parents point at attachments"](../implementation/data-model.md#why-parents-point-at-attachments)).

The cached `posts.author_id` column is a derivation of the graph
authorship rule — see §5 and
[authorship.md "Caching"](../primitive/authorship.md#caching).

---

## 4. Edges

### As source (outgoing)

A Post is not an actor and authors no actor edges. It carries
two outgoing structural edge types, both system-created:

- **`Post → Hashtag` (`:TAGGING`)** — one edge per hashtag the
  post is tagged with. See
  [edges.md §2 "Tagging"](../primitive/edges.md#tagging). The
  Hashtag node is content-addressed by canonical name (per
  [data-model.md "Node identity strategies"](../implementation/data-model.md#node-identity-strategies)),
  so the same hashtag across instances resolves to the same
  node.
- **`Post → any node` (`:REFERENCES`)** — one edge per node the
  Post embeds, quotes, or mentions: another Post it quotes or
  cites (e.g. pointing at the original of a re-uploaded image),
  a User or Collective named in the body, a Proposal it
  campaigns for, etc. Hashtag is the one excluded target —
  body-tag hashtags go through `:TAGGING` (above) and a single
  structural edge per (source, target) pair is the rule. The
  carrier semantics, target catalog, and deferred traversal
  rules live in
  [edges.md §2 "Reference"](../primitive/edges.md#reference).

### As target (incoming)

A Post receives:

- **Actor edges** from Users and Collectives carrying
  `(sentiment, relevance)` per
  [edges.md §1](../primitive/edges.md#1-actor-edges) — the
  like/dislike surface plus per-viewer relevance, used by
  [feed-ranking](../primitive/feed-ranking.md) to weight the
  Post for each viewer. The earliest of these is the authorship
  edge (§5).
- **`Comment → Post` (`:CONTAINMENT`)** when a Comment is
  written on the Post. See
  [edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging).
- **`ChatMessage / Post / Comment → Post` (`:REFERENCES`)** when
  another content node embeds the Post — a chat message sharing
  the Post into a chat, another Post quoting or citing it (e.g.
  pointing at the original of a re-uploaded image), a Comment
  citing it. See
  [edges.md §2 "Reference"](../primitive/edges.md#reference) and
  [chats.md](chats.md) for the worked-out ChatMessage patterns
  (sharing a post into a chat, the personal-newsfeed shape).
- **`Proposal → Post` (`:TARGETS`)** when a moderation Proposal
  targets a property on the Post — `'sensitive'` against
  `moderation_status`, or `'illegal'` against a specific
  user-input field. See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting)
  and §6.

---

## 5. Authorship

A Post's author is the actor whose incoming actor edge has the
earliest layer-1 timestamp — the same rule that derives
authorship for every node type
([authorship.md](../primitive/authorship.md)). Because a Post has
no existence before its creation, the author's edge is always
the earliest incoming edge by construction.

The author's `(dim1, dim2)` on the authorship edge is just a
normal opinion edge — not a special "author" tag — typically
carrying high positive sentiment and relevance toward the
content the author just created.

On the graph, the authoring edge carries the `:AUTHOR`
sub-label — that is the only representation of authorship on the
graph side, and it is what the §5.2 friend-authored fresh-post
detection in
[feed-ranking.md](../primitive/feed-ranking.md#52-frontend-reordering-friend-authored-fresh-posts)
traverses. For Postgres-side display queries, `posts.author_id`
is cached on the row. Both are rebuildable from the graph; the
graph wins in any disagreement.

---

## 6. Lifecycle

Post nodes are **never deleted**. Per
[layers.md §5](../primitive/layers.md#5-deletion-policy), the
only permitted "removal" is in-place layer redaction on graph
properties or a tombstone version row on Postgres-side display
content; both preserve a visible record that the change
occurred.

Two redaction triggers apply to a Post today:

- **Moderation: `'sensitive'` classification.** A passing
  `'sensitive'` Proposal flips the top layer of `moderation_status`
  to `'sensitive'`. No redaction; display content stays. Each
  viewer's `content_filtering_severity_level` (see
  [data-model.md](../implementation/data-model.md) "User
  preferences") decides how aggressively the frontend filters
  the Post. Reversible by a counter-Proposal back to `'normal'`.
  See [moderation.md §1](moderation.md#1-the-two-classification-paths).
- **Moderation: `'illegal'` classification.** A passing
  `'illegal'` Proposal targets one of the Post's user-input
  fields — `content` (the body), `attachments` (every attached
  media), or the literal `'full'` shorthand for both — and
  fires the redaction cascade per
  [moderation.md §1](moderation.md#1-the-two-classification-paths):
  the Postgres body row is tombstoned with a version marker,
  affected `media_attachments` rows are tombstoned and assets
  removed from object storage, the redacted originals are
  written to the [retention archive](../primitive/retention-archive.md)
  under per-row legal hold, and the Post node's
  `moderation_status` is auto-flipped to `'illegal'`. The
  cascade does **not** propagate to descendants — a Post
  classified illegal does not redact its Comments or any
  ChatMessage that references it; each requires its own
  classification.

Account deletion of the Post's author does **not** by default
affect the Post's body, attachments, or graph node — identity
redaction targets the User node's PII only. The Post is
content-redacted only if the author opts in to the content-level
scope of [account-deletion.md](account-deletion.md).

The Post's UUID is stable across every redaction. Authorship
caches keyed on the UUID stay valid; the outgoing `:TAGGING`
and `:REFERENCES` edges and every incoming actor / containment
/ reference / targeting edge keep pointing at the same node. A
redacted Post is a partially-or-fully gutted but
still-graph-resident content node, not a removed one.

---

## What this doc is not

- **Not the feed-ranking spec.** Where a Post surfaces in any
  given viewer's feed — including the friend-authored fresh-post
  reorder layer, the community bot-defense / self-redemption
  usage conventions that ride on top of regular Posts, and the
  per-viewer filter and decay layers — lives in
  [feed-ranking.md](../primitive/feed-ranking.md).
- **Not the authorship rule.** The earliest-incoming-edge
  derivation, the cache rebuild semantics, and the worked
  example live in [authorship.md](../primitive/authorship.md).
- **Not the moderation primitive.** The Proposal mechanism,
  the mod gate, eligibility, thresholds, and the redaction
  cascade live in [moderation.md](moderation.md).
- **Not the deletion mechanism.** The redaction primitive lives
  in [layers.md §5](../primitive/layers.md#5-deletion-policy);
  the per-row legal hold and archive disposition live in
  [retention-archive.md](../primitive/retention-archive.md).
- **Not the edge catalog.** Per-target-type edges with
  dimension labels live in [edges.md](../primitive/edges.md).
- **Not the Memgraph or Postgres schema.** Concrete property
  types, columns, indexes, and the
  `post_attachments` / `media_attachments` shapes live in
  [graph-data-model.md](../implementation/graph-data-model.md)
  and [data-model.md](../implementation/data-model.md).
