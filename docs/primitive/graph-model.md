# Graph Model

How edges, nodes, and their properties work in the Peer Network graph.
This is the foundation that the [feed ranking algorithm](feed-ranking.md)
operates on.

---

## 1. Core Principles

Every edge in the graph is:
- **Directional** — `A -> B` and `B -> A` are separate edges. A friendship is
  two edges. A follow with no follow-back is one edge.
- **Multi-dimensional** — every edge carries exactly **2 dimensions** plus
  **system dimensions**. The meaning of those 2 dimensions depends on the edge
  type (see [edges.md](edges.md) for the full catalog).
- **Append-only** — interactions add layers; they never overwrite. You cannot
  hide that you disliked someone in the past. Your current feelings are the
  top layer, but the full history is preserved. Append-only is a project-wide
  rule that extends beyond edges to node properties and Postgres-side display
  content — see [layers.md](layers.md) for the general principle.

And the graph itself is:
- **Fully transparent** — every node and every edge in the graph is visible
  to every actor on it. The only way to be invisible is to not be on the
  graph (a disconnected, self-hosted instance is possible but unreachable
  from anywhere else). Privacy of *content* is achieved through end-to-end
  encryption; topology itself is always public.

---

## 2. Node Categories

Nodes fall into three categories:

- **Actor nodes** — entities that take actions and create edges
  (User, Collective).
- **Content nodes** — entities that are acted upon (Post, Comment,
  Chat, ChatMessage, Item, Hashtag).
- **Junction nodes** — entities that represent relationships which
  themselves can be interacted with (ChatMember, CollectiveMember,
  ItemOwnership). They have roles, need approval flows, and
  eliminate parallel edges between the same two nodes: a user's
  *membership* in a chat and their *opinion* of that chat are
  edges to different nodes.

**Junction nodes carry typed properties** (role, `ownership_pct`,
etc.) as properties on the junction node itself, not encoded in
edge dimensions. Categorical data belongs in categorical fields;
quantities need more range than the bipolar `[-1, +1]` edge
dimensions provide.

See [nodes.md](nodes.md) for the full catalog — what each node
type is, its graph-side properties, and where its display content
lives in Postgres.

---

## 3. Edge Categories

There are two categories of edges. Both use the same tensor shape (2
dimensions + system dimensions) to keep graph calculations uniform — the
algorithm never needs to branch on edge category.

### Actor edges

Created by actor nodes (User, Collective) toward any other node. Express
**opinion and interaction**. The 2 dimensions carry subjective meaning
(sentiment, relevance, closeness — varies by edge type, see [edges.md](edges.md)).

### Structural edges

Express **containment or belonging** between nodes. Created by the system,
not by actors. By default the 2 dimensions are `(0.0, 0.0)` — neutral
structural links.

Why give structural edges the same shape instead of making them different:
- The ranking algorithm traverses paths that cross both edge types (e.g.
  `User -> User -> Comment -> Post`). Uniform shape means no branching logic
  at each hop.
- Structural edges can carry meaningful weight where the shape calls for
  it. The concrete case today is state-bearing approval-pattern edges on
  junction nodes — see §5 for how revocation and state transitions are
  encoded in structural edge layers. A pinned comment's `Comment -> Post`
  weight could work similarly.

### Structural edge pairs

Structural edges are **not paired for query convenience**. Memgraph (and
openCypher generally) indexes relationships at both endpoints, so a single
one-directional edge is traversable in either direction with equal
efficiency. Adding a reverse edge just so a query reads more naturally
would double storage for no gain.

Structural edge pairs **are valid when each direction encodes a distinct
fact**. The canonical example is approval-required junctions:

- `ChatMember -> Chat` — "this membership claims to be about this chat"
  (exists from the moment the request is made).
- `Chat -> ChatMember` — "this chat has accepted this member" (only exists
  after the approval policy is satisfied).

These are two different facts, so two edges is correct. In contrast,
`Comment -> Post` does not need a `Post -> Comment` companion: the reverse
would carry the same fact and just duplicate storage. See §5 for the full
junction approval pattern.

---

## 4. Edge Structure

Every edge, regardless of category, has the same shape:

```
Edge {
    // --- 2 dimensions (meaning varies by edge type) ---
    dimension_1: f64,   // actor edges: e.g. sentiment, range [-1.0, +1.0]
                        // structural edges: 0.0
    dimension_2: f64,   // actor edges: e.g. relevance, range [-1.0, +1.0]
                        // structural edges: 0.0

    // --- System dimensions (same for all edge types) ---
    timestamp:   DateTime,  // when this layer was created
    layer:       u32,       // which layer this is (1 = first interaction)
}
```

**Range is uniform.** Both dimensions are `f64` in `[-1.0, +1.0]` for every
actor edge, regardless of what the dimension represents. Uniformity is a
first-class design goal: the ranking algorithm never branches on dimension
type, and the math stays consistent across every edge in the graph. See
§6 for how negative values are interpreted when a dimension wouldn't
obviously have a negative meaning.

