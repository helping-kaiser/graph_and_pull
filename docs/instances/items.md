# Items

An **Item** is a content node representing a physical or digital good
— something that can be owned, transferred, and talked about. Items
are interactable content: they can be liked, disliked, commented on,
and tagged with hashtags.

Items are a **future** concern: the first iterations of CoGra focus
on posts and chats, and marketplace-like item flows will build on
this model once the base is running. The model below is committed
to regardless.

This doc covers two related nodes — the **Item** content node and
the **ItemOwnership** junction node — plus the convention for
shared ownership through a Collective.

---

## 1. Creation

An Item is created by a single compound gesture from one actor —
either a User or a Collective. Unlike a Post, Item creation is
**compound**: it brings the Item AND the author's first
ItemOwnership into existence in one atomic step, with the author
as the initial owner. There is no separate "list" then "claim
ownership" flow.

The gesture writes the following records atomically:

- A new `:Item` node on the graph.
- The Postgres `items` row carrying the name and description
  (see [data-model.md](../implementation/data-model.md)).
- `item_attachments` rows for each piece of attached media (zero
  or more).
- An actor edge from the author toward the Item — the
  **authorship edge** (§5). Its `(dim1, dim2)` values are the
  author's initial opinion of their own item, typically high
  positive sentiment and relevance.
- A new `:ItemOwnership` junction node for the author.
- The `ItemOwnership → User/Collective` `:BEARER` structural
  edge, binding the junction to the author.
- The author's `User/Collective → ItemOwnership` actor edge —
  their **Shape A self-claim** to the ownership.
- The `ItemOwnership → Item` claim edge.
- The `Item → ItemOwnership` approval edge with positive top
  layer (`dim1 > 0`).

