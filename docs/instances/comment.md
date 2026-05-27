# Comment

The **Comment** is a content node — a response authored by a
User or Collective on another content node. Comments are the
platform's universal threading primitive: they attach to Posts,
to other Comments (replies), to Chats and individual
ChatMessages, and to Items, layering a discussion surface onto
every kind of content the graph holds. Comments are full graph
nodes, not properties of their target — so they can themselves
be liked, disliked, replied to, embedded, and moderated, with
their own authored opinion edges and their own per-field
moderation-status properties.

---

## 1. Creation

A Comment is created by a single authoring gesture from one
actor — either a User or a Collective — toward exactly one
**target content node**. There is **no approval flow**: like a
Post (see [post.md §1](post.md#1-creation)) and unlike junction
nodes (see [graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows)),
a Comment requires no second-party affirmation.

The valid target set — **Post, Comment, Chat, ChatMessage, or
Item** — is the most distinctive thing about Comments: they are
the platform's universal threading primitive, not a Post-only
concept. The canonical per-target list with edge meanings lives
in [edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging).

The gesture writes four records atomically:

- A new `:Comment` node on the graph.
- An outgoing **`Comment → Target`** `:CONTAINMENT` structural
  edge identifying the parent. Exactly one per Comment; fixed
  at creation and not re-targeted later.
- The Postgres `comments` row carrying the body, any
  attachments, and the cached parent reference (see §3 and
  [data-model.md](../implementation/data-model.md)).
- An actor edge from the authoring actor toward the new Comment
  node — the **authorship edge** (§5). Its `(dim1, dim2)`
  values are the author's initial opinion of their own
  response, typically high positive sentiment and relevance.

A Collective authoring a Comment is the same gesture: the graph
records the Comment as the Collective's, and the off-graph
authentication that produced it traces — possibly through
nested CollectiveMember chains — back to one or more Users with
active sessions per
[user.md §1](../primitive/user.md#1-user-vs-collective) and
[auth.md](../implementation/auth.md). Members of the Collective
do not individually sign or approve the Comment; the
Collective's social-contract governance defines whether and how
member consent is required, per
[collectives.md](collectives.md).

---

## 2. Graph-side properties

A Comment node carries the minimum the graph needs to traverse,
filter, and rank. Substance lives in Postgres (§3).

The Comment carries per-field moderation-status properties on
**`content`** (the body) and **`attachments`** (every attached
media under one status — see
[moderation.md §5](moderation.md#5-scope) on per-attachment
targeting), plus the node-level `moderation_status` cache.
Universal mechanics in
[nodes.md](../primitive/nodes.md#universal-per-field-moderation-status);
Comment-specific cascade in §6. Body and attachment content live
in Postgres / object storage (§3); concrete types and indexes in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 3. Postgres-side content

A Comment's substance is its body and attached media — both
live in Postgres, linked to the graph node by UUID. Edits to
display content are append-only per
[layers.md §4](../primitive/layers.md#4-layers-on-postgres-side-display-content):
a new version row, no overwrite.

- **`content`** — the body text. Stored on the `comments` row;
  see [data-model.md](../implementation/data-model.md).
- **Attachments** — images, videos, and other media via the
  `comment_attachments` junction table, which carries
  per-attachment `display_order`. Each row references one
  `media_attachments` asset, owned by the same author as the
  Comment (anti-hijack rule per
  [data-model.md "Why parents point at attachments"](../implementation/data-model.md#why-parents-point-at-attachments)).

The `comments` row also caches `target_id` + `target_type` as
a discriminator pointer to the parent — Post, Comment, Chat,
ChatMessage, or Item. The graph
(`Comment → Target :CONTAINMENT`) is the source of truth; the
Postgres columns are caches rebuildable from the graph. See
[data-model.md "target_id + target_type"](../implementation/data-model.md#target_id--target_type--same-shape-different-reason).

The cached `comments.author_id` column is a derivation of the
graph authorship rule — see §5 and
[authorship.md "Caching"](../primitive/authorship.md#caching).

---

## 4. Edges

### As source (outgoing)

A Comment is not an actor and authors no actor edges. It
carries three outgoing structural edge types, all
system-created:

- **`Comment → (Post | Comment | Chat | ChatMessage | Item)`
  `:CONTAINMENT`** — identifies the Comment's parent. Exactly
  one per Comment, written at creation and never re-targeted.
  The per-target catalog with row-level meanings lives in
  [edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging);
  this doc deliberately does not mirror that list (§1).
- **`Comment → Hashtag` (`:TAGGING`)** — one edge per hashtag
  the Comment is tagged with. See
  [edges.md §2 "Tagging"](../primitive/edges.md#tagging). The
  Hashtag node is content-addressed by canonical name (per
  [data-model.md "Node identity strategies"](../implementation/data-model.md#node-identity-strategies)),
  so the same hashtag across instances resolves to the same
  node.
- **`Comment → any node` (`:REFERENCES`)** — one edge per node
  the Comment embeds, quotes, or mentions: the original of a
  re-uploaded image on a parent Post, a User or Collective
  named in the body, a Proposal it cites in debate, etc. Two
  targets are excluded by the single-structural-edge invariant
  per [edges.md §2 "Reference"](../primitive/edges.md#reference):
  **Hashtag** (the `:TAGGING` edge already encodes the pair) and
  the Comment's own `:CONTAINMENT` parent (the `:CONTAINMENT`
  edge already encodes the pair — a Comment that quotes the very
  Post / Comment / Chat / ChatMessage / Item it is posted on does
  not write a parallel `:REFERENCES` edge). The carrier
  semantics, target catalog, and deferred traversal rules live in
  [edges.md §2 "Reference"](../primitive/edges.md#reference).

### As target (incoming)

A Comment receives:

- **Actor edges** from Users and Collectives carrying
  `(sentiment, relevance)` per
  [edges.md §1](../primitive/edges.md#1-actor-edges) — the
  like/dislike surface plus per-viewer relevance, used by
  [feed-ranking](../primitive/feed-ranking.md) to weight the
  Comment for each viewing user. The earliest of these is the
  authorship edge (§5).
- **`Comment → Comment` `:CONTAINMENT`** when another Comment
  replies to this one. A reply is itself a Comment whose
  target is the parent Comment; from the parent's perspective
  this is an incoming `:CONTAINMENT` edge. Reply chains
  accumulate `R` (path length) naturally and decay via `d(R)`
  in [feed-ranking](../primitive/feed-ranking.md) — there is
  no explicit depth cap.
- **`ChatMessage / Post / Comment → Comment` `:REFERENCES`**
  when another content node embeds the Comment — a chat
  message sharing it into a chat, a Post citing it, another
  Comment referencing it in debate. See
  [edges.md §2 "Reference"](../primitive/edges.md#reference).
- **`Proposal → Comment` `:TARGETS`** when a moderation
  Proposal targets one of the Comment's per-field
  moderation-status properties (§3). See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting);
  cascade in §6.

---

## 5. Authorship

A Comment's author is the actor whose incoming actor edge has
the earliest layer-1 timestamp — the same rule that derives
authorship for every node type
([authorship.md](../primitive/authorship.md)). On the graph that
edge carries the `:AUTHOR` sub-label; the author's `(dim1, dim2)`
on the same edge are normal opinion values (sentiment / relevance),
not a stand-in for the label. The two coexist.

`:AUTHOR` is the only representation of authorship on the graph
side. For Postgres-side display queries, `comments.author_id`
is cached on the row. Both are rebuildable from the graph; the
graph wins in any disagreement. See
[authorship.md "Caching"](../primitive/authorship.md#caching).

---

## 6. Lifecycle

Comment nodes are **never deleted**. Per
[layers.md §5](../primitive/layers.md#5-deletion-policy), the
only permitted "removal" is in-place layer redaction on graph
properties or a tombstone version row on Postgres-side display
content; both preserve a visible record that the change
occurred.

Redaction triggers on a Comment are moderation
([moderation.md §1](moderation.md#1-the-two-classification-paths))
and — with the author's opt-in — content-level account deletion.

Account deletion of the Comment's author does **not** by
default affect the Comment's body, attachments, or graph node —
identity redaction targets the User node's PII only. The
Comment is content-redacted only if the author opts in to the
content-level scope of
[account-deletion.md](account-deletion.md).

The Comment's UUID is stable across every redaction. Authorship
caches keyed on the UUID stay valid; the outgoing `:CONTAINMENT`,
`:TAGGING`, and `:REFERENCES` edges and every incoming actor /
reply / reference / targeting edge keep pointing at the same
node. A redacted Comment is a partially-or-fully gutted but
still-graph-resident content node, not a removed one.

---

## What this doc is not

- **Not the feed-ranking spec.** Where a Comment surfaces in
  any given viewing user's feed — including how reply chains
  accumulate `R`, how reactor-edge time decay attenuates stale
  threads, and the seen-list behavior for threads that gain
  fresh activity — lives in
  [feed-ranking.md](../primitive/feed-ranking.md).
- **Not the authorship rule.** The earliest-incoming-edge
  derivation, the cache rebuild semantics, and the worked
  example live in [authorship.md](../primitive/authorship.md).
- **Not the moderation primitive.** The Proposal mechanism, the
  mod gate, eligibility, thresholds, and the redaction cascade
  live in [moderation.md](moderation.md).
- **Not the deletion mechanism.** The redaction primitive lives
  in [layers.md §5](../primitive/layers.md#5-deletion-policy);
  the per-row legal hold and archive disposition live in
  [retention-archive.md](../primitive/retention-archive.md).
- **Not the edge catalog.** The full per-target containment
  list, all other edge types Comments participate in, and the
  label scheme live in [edges.md](../primitive/edges.md).
- **Not the Memgraph or Postgres schema.** Concrete property
  types, columns, indexes, and the
  `comment_attachments` / `media_attachments` shapes live in
  [graph-data-model.md](../implementation/graph-data-model.md)
  and [data-model.md](../implementation/data-model.md).
