# Post

The **Post** is a content node — a piece of authored content
(text and/or attached media) authored by a User or Collective.
Posts are the primary public-content surface of the platform: they
are the canonical target the [feed-ranking](../primitive/feed-ranking.md)
algorithm orders, and they are what most opinion-bearing actor
edges in a typical instance point at.

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

The Post carries per-field moderation-status properties on
**`content`** (the body) and **`attachments`** (every attached
media under one status — see [moderation.md §5](moderation.md#5-scope)
on per-attachment targeting), plus the node-level
`moderation_status` cache. Universal mechanics in
[nodes.md](../primitive/nodes.md#universal-per-field-moderation-status);
Post-specific cascade in §6. Body and attachment content live in
Postgres / object storage (§3); concrete types and indexes in
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
  Post for each viewing user. The earliest of these is the authorship
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
  targets one of the Post's per-field moderation-status
  properties (§3). See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting);
  cascade in §6.

---

## 5. Authorship

A Post's author is the actor whose incoming actor edge has the
earliest layer-1 timestamp — the same rule that derives
authorship for every node type
([authorship.md](../primitive/authorship.md)). On the graph that
edge carries the `:AUTHOR` sub-label; the author's `(dim1, dim2)`
on the same edge are normal opinion values (sentiment / relevance),
not a stand-in for the label. The two coexist: the label marks
authorship, the dimensions carry the author's opinion of their
own work.

`:AUTHOR` is the only representation of authorship on the graph
side, and what the friend-authored fresh-post detection in
[feed-ranking.md §5.2](../primitive/feed-ranking.md#52-frontend-reordering-friend-authored-fresh-posts)
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

Redaction triggers on a Post are moderation
([moderation.md §1](moderation.md#1-the-two-classification-paths))
and — with the author's opt-in — content-level account deletion.

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
  given viewing user's feed — including the friend-authored fresh-post
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