With no prior owner to cast a Shape B vote, the
[two-edge approval pattern](../primitive/graph-model.md#5-junction-node-flows)
collapses to its 1-of-1 special case: the author's Shape A
self-claim is the only required vote, and the system writes
both structural edges atomically alongside it. Same bootstrap
shape as the founder's `CollectiveMember` in
[collectives.md "Creation"](collectives.md#1-creation) and the
founder of a Chat in
[chats.md §2.1](chats.md#21-chat). Subsequent transfers are
regular two-edge approvals (§6).

A Collective creating an Item is the same gesture: the graph
records the Item as the Collective's, and the off-graph
authentication that produced it traces — possibly through nested
CollectiveMember chains — back to one or more Users with active
sessions per
[user.md §1](../primitive/user.md#1-user-vs-collective) and
[auth.md](../implementation/auth.md). Whether member consent is
required is determined by the Collective's social-contract
treatment of content-acts per
[collectives.md "Acting through the Collective"](collectives.md#2-acting-through-the-collective).

---

## 2. Graph-side properties

### Item

An Item node carries only what the graph needs to traverse,
filter, and rank. Substance lives in Postgres (§3).

- **`moderation_status`** — `'normal'` / `'sensitive'` /
  `'illegal'`, default `'normal'`, layered. Universal across all
  user-input-bearing nodes; per-node mechanics — set by a passing
  `'sensitive'` Proposal, auto-flipped to `'illegal'` by the
  redaction cascade — are described in
  [nodes.md "Universal: moderation_status"](../primitive/nodes.md#universal-moderation_status)
  and §8 below.

The current owner is **not** stored as a property on the Item;
it is derived from the single ItemOwnership whose
`Item → ItemOwnership` approval edge has a positive top layer
(§7). Concrete property types and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

### ItemOwnership

ItemOwnership carries no per-instance properties beyond its
`id` — transfer state lives entirely in the surrounding edges
(claim, approval, and supersession layers per §§6-7). Bearer
identity rides on the `ItemOwnership → User/Collective`
`:BEARER` edge written at creation; see §1 and
[edges.md §2 "Bearer binding"](../primitive/edges.md#bearer-binding).
Concrete types and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 3. Postgres-side content

### Item

An Item's display content lives in Postgres, linked to the graph
node by UUID. Edits are append-only per
[layers.md §4](../primitive/layers.md#4-layers-on-postgres-side-display-content):
a new version row, no overwrite.

- **`name`** — required. The handle the Item is listed under.
  Stored on the `items` row.
- **`description`** — optional body text. Stored on the `items`
  row.
- **Attachments** — images and other media via the
  `item_attachments` junction table, which carries per-attachment
  `display_order` and an optional `is_cover` flag analogous to
  `post_attachments`. Each row references one `media_attachments`
  asset, owned by the same author as the Item (anti-hijack rule
  per
  [data-model.md "Why parents point at attachments"](../implementation/data-model.md#why-parents-point-at-attachments)).

Concrete schema lives in
[data-model.md](../implementation/data-model.md).

### ItemOwnership

None. ItemOwnership is a pure graph-side junction node — no
Postgres-side display content, no author-bearing row.

---

## 4. Edges

Dimension labels, sub-category labels, and traversal semantics
live in [edges.md](../primitive/edges.md).

### 4.1 Item

#### As source (outgoing)

An Item is not an actor and authors no actor edges. It carries
two outgoing structural edge types, both system-created:

- **`Item → ItemOwnership` (`:APPROVAL`)** — the approval side
  of the two-edge approval pattern. Created when the current
  owner's `(dim1 > 0)` actor edge toward a new `ItemOwnership`
  satisfies the approval policy (§6). **State transitions on
  this edge are the supersession mechanism described in §7**:
  when a subsequent transfer completes, the previous
  `ItemOwnership`'s `Item → ItemOwnership` top layer flips to
  `dim1 < 0` automatically. This Edges section catalogues only
  the edge type and direction; the layer mechanics live in §7.
  See
  [edges.md §2 "Approval completion"](../primitive/edges.md#approval-completion).
- **`Item → Hashtag` (`:TAGGING`)** — one edge per hashtag the
  Item is tagged with. See
  [edges.md §2 "Tagging"](../primitive/edges.md#tagging). The
  Hashtag node is content-addressed by canonical name (per
  [data-model.md "Node identity strategies"](../implementation/data-model.md#node-identity-strategies)),
  so the same hashtag across instances resolves to the same
  node.

#### As target (incoming)

An Item receives:

- **Actor edges** from Users and Collectives per
  [edges.md §1](../primitive/edges.md#1-actor-edges) — the
  like/dislike surface plus per-viewer relevance, used by
  [feed-ranking](../primitive/feed-ranking.md) to weight the
  Item for each viewing user. The earliest of these is the
  authorship edge (§5).
- **`Comment → Item` (`:CONTAINMENT`)** when a Comment is
  written on the Item. See
  [edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging).
- **`ItemOwnership → Item` (`:CLAIM`)** — the claim side of the
  two-edge approval pattern, paired with the outgoing
  `Item → ItemOwnership` above.
- **`ChatMessage / Post / Comment → Item` (`:REFERENCES`)** when
  another content node embeds the Item — a message sharing it
  into a chat, a Post recommending or citing it, a Comment
  pointing at it. See
  [edges.md §2 "Reference"](../primitive/edges.md#reference).
- **`Proposal → Item` (`:TARGETS`)** when a moderation Proposal
  targets a property on the Item — `'sensitive'` against
  `moderation_status`, or `'illegal'` against `name`,
  `description`, or `attachments` (§8). See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting).

### 4.2 ItemOwnership

#### As source (outgoing)

ItemOwnership is a junction, not an actor. It carries:

- **`ItemOwnership → Item` (`:CLAIM`)** — the claim side of the
  two-edge approval pattern, closed by the item's
  `Item → ItemOwnership` approval edge (§4.1) once the current
  owner casts their Shape B vote (§6). At Item creation the
  claim and approval are written in the same atomic gesture
  (§1 bootstrap). See
  [edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging).
- **`ItemOwnership → User/Collective` (`:BEARER`)** — identity-
  binding edge written at junction creation, pointing at the
  actor the ownership represents. Never re-pointed; the Shape A
  self-claim that activates the ownership must originate from
  this actor (§§1, 6). See
  [edges.md §2 "Bearer binding"](../primitive/edges.md#bearer-binding).
- **`ItemOwnership → ItemOwnership` (Shape B vote)** — the
  current owner's approval vote on a transfer to the new
  ItemOwnership for the same Item (§6). `dim1 > 0` approves the
  transfer. Once the transfer completes, the previous owner's
  ItemOwnership is no longer active (§7), so this edge type
  rarely carries further layers — but the primitive permits
  them (e.g. an ex-owner adding a stance update for audit). See
  [edges.md §2 "Voting (Shape B)"](../primitive/edges.md#voting-shape-b).

#### As target (incoming)

An ItemOwnership receives:

- **Actor edges** from Users and Collectives per
  [edges.md §1](../primitive/edges.md#1-actor-edges). For the
  acquirer themselves, the `User/Collective → ItemOwnership`
  edge is the **Shape A self-claim** that initiates the
  ownership (§6). For other actors, these edges are personal
  sentiment about the ownership record — they do not drive the
  approval vote, which uses Shape B (above).
- **`ItemOwnership → ItemOwnership` (Shape B vote)** — incoming
  approval vote from the current owner's existing ItemOwnership
  (§6).
- **`Item → ItemOwnership` (`:APPROVAL`)** — the approval side
  of the two-edge pattern, paired with the outgoing
  `ItemOwnership → Item` claim above. Supersession layers per
  §7 ride on this edge — see §4.1 for the carve-out.
- **`ChatMessage / Post / Comment → ItemOwnership`
  (`:REFERENCES`)** when a content node embeds an ownership
  record — e.g. a Post citing a provenance chain. See
  [edges.md §2 "Reference"](../primitive/edges.md#reference).

---

## 5. Authorship

An Item's author is the actor whose incoming actor edge has the
earliest layer-1 timestamp — the same earliest-incoming-edge rule
that derives authorship for every node type
([authorship.md](../primitive/authorship.md)). At creation, the
author's actor edge is written in the same compound gesture as the
Item node and the bootstrap ItemOwnership (§1); the author's edge
is the earliest incoming actor edge by construction.

**Authorship and ownership are distinct.** The author is the
**author** — the actor who minted, listed, or registered the
Item; this is immutable and derived from the earliest actor edge.
The **current owner** is whoever holds the active ItemOwnership
(§7) and changes with each transfer. An Item authored by one User
and currently owned by a different User or by a Collective is the
typical case after one or more transfers.

ItemOwnership is a junction node and has no authorship in the
[authorship.md](../primitive/authorship.md) sense — it represents
a transfer relationship, not an authored piece of content.

---

## 6. Transfer flow

ItemOwnership uses the **two-edge approval pattern** described in
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows):

1. The **acquirer** (User or Collective) writes a
   `User/Collective → new ItemOwnership` actor edge — their
   **Shape A self-claim** to the ownership. The acquirer has no
   ItemOwnership for this item yet, so the claim is necessarily
   Shape A. The system creates the `ItemOwnership → Item` claim
   edge in response.
2. The **current owner** casts a **Shape B vote** from their
   existing ItemOwnership for this item to the new ItemOwnership
   (`ItemOwnership_current → ItemOwnership_new`, `dim1 > 0`) —
   their approval of the transfer.
3. Approval policy is satisfied (single-approver: just the
   current owner); the system creates the
   `Item → ItemOwnership` approval edge.
4. The system also writes the supersession layer on the
   previous `Item → ItemOwnership_current` edge with
   `dim1 < 0`, marking the old ownership revoked (§7).
5. The new ItemOwnership is now the active one.

No one can take ownership without the current owner's Shape B
vote — there is no "take" operation in the graph. The Item-
creation bootstrap (§1) is the one exception. The Shape B vote
flows from the very ownership record that's about to be revoked
— fitting, since approving the transfer is the same act that
ends the voter's own ownership.

---

## 7. Supersession: exactly one active ItemOwnership per item

When a transfer completes and the new `Item → ItemOwnership`
approval edge is created, the system **automatically** adds a new
layer on the **previous** ItemOwnership's `Item → ItemOwnership`
approval edge with `dim1 < 0` — marking it revoked.
This uses the general state-transition mechanism on structural
edges described in
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows).

**Invariant — single active ownership:** At most one
ItemOwnership per Item has a positive top layer on its
`Item → ItemOwnership` approval edge at any time. Identifying the
current owner is therefore a single-edge query — "find the
ItemOwnership whose `Item → ItemOwnership` top layer has
`dim1 > 0`" — with no timestamp comparisons required.

Concurrent transfer attempts are prevented at the transaction
level rather than by a separate lock. Only the current owner can
cast the Shape B approval vote (§6); the same service-layer
transaction that writes that vote also writes the new
`Item → ItemOwnership` approval edge and the supersession layer
on the previous one. Once the transaction commits, the casting
ItemOwnership is no longer the active one and so can no longer
cast a second Shape B vote — the authority required to initiate
a transfer is consumed by the transfer it initiates. A
concurrent second transfer attempt by the same owner is
serialized behind the first by the transaction and fails the
current-owner check when it runs.

**Invariant — append-only ownership chain:** ItemOwnership nodes
and the layers on their approval edges are never deleted. The
old approval edge isn't removed on transfer, just superseded by
a newer layer that flips its state to revoked. Every past owner
of an Item remains visible on the graph as a revoked
ItemOwnership; only the active one changes.

---

## 8. Lifecycle

### Item

Item nodes are **never deleted**. Per
[layers.md §5](../primitive/layers.md#5-deletion-policy), the only
permitted "removal" is in-place layer redaction on graph
properties or a tombstone version row on Postgres-side display
content; both preserve a visible record that the change occurred.

Two redaction triggers apply to an Item today:

- **Moderation: `'sensitive'` classification.** A passing
  `'sensitive'` Proposal flips the top layer of `moderation_status`
  to `'sensitive'`. No redaction; display content stays. Each
  viewing user's `content_filtering_severity_level` (see
  [data-model.md](../implementation/data-model.md) "User
  preferences") decides how aggressively the frontend filters
  the Item. Reversible by a counter-Proposal back to `'normal'`.
  See [moderation.md §1](moderation.md#1-the-two-classification-paths).
- **Moderation: `'illegal'` classification.** A passing
  `'illegal'` Proposal targets one of the Item's user-input
  fields — `name`, `description`, `attachments` (every attached
  media), or the `'node'` sentinel covering all of the above
  per the per-node field list in
  [moderation.md §5](moderation.md#5-scope) —
  and fires the redaction cascade per
  [moderation.md §1](moderation.md#1-the-two-classification-paths):
  the affected Postgres rows are tombstoned with version markers,
  affected `media_attachments` rows are tombstoned and assets
  removed from object storage, the redacted originals are written
  to the [retention archive](../primitive/retention-archive.md)
  under per-row legal hold, and the Item node's `moderation_status`
  is auto-flipped to `'illegal'`. The cascade does **not**
  propagate to descendants — an Item classified illegal does not
  redact its Comments, any ChatMessage or Post that references
  it, or its ItemOwnership chain. Each requires its own
  classification.

**Account deletion of the Item's author** does not affect the
Item's name, description, attachments, or graph node. Identity-
level deletion redacts the User's PII; the User node's UUID is
stable and the Item's authorship edge keeps pointing at it.
Content-level deletion does **not** sweep up Items: Items are
goods, not first-person expression, and
[account-deletion.md §1](account-deletion.md#1-two-redaction-levels)
scopes content-level redaction to Posts, Comments, and
ChatMessages only.

**Account deletion of an owner** is the same shape: the User node
persists with redacted PII, the ItemOwnership chain UUIDs remain
valid, and ownership continues to resolve. If the deleted owner
is the current owner, the Item continues to be owned by that
(now-anonymous) User node. A subsequent transfer follows the
regular two-edge approval (§6) — the graph mechanics are
unchanged by PII redaction.

The Item's UUID is stable across every redaction. Authorship
caches, the ItemOwnership chain, comments, references, and
tagging edges all remain valid pointers.

### ItemOwnership

ItemOwnership nodes are also **never deleted**. A transfer
supersedes the previous ItemOwnership via the supersession layer
(§7); the old node and its edges remain in the graph as part of
the item's ownership history.

An item with **no** active ItemOwnership — no positive top layer
on any `Item → ItemOwnership` edge — is considered **abandoned**.
The history of all previous owners remains visible in the layer
stacks. Whether and how an abandoned item can be re-acquired is
a marketplace-layer concern not yet specified by the graph
model.

---

## 9. Shared ownership routes through a Collective

**Invariant — no parallel co-ownership:** An Item has at most one
active ItemOwnership at any time (§7); the graph does not support
parallel ItemOwnership junctions for the same Item by different
actors. Shared ownership must route through a Collective: the
Collective holds the single ItemOwnership, and internal sharing
is the Collective's social contract, not a graph-level mechanism.

A married couple co-owning a car, three roommates sharing a coffee
machine, a band co-owning equipment, a co-op holding tools — all
of these are modeled as: a Collective node, the sharing actors as
its CollectiveMembers (see [collectives.md](collectives.md)), the
Collective as the holder of the ItemOwnership. Internal disputes
are resolved by the Collective's own governance, not by
parallel-ItemOwnership voting on the graph.

---

## What this doc is not

- **Not the edge catalog.** Per-target-type edges with dimension
  labels live in [edges.md](../primitive/edges.md).
- **Not the moderation primitive.** The Proposal mechanism, the
  mod gate, eligibility, thresholds, and the redaction cascade
  live in [moderation.md](moderation.md).
- **Not the deletion mechanism.** The redaction primitive lives
  in [layers.md §5](../primitive/layers.md#5-deletion-policy);
  the per-row legal hold and archive disposition live in
  [retention-archive.md](../primitive/retention-archive.md);
  the account-deletion flow lives in
  [account-deletion.md](account-deletion.md).
- **Not the Memgraph or Postgres schema.** Concrete property
  types, columns, indexes, and the `item_attachments` /
  `media_attachments` shapes live in
  [graph-data-model.md](../implementation/graph-data-model.md)
  and [data-model.md](../implementation/data-model.md).
- **Not the marketplace UX or transaction shape.** Listing,
  pricing, escrow, transfer-confirmation UX, and any economic
  records that accrue around ownership transfers are future
  work. The graph model committed to above is the substrate they
  will build on.
