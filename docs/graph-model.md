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
  type (see section 5).
- **Append-only** — interactions add layers; they never overwrite. You cannot
  hide that you disliked someone in the past. Your current feelings are the
  top layer, but the full history is preserved.

And the graph itself is:
- **Fully transparent** — every node and every edge in the graph is visible
  to every actor on it. The only way to be invisible is to not be on the
  graph (a disconnected, self-hosted instance is possible but unreachable
  from anywhere else). Privacy of *content* is achieved through end-to-end
  encryption; topology itself is always public.

---

## 2. Node Types

Nodes are either **actor nodes** (entities that take actions and create edges),
**content nodes** (entities that are acted upon), or **junction nodes**
(entities that represent relationships which themselves can be interacted with).

### Actor nodes

| Node type | Description |
|-----------|-------------|
| **User** | A person on the platform. |
| **Company** | A business, organization, band, solo artist profile — any collective or professional entity. Can author content, be followed, post items. Central to the economic model — companies pay for ads and receive ad revenue. |

### Content nodes

| Node type | Description |
|-----------|-------------|
| **Post** | Content authored by a user or company (text, image, video). |
| **Comment** | A response to a post or another comment. Is a full node because comments can be liked, disliked, and replied to. |
| **Chat** | A conversation container (group or 1:1). |
| **ChatMessage** | A single message within a chat. |
| **Item** | A physical or digital good (future). |
| **Hashtag** | A topic tag. Also covers concepts like places (e.g. `#berlin`) — if places ever need dedicated properties they can become their own node type later. |

### Junction nodes

Junction nodes represent relationships that have **roles**, need **approval
flows** (multi-sig), and can themselves be **interacted with** (liked,
voted on, etc.). They follow the same pattern as ChatMessage (which is a
junction between a Chat and the content within it).

| Node type | Connects | Why it's a node |
|-----------|----------|-----------------|
| **ChatMember** | Chat <-> User/Company | Has roles (admin, mod, member). Entry can require multi-sig approval (invite-only chats). Can be interacted with (vote to kick, promote to admin). |
| **CompanyMember** | Company <-> User/Company | Has roles (founder, shareholder, worker, band member, subsidiary). Multi-sig for adding/removing members. Ownership stakes. Companies can be members of other companies (holdings, subsidiaries, label rosters). |
| **ItemOwnership** | Item <-> User/Company | Represents ownership claim. Multi-sig for transfer (acquirer requests, current owner approves). Full ownership history. |

Junction nodes eliminate the need for parallel edges between the same two
nodes. A user's **membership** in a chat and their **opinion** of that chat
are edges to different nodes:

```
Jakob -[actor edge]-> ChatMember_Jakob_Chat1 -[structural]-> Chat1   (membership)
Jakob -[actor edge]-> Chat1                                          (opinion)
```

**Junction nodes carry typed properties.** Role (`admin`, `mod`, `member`,
`founder`, `shareholder`, `worker`, etc.) and any role-attached quantities
(e.g. `ownership_pct` on a shareholder CompanyMember) are stored as
properties on the junction node itself, not encoded in edge dimensions.
Categorical data belongs in categorical fields; quantities need more range
and resolution than the bipolar `[-1, +1]` edge dimensions provide.
Multi-sig weighting for approvals is then derived from these role
properties when actor edges toward the junction are evaluated.

---

## 3. Edge Categories

There are two categories of edges. Both use the same tensor shape (2
dimensions + system dimensions) to keep graph calculations uniform — the
algorithm never needs to branch on edge category.

### Actor edges

Created by actor nodes (User, Company) toward any other node. Express
**opinion and interaction**. The 2 dimensions carry subjective meaning
(sentiment, relevance, closeness — varies by edge type, see section 5).

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
  junction nodes — see §6 for how revocation and state transitions are
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
would carry the same fact and just duplicate storage. See §6 for the full
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
§7 for how negative values are interpreted when a dimension wouldn't
obviously have a negative meaning.