An edge between two nodes is a **stack of layers**. Each interaction appends a
new layer. The "current" state of the edge is the top layer. The full history
is always available.

---

## 5. Junction Node Flows

Junction nodes enable approval-required relationships and role management
without parallel edges. All three junction types — ChatMember,
CollectiveMember, ItemOwnership — share a common shape.

Junction approval is one application of CoGra's broader governance
primitive (weighted role-based voting) — see
[governance.md](governance.md) for the full shape (five components,
two vote shapes, weight-at-tally-time rule) that junction approval,
message moderation, and future voting patterns all share. This
section focuses on how the primitive specifically applies to
junction relationships.

### The two-edge approval pattern

A junction relationship is created in two steps:

1. **Claim edge** — when the relationship is initiated, the system creates
   a structural edge from the junction node toward its parent (e.g.
   `ChatMember -> Chat`). The claim exists as long as the junction node
   exists.
2. **Approval edge** — when the relationship's approval policy is
   satisfied (all required actor edges toward the junction node exist), the
   system creates the reverse structural edge (e.g. `Chat -> ChatMember`).
   The presence of this edge marks the relationship as *active*.

**State is encoded in the graph topology itself** — no status flag is
needed:

- Only the claim edge exists → pending.
- Both edges exist → active.

The **approval policy** for each relationship is "N actor edges from
specific roles required toward the junction node" — an instance of
the threshold policy described in
[governance.md §2.4](governance.md). Open chats have N = 1 (the
joining user); invite-only and request-entry chats have N = 2 (user +
admin, in either order); governance-heavy joins can require larger N
with weighted multi-sig (weights derived from role properties on the
approving actors' own junction nodes, per
[governance.md §2.3](governance.md)).

### Revocation and state transitions

Because edges are append-only, the approval edge created when a
junction relationship becomes active cannot be removed. "No longer
active" is therefore encoded as **new layers on the two structural
edges of the approval pair** — not by deletion.

**Dimension semantics on state-bearing structural edges.** For the
claim and approval edges of a junction approval pair, `dim1` carries
the edge's current stance; `dim2` is reserved (default `0`, available
for future use such as reason codes):

- `dim1 > 0` — **affirmed** (claim stands / parent accepts).
- `dim1 = 0` — **abstained / neutral**.
- `dim1 < 0` — **revoked / withdrawn**.

Layer 1 of each edge is created by the system with `(+1, 0)` — the
edge is created because someone affirmed it. Revocation adds a new
layer with `dim1 < 0`. Re-affirming after a revocation adds another
new layer with `dim1 > 0`. Append-only is preserved; the full state
history is visible in the layer stacks.

**Relationship state is derived from both top layers.** A junction
relationship is **active** iff both paired edges' top layers have
`dim1 > 0`. Any negative top layer on either edge makes the
relationship inactive. The pending state (claim edge only, no
approval edge yet) still applies — it's a separate case from revoked.

**Who triggers state transitions.** The system is still the only
entity that writes to structural edges. It reacts to actor-edge
changes:

- **Voluntary leave / withdrawal.** Actor adds a new negative-sentiment
  layer on their actor edge toward the junction node. System adds a
  new layer on the **claim-side** structural edge with `dim1 < 0`.
  Self-determined; not a governance decision.
- **Removal via governance instance (kick / fire / expel).** Removal
  is the outcome of a governance instance defined on the parent
  node — same primitive as approval, with its own eligibility,
  weights, and threshold (see
  [governance.md](governance.md)). Configurations span the full
  range:
    - Single-approver instance (1-of-1 from the original approver) —
      retains the simple "the approver who let you in can revoke"
      shape where it's appropriate.
    - Multi-sig instance (N-of-M from named roles) — the inverse of
      a multi-sig approval.
    - Community-vote instance (Shape B disavowal with quorum and
      threshold) — used by ChatMember per
      [chats.md §6](../instances/chats.md) and configurable per
      [collectives.md](../instances/collectives.md).
  When the instance's threshold is crossed, the system adds a new
  layer on the **approval-side** structural edge with `dim1 < 0`.
- **System-initiated** (auto-expiry, violation handling, etc.). System
  adds the appropriate negative layer directly.

**Intermediate states are not materialized.** For multi-vote
governance instances, the approval-side structural edge stays at
its top layer until the instance's threshold is crossed. Partial
progress is visible on individual vote edges; the structural edge
reflects only the threshold-resolved state.

**Cascading updates across structural edges.** A state change on one
structural edge can trigger the system to add a corresponding layer on
another structural edge when consistency requires it. This is a
general mechanism — the canonical case today is ItemOwnership
supersession (see [items.md](../instances/items.md)), where creating a new approval
edge causes the previous one to be marked revoked so that exactly one
ownership is active at a time. Future junction or content patterns
may use the same cascade shape.

### Chat Membership (ChatMember)

Chat-specific flows (open / invite-only / request-entry) are explained
in [docs/chats.md](../instances/chats.md). They are all variants of the two-edge
approval pattern described above.

### Collective Membership (CollectiveMember)

Collective-specific flows are explained in [docs/collectives.md](../instances/collectives.md).
They follow the same two-edge approval pattern described above.

