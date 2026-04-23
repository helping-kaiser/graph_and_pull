# Edge Tensor Model

How edges work in the Peer Network graph. This is the foundation that the
[feed ranking algorithm](feed-ranking.md) operates on.

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
| **CompanyMember** | Company <-> User | Has roles (founder, shareholder, worker, band member). Multi-sig for adding/removing members. Ownership stakes. |
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
not by actors. The 2 dimensions are `(0.0, 0.0)` — neutral structural links.

Why give structural edges the same shape instead of making them different:
- The ranking algorithm traverses paths that cross both edge types (e.g.
  `User -> User -> Comment -> Post`). Uniform shape means no branching logic
  at each hop.
- Structural edges may carry meaningful weight in the future (e.g. a pinned
  comment's `Comment -> Post` edge could be weighted differently).

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
would carry the same fact and just duplicate storage. See §8 for the full
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
§9 for how negative values are interpreted when a dimension wouldn't
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

**Approval completion** (paired with the claim edges above — see §8):

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

## 6. Invitations

When an actor (User or Company) invites a new actor to the platform, **two
actor edges** are created:

```
Inviter -[sentiment: +X, closeness: +Y]-> New Actor   (layer 1: "I invited them")
New Actor -[sentiment: +X, closeness: +Y]-> Inviter   (layer 1: "they invited me")
```

Both are normal actor edges. This ensures the new actor has **at least one
outgoing edge** from the moment they join — without it, they would have zero
hops to anywhere in the graph and no way to calculate a feed.

The initial dimension values for the new actor's edge toward the inviter are a
design decision (likely moderate positive defaults — you presumably like the
person who invited you). The new actor can update this edge over time like any
other.

---

## 7. Authorship

There is no special authorship mechanism. Authorship is a **derived fact**:
the author of a node is the actor whose incoming edge has the earliest
layer 1 timestamp. A node cannot exist without someone creating it, so the
very first edge ever created toward a node identifies the author.

The dimension values on the author's edge are just normal opinion values —
the author's initial feelings about their own content (typically high positive
sentiment and relevance).

**Example:** Jakob creates a post. His actor edge `Jakob -> Post_X` is
layer 1, with the earliest timestamp of any incoming edge on Post_X. That
makes Jakob the author. Later, Alice likes the same post — her edge
`Alice -> Post_X` also has a layer 1, but its timestamp is later than
Jakob's. The author is always the earliest.

**Caching:** Looking up "who authored this?" by scanning all incoming layer 1
timestamps on every view would be expensive. The author ID should be cached:
- **On the node itself** as a property (`author_id`) — keeps the info in the
  graph for traversal queries.
- **In Postgres metadata** (e.g. `posts.author_id`) — for display queries
  that don't need the graph.

Both are derived caches. The graph (earliest incoming layer 1) is the source
of truth.

---

## 8. Junction Node Flows

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

### Chat Membership (ChatMember)

Three chat types, differing only in the approval policy:

**Open chat** — no approval required.
1. User creates an actor edge toward a new **ChatMember** node.
2. System creates `ChatMember -> Chat` (claim).
3. Policy satisfied immediately; system creates `Chat -> ChatMember`
   (approval).
4. User is an active member.

**Invite-only chat** — admin invites, user accepts.
1. Admin (or other member with invite rights) creates an actor edge toward
   a new **ChatMember** node for the invitee.
2. System creates `ChatMember -> Chat` (claim, pending).
3. Invitee creates an actor edge toward the same ChatMember node
   (accepting).
4. Policy satisfied (both required edges exist); system creates
   `Chat -> ChatMember` (approval).
5. User is an active member.

**Request-entry chat** — user requests, admin approves.
1. User creates an actor edge toward a new **ChatMember** node (request).
2. System creates `ChatMember -> Chat` (claim, pending).
3. An admin creates an actor edge toward the same ChatMember node
   (approving).
4. Policy satisfied; system creates `Chat -> ChatMember` (approval).
5. User is an active member.

The invite-only and request-entry flows are topologically identical — the
only difference is which party initiates. Multi-sig variants (e.g. two
admins required) are just a higher N in the approval policy.

### Company Membership (CompanyMember)

Same two-edge pattern as ChatMember. A CompanyMember node carries `role`
and any role-attached quantities (e.g. `ownership_pct` for a shareholder)
as properties on the node. Approval policies for adding or promoting
members are governed by role requirements — a new shareholder may require
approval from existing founders and/or a threshold of current
shareholders.

1. Actor creates an actor edge toward a new **CompanyMember** node.
2. System creates `CompanyMember -> Company` (claim).
3. Required approving actors create actor edges toward the same
   CompanyMember node.
4. Once the company's approval policy is satisfied, system creates
   `Company -> CompanyMember` (approval).
5. Actor is an active member.

### Ownership Transfer (ItemOwnership)

Each transfer creates a **new** ItemOwnership node. Historic ItemOwnership
nodes are never removed — they form an append-only chain of the item's
ownership history.

1. **Acquirer** (User/Company) creates an actor edge toward a new
   **ItemOwnership** node.