An edge between two nodes is a **stack of layers**. Each interaction appends a
new layer. The "current" state of the edge is the top layer. The full history
is always available.

---

## 5. Complete Edge Catalog

### Actor edges

All actor edges are created by User or Company nodes toward other nodes. The
2 dimensions are set by the actor.

**User as actor:**

| Edge type | Dimension 1 | Dimension 2 |
|-----------|-------------|-------------|
| User -> User | **Sentiment** (love to hate) | **Closeness** (how much we interact / know each other) |
| User -> Company | **Sentiment** (love to hate) | **Closeness** (how much I engage with this brand) |
| User -> Post | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User -> Comment | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User -> Chat | **Sentiment** (like to dislike) | **Relevance** (how important is this chat to me) |
| User -> ChatMessage | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User -> ChatMember | **Sentiment** (approve to reject) | **Relevance** (how important is this membership to me) |
| User -> CompanyMember | **Sentiment** (approve to reject) | **Relevance** (how important is this membership to me) |
| User -> ItemOwnership | **Sentiment** (approve to reject) | **Relevance** (how important is this transfer to me) |
| User -> Item | **Sentiment** (want to avoid) | **Relevance** (how interesting to me) |
| User -> Hashtag | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |

**Company as actor:**

| Edge type | Dimension 1 | Dimension 2 |
|-----------|-------------|-------------|
| Company -> User | **Sentiment** | **Relevance** (how valuable is this user to the company) |
| Company -> Company | **Sentiment** | **Relevance** |
| Company -> Post | **Sentiment** | **Relevance** |
| Company -> Comment | **Sentiment** | **Relevance** |
| Company -> Chat | **Sentiment** | **Relevance** |
| Company -> ChatMessage | **Sentiment** | **Relevance** |
| Company -> ChatMember | **Sentiment** (approve to reject) | **Relevance** |
| Company -> CompanyMember | **Sentiment** (approve to reject) | **Relevance** |
| Company -> ItemOwnership | **Sentiment** (approve to reject) | **Relevance** |
| Company -> Item | **Sentiment** | **Relevance** (how important is this product) |
| Company -> Hashtag | **Sentiment** | **Relevance** |

### Structural edges

Structural edges are system-created. Dimensions are `(0.0, 0.0)`.

**Containment / belonging:**

| Edge type | Meaning |
|-----------|---------|
| Comment -> Post | This comment is on this post |
| Comment -> Comment | This comment is a reply to that comment |
| Comment -> Chat | This comment is on this chat as a whole |
| Comment -> ChatMessage | This comment is on this specific message |
| Comment -> Item | This comment is on this item |
| ChatMessage -> Chat | This message belongs to this chat |
| ChatMember -> Chat | This membership claims to be about this chat (claim) |
| CompanyMember -> Company | This membership claims to be about this company (claim) |
| ItemOwnership -> Item | This ownership claim relates to this item (claim) |

**Approval completion** (paired with the claim edges above — see §6):

| Edge type | Meaning |
|-----------|---------|
| Chat -> ChatMember | This chat has accepted this member |
| Company -> CompanyMember | This company has accepted this member |
| Item -> ItemOwnership | This item's ownership transfer to this claim is complete |

**Tagging:**

| Edge type | Meaning |
|-----------|---------|
| Post -> Hashtag | This post is tagged with this hashtag |
| Item -> Hashtag | This item is tagged with this hashtag |

---

## 6. Junction Node Flows

