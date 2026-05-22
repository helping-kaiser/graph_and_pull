# Hashtag

The **Hashtag** is a content node — a topic tag that other
content nodes attach to. Among content nodes, Hashtag is the
odd one out: its identity is **content-addressed**. The UUID
is `UUIDv5(HASHTAG_NAMESPACE, canonical_name)` with a fixed
project-scoped namespace, derived from the canonical name
rather than minted at random. That choice cascades through
everything else this doc describes — creation is implicit on
first use, idempotent across actors, and federation across
instances requires no reconciliation.

This doc is the per-node catalog for the Hashtag: how it is
created, what it carries on the graph and (deliberately) not
in Postgres, what edges it can participate in, and how it
ends. The mechanics those topics depend on stay in their
topical docs — this doc links rather than duplicates.

---

## 1. Creation

A Hashtag is **brought into existence implicitly by the first
edge that needs it**. No actor authors it, and no explicit
"create hashtag" gesture exists. When a Post, Comment, or Item
is created with a tag string in its body, the API:

1. Normalizes the tag string — currently lowercase, no `#`.
2. Computes the UUID via
   `UUIDv5(HASHTAG_NAMESPACE, canonical_name)`.
3. Upserts the `:Hashtag` node (the graph node, the registry
   row in Postgres — see §3 — and the `name` graph property
   are written together, idempotently).
4. Writes the `Post → Hashtag`, `Comment → Hashtag`, or
   `Item → Hashtag` `:TAGGING` edge.

Because the UUID is a pure function of the canonical name,
step 3 is **idempotent**: a second post tagging the same
name in the same instance, or in any other instance running
the same namespace UUID, computes the same UUID and lands on
the same node. The "first" use is only first relative to a
given instance's storage — semantically, every reference to
`bot-defense` is to *the* `bot-defense` Hashtag, by
construction.