### Ownership Transfer (ItemOwnership)

Item-specific flows are explained in [docs/items.md](../instances/items.md). They
follow the same two-edge approval pattern described above, with the
additional property that transfers form an append-only chain of
ItemOwnership nodes per item.

---

## 6. Dimension Semantics

### Why the dimensions differ per edge type

The same numeric value means different things in different contexts:
- **User -> User**: dimension_2 = `+0.9` means "we interact constantly, very
  close." This is **closeness**.
- **User -> Post**: dimension_2 = `+0.9` means "this is extremely relevant /
  fascinating to me." This is **relevance**.

But because both are `f64` in `[-1.0, +1.0]`, the ranking algorithm can
compute over them uniformly. The *interpretation* differs; the *math* doesn't.

### Range and polarity

Every actor-edge dimension is bipolar in `[-1.0, +1.0]`:

- `0.0` = no opinion / no interaction / neutral.
- Positive = the "forward" meaning (like, approve, close, want, relevant).
- Negative = the **active opposite**, not merely the absence.

The polarity matters most where the forward meaning sounds like a one-sided
scale — most notably **closeness**. A closeness of `0.0` means "we don't
interact"; a negative closeness means "I am actively avoiding this person"
(muted, blocked, ghosted). The two are distinct signals, and collapsing
negative closeness into `0.0` would discard real information. The same
reading extends to relevance (negative = "I actively don't want this in my
feed") and to approval dimensions on junction nodes (negative = active
rejection, not abstention).

Holding the full `[-1.0, +1.0]` range for every dimension also keeps the
ranking math uniform and avoids per-dimension clamping or branching logic.

### Independence of dimensions

The two dimensions are independent. Examples:

- **High sentiment, low relevance**: I'm glad a foreign dictator was removed
  from power (+0.75 sentiment), but I have no ties to that country and I'm not
  into politics (-0.5 relevance).
- **Low sentiment, high relevance**: I don't have strong feelings about a new
  tax law (0.0 sentiment), but it directly affects my business (+0.9
  relevance).
- **User -> User**: I love a celebrity's work (+0.8 sentiment) but we've never
  interacted and they don't know I exist (-0.8 closeness).

---

## 7. Directionality: Inbound Edges Don't Affect Your Graph

This is a critical design decision for anti-spam and anti-manipulation:

**Edges created toward you by others do not change your feed.**

If a cluster of bots likes Jakob's posts 10,000 times:
- The bots now have strong edges toward Jakob — so Jakob appears high in
  *their* feeds.
- Jakob has zero edges toward the bots — they don't appear in *his* feed at
  all.
- The bot cluster gains nothing economically because the economically
  important nodes (real users and advertisers) never point toward them.

This is only possible because all edges are directional. There is no concept
of an undirected "connection." A friendship is explicitly:
```
Jakob -[sentiment: +0.8, closeness: +0.9]-> Alice
Alice -[sentiment: +0.7, closeness: +0.9]-> Jakob
```

Two independent edges. Removing one does not remove the other.

---

## 8. Append-Only History (edges)

This section covers the edge-specific shape of append-only history.
For the project-wide principle — including node properties and
Postgres-side display content — see [layers.md](layers.md).

Each edge is not a single value but a stack of layers:

```
Jakob -> Post_X:
  Layer 1 (2025-01-15): sentiment: +0.3, relevance: +0.1   # mild like
  Layer 2 (2025-06-20): sentiment: +0.8, relevance: +0.6   # revisited, loved it
  Layer 3 (2026-02-01): sentiment: +0.2, relevance: -0.3   # feelings faded
```

**Rules:**
- New interactions always append a new layer.
- No layer can be deleted or modified after creation.
- The "current" edge state = the most recent layer.
- The full history is available for algorithms that need it (e.g., detecting
  opinion shifts, weighting by interaction frequency).

**Layer count as a signal:** The number of layers on an edge is itself
meaningful. An edge with 50 layers represents a deep, frequently-revisited
relationship. An edge with 1 layer is a passing interaction. How exactly
this signal factors into ranking is an open question — see
[open-questions.md Q1](../open-questions.md).

---

## 9. Relationship to feed ranking

The [feed ranking algorithm](feed-ranking.md) is a general rule for
ordering target nodes in any signed, weighted graph from a root node's
perspective. It is deliberately layer-agnostic — the math applies
regardless of what the signs and weights represent.

This document defines the **concrete inputs** the ranking algorithm
operates on in CoGra:

- Node categories (actor, content, junction) — §2.
- Edge categories (actor, structural) — §3.
- The uniform 2-dimensional `[-1.0, +1.0]` tensor shape — §4.
- Directional semantics — §7 (inbound edges don't affect the viewer's
  feed).
- Append-only layer stacks — §8 (the current state is the top layer).

How the continuous tensor values map into the ranker's signed-edge
math (sign + weight, product, per-dimension contribution, or
something else) is not yet pinned down. See
[open-questions.md Q2](../open-questions.md) for the full shape of this
question and the options considered.