Junction nodes enable approval-required relationships and role management
without parallel edges. All three junction types — ChatMember,
CompanyMember, ItemOwnership — share a common shape.

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
specific roles required toward the junction node." Open chats have N = 1
(the joining user); invite-only and request-entry chats have N = 2 (user +
admin, in either order); governance-heavy joins can require larger N with
weighted multi-sig (weights derived from role properties on the approving
actors' own junction nodes).

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
- **Removal by approving actor (kick / firing / non-renewal).**
  Approving actor adds a new negative-sentiment layer on their actor
  edge toward the junction node. System evaluates the approval policy;
  if the revocation threshold is met, system adds a new layer on the
  **approval-side** structural edge with `dim1 < 0`.
- **System-initiated** (auto-expiry, violation handling, etc.). System
  adds the appropriate negative layer directly.

**Intermediate states are not materialized.** For multi-sig policies
where N > 1 admins must act to kick a member, the approval-side
structural edge stays at its top layer until the policy threshold is
met. Partial progress is visible on individual actor edges; the
structural edge reflects only the policy-resolved state.

**Cascading updates across structural edges.** A state change on one
structural edge can trigger the system to add a corresponding layer on
another structural edge when consistency requires it. This is a
general mechanism — the canonical case today is ItemOwnership
supersession (see [items.md](items.md)), where creating a new approval
edge causes the previous one to be marked revoked so that exactly one
ownership is active at a time. Future junction or content patterns
may use the same cascade shape.

### Chat Membership (ChatMember)

Chat-specific flows (open / invite-only / request-entry) are explained
in [docs/chats.md](chats.md). They are all variants of the two-edge
approval pattern described above.

### Company Membership (CompanyMember)

Company-specific flows are explained in [docs/companies.md](companies.md).
They follow the same two-edge approval pattern described above.

### Ownership Transfer (ItemOwnership)

Item-specific flows are explained in [docs/items.md](items.md). They
follow the same two-edge approval pattern described above, with the
additional property that transfers form an append-only chain of
ItemOwnership nodes per item.

---

## 7. Dimension Semantics

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

## 8. Directionality: Inbound Edges Don't Affect Your Graph

This is a critical design decision for anti-spam and anti-manipulation:

**Edges created toward you by others do not change your feed.**

If a cluster of bots likes Jakob's posts 10,000 times:
- The bots now have strong edges toward Jakob — so Jakob appears high in
  *their* feeds.
- Jakob has zero edges toward the bots — they don't appear in *his* feed at
  all.
- The bot cluster gains nothing economically because the economically
  important nodes (real users, advertisers, companies) never point toward them.

This is only possible because all edges are directional. There is no concept
of an undirected "connection." A friendship is explicitly:
```
Jakob -[sentiment: +0.8, closeness: +0.9]-> Alice
Alice -[sentiment: +0.7, closeness: +0.9]-> Jakob
```

Two independent edges. Removing one does not remove the other.

---

## 9. Append-Only History

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
relationship. An edge with 1 layer is a passing interaction. How exactly to
use this signal is an open question (see section 10).

---

## 10. Open Questions

These are known unknowns that need to be resolved as the project progresses:

1. **Layer count usage**: The number of layers on an edge is a signal, but
   how does it factor into ranking? Is it a modifier on the dimension values?
   A separate ranking parameter?

2. **Cross-type dimension comparability**: When the ranking algorithm
   traverses `User -> User -> Comment -> Post`, it crosses three edge types
   with different dimension meanings. How exactly are
   sentiment-toward-a-user and sentiment-toward-a-post combined? The math is
   uniform (both are floats) but the semantics differ.

3. **Minimum interaction for edge creation**: Does viewing a post for 3
   seconds create an edge? Does scrolling past it? Where is the line between
   "implicit signal" and "explicit action"? This ties into the transparency
   principle — implicit signals feel like surveillance.



---

## 11. Relationship to Feed Ranking

The [feed ranking algorithm](feed-ranking.md) currently operates on simple
signed (+/-) edges. The tensor model described here is the next evolution:

- The ranking algorithm's `sign(U -> node)` becomes a function of the tensor
  dimensions (not just positive/negative, but a weighted combination of
  sentiment and relevance/closeness).
- The `h`, `i`, `j`, `k` metrics will need to operate on continuous values
  rather than discrete signs.
- The sort/order phases remain structurally the same, but the inputs become
  richer.

The basic signed-edge ranking is the **v0 implementation**. The full tensor
model is the **target state**. We build v0 first to validate the algorithm,
then evolve the edge model.