The canonical-name normalization and the namespace UUID are
**load-bearing schema**, not UI affordances — changing
either invalidates every previously-derived Hashtag UUID.
The full mechanism, the namespace-fixity rule, and the
federation implication live in
[data-model.md "Node identity strategies"](../implementation/data-model.md#node-identity-strategies).

### Federation implication

The content-addressed UUID is what makes Hashtag the only
node type for which federation across separated instances is
free of reconciliation. Two instances that have never
communicated, each holding a `:Hashtag` with `name =
"bot-defense"`, hold *the same node* — the UUIDs are equal
by construction. When their graphs are later joined,
federated, or synchronized, no merge layer or alias table is
needed for hashtags. Every other node type — User,
Collective, Post, Comment, etc. — requires a federation
protocol to decide whether two same-named or independently-
created records refer to the same identity (per
[data-model.md](../implementation/data-model.md#node-identity-strategies),
deferred to [open-questions.md](../open-questions.md) Q15).
Hashtag side-steps that question entirely.

---

## 2. Graph-side properties

A Hashtag node carries the minimum the graph needs to
traverse, filter, and rank. There is no Postgres-side
display content (§3).

- **`name`** — the canonical tag string (lowercase, no
  `#`). Immutable except via the redaction cascade (§5).
  The tag *is* the identifier in the everyday sense, but
  the graph key is still the UUID; `name` is mathematically
  redundant with the UUID by the content-addressing rule
  (§1) yet is stored explicitly so the graph can render the
  tag without a Postgres lookup and so name-redaction (§5)
  has a field to act on.
- **`moderation_status`** — `'normal'` / `'sensitive'` /
  `'illegal'`, default `'normal'`, layered. Universal
  across all user-input-bearing nodes; the per-node
  mechanics — set by a passing `'sensitive'` Proposal,
  auto-flipped to `'illegal'` by the redaction cascade —
  are described in
  [nodes.md "Universal: moderation_status"](../primitive/nodes.md#universal-moderation_status)
  and §5 below.

Concrete property types and indexes for these graph-side
properties live in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 3. Postgres-side content

Hashtag has **no Postgres-side display content** — no body,
no description, no media, no profile material. The reason
flows from §1: a Hashtag is a topic identifier, not authored
content. There is nothing to display beyond the tag string
itself, which already lives on the graph.

A small `hashtags` registry row does exist in Postgres
(`id`, `name`, `created_at`) per
[data-model.md](../implementation/data-model.md), but it is
a name-lookup and enumeration aid (autocomplete, alphabetical
indexing) rather than display content. The `id` column has
no `DEFAULT` — the API must always supply the deterministic
UUIDv5; falling back to a random UUID would silently break
content-addressing for any row that hit the fallback. This
is the only Postgres table where the rule "the API always
supplies the UUID" is enforced by *removing* the default
rather than just by convention.

---

## 4. Edges

### As source (outgoing)

A Hashtag authors no edges and originates no structural
edges either — it is a pure target. There are no outgoing
edges of any kind. (Among content nodes, this is also the
distinguishing shape vis-à-vis Post and Comment, both of
which originate at least one outgoing structural edge.)

### As target (incoming)

**Cosmetic-only — no actor edge to Hashtag.** A Hashtag
receives no actor edges from anyone. The catalog has no
`User → Hashtag` or `Collective → Hashtag` row
([edges.md §1](../primitive/edges.md#1-actor-edges)), and the
gesture of "liking a hashtag" simply doesn't exist as a graph
operation. Hashtags are reachable via `:TAGGING` for
discovery queries — the surface filters tagged content, ranks
that content, and presents the result — but the Hashtag
itself is never a ranking endpoint or a path participant.
Combined with the no-outgoing-edges rule (§4 "As source"),
any path that reaches a Hashtag terminates there without
contributing further: there is no edge to continue the path
through.

**Invariant:** Hashtags do not participate in feed-ranking
path products. `:TAGGING` is pure topology used for discovery
filters, never traversed by the ranking math. The
`ChatMessage → Hashtag` `:REFERENCES` edge below is on the
same footing — a topology record that a message body
mentioned the tag, mathematically inert because the path
terminates at the Hashtag in exactly the same way.

The structural edges that do land at a Hashtag:

- **`Post → Hashtag` `:TAGGING`** when a Post is tagged
  with this hashtag.
- **`Comment → Hashtag` `:TAGGING`** when a Comment is
  tagged with this hashtag.
- **`Item → Hashtag` `:TAGGING`** when an Item is tagged
  with this hashtag. See
  [edges.md §2 "Tagging"](../primitive/edges.md#tagging)
  for the full source catalog.
- **`ChatMessage → Hashtag` `:REFERENCES`** when a chat
  message embeds the hashtag (e.g. surfacing a topic feed
  into a chat). ChatMessage is the only carrier for this
  edge: Post and Comment reach Hashtag via `:TAGGING`, and a
  single structural edge per (source, target) pair is the
  rule — see the Hashtag carve-out in
  [edges.md §2 "Reference"](../primitive/edges.md#reference).
  Mathematically inert per the invariant above; recorded for
  topology completeness only.
- **`Proposal → Hashtag` `:TARGETS`** when a moderation
  Proposal targets a property on the Hashtag —
  `'sensitive'` against `moderation_status`, or
  `'illegal'` against `name`. See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting)
  and §5.

---

## 5. Lifecycle

Hashtag nodes are **never deleted**. Per
[layers.md §5](../primitive/layers.md#5-deletion-policy),
the only permitted "removal" is in-place layer redaction on
graph properties; both preserve a visible record that the
change occurred.

There is no authorship section for the Hashtag — by §1 a
hashtag has no author. The "earliest incoming layer-1 edge"
rule from
[authorship.md](../primitive/authorship.md) does not
meaningfully apply: the first edge a Hashtag receives is
typically a `:TAGGING` edge from whichever Post, Comment,
or Item happened to be created first, but that actor is the
author of the *tagging node*, not of the topic. Hashtags
are registry concepts, not authored content; account
deletion of any one contributor has no effect on the
Hashtag itself.

Two redaction triggers apply to a Hashtag today, both via
moderation:

- **Moderation: `'sensitive'` classification.** A passing
  `'sensitive'` Proposal flips the top layer of
  `moderation_status` to `'sensitive'`. No redaction on
  `name`. Reversible by a counter-Proposal back to
  `'normal'`. See
  [moderation.md §1](moderation.md#1-the-two-classification-paths).

  The classification is a **passive filter on incidental
  exposure**, not a block on intentional retrieval. A viewer
  with their `content_filtering_severity_level` (see
  [data-model.md](../implementation/data-model.md) "User
  preferences") set to filter sensitive content sees **no
  presence** of the hashtag on nodes that tag or reference it —
  the frontend either drops the chip entirely or renders a
  neutral placeholder (frontend choice; the primitive doesn't
  specify which). The viewer's incidental exposure surfaces stay
  clean.

  **Direct retrieval is unaffected.** A viewer who types the
  exact hashtag name into a search box still resolves to the
  Hashtag node and can follow `:TAGGING` /
  `:REFERENCES` edges from there to the full set of tagged
  nodes. The sensitive flag does not block intentional lookup —
  it only suppresses the chip on surfaces the viewer didn't ask
  to see. This is the same logic as `'sensitive'` on any other
  node: the content stays, the incidental surface filters.
- **Moderation: `'illegal'` classification.** A passing
  `'illegal'` Proposal targets `name` (the only user-input
  field on the Hashtag) and fires the redaction cascade per
  [moderation.md §1](moderation.md#1-the-two-classification-paths):
  the top layer of `name` is replaced with a redaction
  marker, the corresponding `hashtags.name` registry row
  is tombstoned, and `moderation_status` is auto-flipped to
  `'illegal'`. The cascade does **not** propagate across
  `:TAGGING` edges in either direction — a Hashtag
  classified illegal does not redact the Posts and Items
  that tag it, and vice versa; each node requires its own
  classification.

**Invariant:** `:Hashtag.name` is immutable except via the
redaction cascade. No property-amendment Proposal targeting
`name` is admissible — the only mechanism that can write a
new layer on `name` is the `'illegal'` moderation cascade
above, which replaces the top layer with a redaction marker.
The `'sensitive'` classification path doesn't touch `name`
at all (it flips `moderation_status` only). Both moderation
paths leave the UUID untouched.

The Hashtag's UUID is stable across redaction. Crucially,
because the UUID was derived from the *original* canonical
name, a future post that tags the same name will compute
the same UUID and resolve to the same — now-redacted —
node. Content-addressed identity holds even after the
public name is gone; the UUID is permanently bound to the
original string by construction. Every incoming actor /
`:TAGGING` / `:REFERENCES` / `:TARGETS` edge keeps pointing
at the same node. A redacted Hashtag is a graph-resident
content node with its `name` field gutted, not a removed
one.

---

## What this doc is not

- **Not the node identity primitive.** The three identity
  strategies (canonical-string / chosen-handle /
  per-creation), the UUIDv5 mechanism, the namespace-fixity
  rule, and the federation implications across all node
  types live in
  [data-model.md "Node identity strategies"](../implementation/data-model.md#node-identity-strategies).
- **Not the feed-ranking spec.** How a hashtag-driven
  discovery surface is composed — which posts, in what
  order, with what decay — lives in
  [feed-ranking.md](../primitive/feed-ranking.md). The
  bot-defense hashtag worked example in
  [feed-ranking.md §3.8.3](../primitive/feed-ranking.md#383-community-bot-defense-posts--supplementary-evidence)
  is a usage convention layered on top of the primitive
  this doc describes, not part of the primitive itself.
- **Not the moderation primitive.** The Proposal mechanism,
  the mod gate, eligibility, thresholds, and the redaction
  cascade live in [moderation.md](moderation.md).
- **Not the deletion mechanism.** The redaction primitive
  lives in
  [layers.md §5](../primitive/layers.md#5-deletion-policy);
  the per-row legal hold and archive disposition live in
  [retention-archive.md](../primitive/retention-archive.md).
- **Not the edge catalog.** The full set of edges
  Hashtag participates in, with row-level meanings and
  label assignments, lives in
  [edges.md](../primitive/edges.md).
- **Not the Memgraph or Postgres schema.** Concrete
  property types, columns, indexes, and the registry-row
  shape live in
  [graph-data-model.md](../implementation/graph-data-model.md)
  and [data-model.md](../implementation/data-model.md).