2. System creates `ItemOwnership -> Item` (claim, pending).
3. **Current owner** creates an actor edge toward the same ItemOwnership
   node with positive sentiment (approval).
4. Policy satisfied; system creates `Item -> ItemOwnership` (approval).
5. Transfer is complete; the new ItemOwnership is now the active one.

No one can take ownership without the current owner's explicit approval.
Earlier ItemOwnership nodes still have their `Item -> ItemOwnership` edges
from when they were current — the **current** owner is identified by the
most recent `Item -> ItemOwnership` approval edge (analogous to how
authorship is derived from the earliest incoming edge in §7). See §13 for
the open question on how superseded states — and departures like leaving a
chat or company — are encoded under append-only.

---

## 9. Dimension Semantics

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

## 10. Directionality: Inbound Edges Don't Affect Your Graph

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

## 11. Append-Only History

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
use this signal is an open question (see section 13).

---

## 12. Time Considerations

### Time decay (OPEN DESIGN QUESTION)

Time decay must exist in some form but is not yet fully designed. Known
constraints:

- Old content can become newly relevant (a friend comments on a post I liked
  years ago — I should see the comment, and the post becomes slightly more
  relevant again).
- New content can be irrelevant (a brand new post from someone 5 hops away
  that no one I know has interacted with).
- **Recency is not importance.** Time is a factor but not a dominant one.

### The "already seen" problem (OPEN DESIGN QUESTION)

Users should not be re-shown content they've already seen unless something
meaningful happened (e.g., a friend commented on it). This creates a problem:

**Option A: "View" edges (0, 0 sentiment/relevance edges for any node visited)**
- Pro: Clean graph-native solution. "I've seen this" is just another edge.
- Con: Explodes the edge count. Instead of sorting through 3 posts a friend
  liked, you sort through 10,000 posts they've viewed. Computation cost
  becomes untenable.

**Option B: Separate "seen" store outside the graph**
- Pro: Doesn't pollute the graph. Can use a compact data structure (bloom
  filter, bitset, Redis set).
- Con: Breaks the "everything is in the graph" purity. Adds a third data
  store.

**Option C: Client-side "seen" tracking**
- Pro: Aligns with the decentralized feed calculation vision (the client
  already has a subgraph). The client knows what it's shown the user.
- Con: Doesn't sync across devices without additional infrastructure.

**Option D: View edges with aggressive compaction**
- Pro: Graph-native. Only recent view edges are kept as individual layers;
  older ones are compacted into a summary.
- Con: Compaction logic adds complexity. Defining "recent" is another design
  decision.

This needs a dedicated design session. The solution must:
1. Not flood the graph with low-signal edges.
2. Not be a black box.
3. Allow users to revisit content manually.
4. Surface content again when something meaningful changes (new interactions
   from people the user cares about).

---

## 13. Open Questions

These are known unknowns that need to be resolved as the project progresses:

1. **Time decay function**: What shape? Exponential? Linear? Step function?
   How does it interact with the ranking algorithm's `R`, `h`, `i`, `j`, `k`?

2. **Layer count usage**: The number of layers on an edge is a signal, but
   how does it factor into ranking? Is it a modifier on the dimension values?
   A separate ranking parameter?

3. **Cross-type dimension comparability**: When the ranking algorithm
   traverses `User -> User -> Comment -> Post`, it crosses three edge types
   with different dimension meanings. How exactly are
   sentiment-toward-a-user and sentiment-toward-a-post combined? The math is
   uniform (both are floats) but the semantics differ.

4. **View/seen tracking**: See section 12. Needs a dedicated solution.

5. **Minimum interaction for edge creation**: Does viewing a post for 3
   seconds create an edge? Does scrolling past it? Where is the line between
   "implicit signal" and "explicit action"? This ties into the transparency
   principle — implicit signals feel like surveillance.

6. **Chat and ChatMessage ranking**: How do chats fit into the feed? Are they
   ranked alongside posts and comments, or do they have their own separate
   ranking context? A chat is a persistent container; a ChatMessage is
   ephemeral-feeling. The ranking dynamics likely differ from public content.

7. **Company vs User distinction**: Companies can do most things users can
   (author posts, own items, connect to hashtags). What can't they do? Can a
   Company follow a User? Can Companies have sentiment toward each other? The
   boundary between Company and User node types needs clarification —
   especially for the economic model where companies are ad-revenue sources.

8. **State transitions on junction relationships under append-only**: How
   are departures encoded — a member leaving a chat, getting kicked from a
   company, an ItemOwnership being superseded by the next transfer? The
   `Parent -> Junction` approval edge cannot be deleted (append-only), so
   "no longer active" needs a different representation. Candidates: a new
   layer on the approval edge marking revocation (requires structural edges
   to carry state in layers, which they currently don't); a dedicated
   "revoked" actor edge from admin/system; a separate junction-node
   property; deriving "current" purely from timestamps (works for
   sequential cases like ItemOwnership but not for leaving a chat).

9. **Invitation default values**: What sentiment/closeness values should the
   auto-created edge from the new actor toward the inviter have? Too high
   and it biases the new user's feed heavily toward one person. Too low and
   the new user has a weak starting position in the graph.

---

## 14. Relationship to Feed Ranking

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
