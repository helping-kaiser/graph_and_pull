# Graph Model

How edges, nodes, and their properties work in the Peer Network graph.
This is the foundation that the [feed ranking algorithm](feed-ranking.md)
operates on.

---

## 1. Core Principles

Every edge in the graph is:
- **Directional** — `A → B` and `B → A` are separate edges. A friendship is
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

Nodes fall into four categories:

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
- **System nodes** — singletons that carry instance-level
  configuration governed via Proposals (Network). They aren't
  actors, aren't content, and don't represent relationships;
  they exist because governance needs a node to target.

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
(sentiment, interest, relevance — varies by edge type, see [edges.md](edges.md)).

### Structural edges

Express **containment or belonging** between nodes. Created by the system,
not by actors. By default the 2 dimensions are `(0, 0)` — neutral
structural links.

Why give structural edges the same shape instead of making them different:
- The ranking algorithm traverses paths that cross both edge types (e.g.
  `User → User → Comment → Post`). Uniform shape means no branching logic
  at each hop.
- Structural edges can carry meaningful weight where the shape calls for
  it. The concrete case today is state-bearing approval-pattern edges on
  junction nodes — see §5 for how revocation and state transitions are
  encoded in structural edge layers. A pinned comment's `Comment → Post`
  weight could work similarly.

### Structural edge pairs

Structural edges are **not paired for query convenience**. Memgraph (and
openCypher generally) indexes relationships at both endpoints, so a single
one-directional edge is traversable in either direction with equal
efficiency. Adding a reverse edge just so a query reads more naturally
would double storage for no gain.

Structural edge pairs **are valid when each direction encodes a distinct
fact**. The canonical example is approval-required junctions:

- `ChatMember → Chat` — "this membership claims to be about this chat"
  (exists from the moment the request is made).
- `Chat → ChatMember` — "this chat has accepted this member" (only exists
  after the approval policy is satisfied).

These are two different facts, so two edges is correct. In contrast,
`Comment → Post` does not need a `Post → Comment` companion: the reverse
would carry the same fact and just duplicate storage. See §5 for the full
junction approval pattern.

### What creates an actor edge — stances-not-events

Actor edges are created or updated only when an **actor** (User
or Collective) takes an **explicit, deliberate gesture** that
expresses a position toward a node. The graph encodes actor
**stances** — relationships, opinions, intents — not session
**events** like scrolling, hovering, or briefly opening content.

The basic operation is: an actor sets the dimensions on an actor
edge from themselves to a target node — either creating a new
edge or adding a new layer to an existing one. Expressing
sentiment toward another node, calibrating interest or
relevance, taking any other position the dimensions can encode
— all reduce to this.

Compound gestures defined elsewhere also reduce to setting
dimensions on actor edges, sometimes creating multiple edges in
one operation:

- Authoring a node ([authorship.md](authorship.md)) — the
  author's first outgoing edge to their newly-created content.
- Joining or leaving a chat or collective — the actor's edge to
  the relevant junction (§5).
- Inviting a new actor ([invitations.md](invitations.md)).
- Casting a vote in a governance instance
  ([governance.md](governance.md)).

What does NOT create or update an actor edge:

- Scrolling past content.
- Dwell time, read time, hover.
- Brief preview / peek.
- Viewing a post or opening a chat without further action — even
  repeated opens.
- Search queries.
- Sharing content externally (link copy, share-to-another-app,
  export). The act of sharing is a frontend event — not a stance
  the actor took toward the content.
- Bookmarking content. Bookmarks are private per-user state (see
  `user_bookmarks` in [data-model.md](../implementation/data-model.md)),
  not a public stance — they say "I want to find this later," not
  "I want this to reach my network."
- Tagging an own post with a hashtag. The `Post → Hashtag`
  structural edge already encodes the association; the actor
  reaches the hashtag via the `Actor → Post → Hashtag` path.
  An autogenerated `Actor → Hashtag` actor edge would be the
  system inferring a stance the actor did not directly take.

The frontend can keep session data **locally** (already-seen,
recently-viewed, prompts based on observed behavior). That data
never becomes graph state unless the actor makes an explicit
gesture in response.

#### Why

- **Transparency** ([CLAUDE.md](../../CLAUDE.md) principle #6).
  Every edge corresponds to something the actor consciously did.
  The graph is not a surveillance log.
- **Auditability of bot activity.** Bot accounts trying to
  influence others' graphs have to take explicit, layer-creating
  gestures rather than feed engagement through invisible
  implicit-signal channels — every interaction is visible on the
  graph. Combined with §7 (inbound edges don't affect a viewer's
  feed), this leaves bot farms little leverage.
- **Freedom of the mind** ([CLAUDE.md](../../CLAUDE.md) principle
  #8). The system doesn't reward outrage or measure involuntary
  attention. It knows only what actors chose to tell it.

#### Acceptable cost

An actor who lurks chats without ever taking a position on them
won't have those chats reinforce their feed — even if they look
at them every day. This is deliberate. The system doesn't infer
preference from behavior; if an actor wants signal, they make a
gesture.

#### Frontend latitude

The graph layer doesn't enforce this — it accepts whatever edges
actors create. CoGra's reference frontend follows the rule
above, and any frontend aligned with the project's principles
should too. A frontend that creates `(0, 0)` view-edges or
silently translates dwell time into edge layers is technically
possible but contradicts the transparency principle and pollutes
the graph for everyone reading it.

#### Structural edges follow topology

Structural edges (containment like `ChatMessage → Chat`,
approval pairs, tagging like `Post → Hashtag`) are
system-created when graph topology demands them — typically as a
side effect of an actor's stance gesture. The rule above governs
actor edges directly; structural edges are governed by the
topology rules in §5 and the per-instance flows in
[chats.md](../instances/chats.md),
[collectives.md](../instances/collectives.md), and
[items.md](../instances/items.md).

---

## 4. Edge Structure

Every edge, regardless of category, has the same shape:

```
Edge {
    // --- 2 dimensions (meaning varies by edge type) ---
    dimension_1: f64,   // actor edges: e.g. sentiment, range [-1.0, +1.0]
                        // structural edges: 0
    dimension_2: f64,   // actor edges: e.g. relevance, range [-1.0, +1.0]
                        // structural edges: 0

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
   `ChatMember → Chat`). The claim exists as long as the junction node
   exists.
2. **Approval edge** — when the relationship's approval policy is
   satisfied (all required actor edges toward the junction node exist), the
   system creates the reverse structural edge (e.g. `Chat → ChatMember`).
   The presence of this edge marks the relationship as *active*.

**State is encoded in the graph topology itself** — no status flag is
needed:

- Only the claim edge exists → pending.
- Both edges exist → active.

The **approval policy** for each relationship is "N actor edges
from specific roles required toward the junction node" — an
instance of the threshold policy described in
[governance.md §2.4](governance.md). N ranges from 1 (single
approver: the joining actor or a single decider) to multi-sig with
weighted votes (weights derived from role properties on the
approving actors' own junction nodes, per
[governance.md §2.3](governance.md)). Specific applications pick
their N — see [chats.md](../instances/chats.md) and
[collectives.md](../instances/collectives.md).

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

### A unified two-axis grammar

Across all actor edges, the two dimensions follow a uniform grammar:

- **`dim1` is signed valence** — sentiment, approval, affirmation. The
  "do I feel positively or negatively about this?" axis.
- **`dim2` is signed connection-weight** — interest, relevance,
  importance. The "how much does this matter to me?" axis.

The user-facing **labels** vary by edge type to surface the relevant
aspect (interest on `User → User`, relevance on `User → Post`,
importance on `User → ChatMember`, etc.). The **role** each dimension
plays in the math is uniform: dim1 carries direction; dim2 carries
weight. See [edges.md](edges.md) for the per-edge-type label catalog.

This unification keeps the ranking math single-shape — the algorithm
reads `(dim1, dim2)` from any actor edge without branching on edge
type. The interpretation (sentiment vs. interest, sentiment vs.
relevance) lives at the user-presentation layer; the math sees a
uniform 2D tensor. See [feed-ranking.md §3](feed-ranking.md) for how
the two axes compose along a path under different rules — `dim1` via
signed multiplication (signed-graph balance), `dim2` via taint sign ×
magnitude product (no transitivity for connection).

### Interest is not personal closeness

The `dim2` label on `User → User` edges is **interest**, not personal
closeness. The two are easy to conflate but the math depends on keeping
them distinct.

- **Personal closeness** (proximity in the social sense — frequency of
  interaction, how well you know someone) is *not* what `dim2` measures.
  Two users can know each other for decades and see each other every
  day; this does not by itself imply high `dim2`.
- **Interest** is how much you want the target's content/output flowing
  through your feed — their posts, comments, items, hashtags. It is
  a viewer-side judgment about *content relevance*, not a
  relationship-depth statement.

The decoupling is real and important. A valid edge shape:

```
+1 sentiment, -0.5 interest  →  "I love this person, but their
                                   content isn't for me"
```

This composes correctly under the existing math: the sentiment chain
(`s_path`) carries the affection through traversal via signed
multiplication, while the interest chain (`c_path`) is tainted negative
so the path does not amplify the target's content into the viewer's
feed. Loving someone and not following their posts are independent
positions on the graph; the dim grammar respects that.

The same independence applies on `User → Collective`, `User → Hashtag`,
and elsewhere. `dim2` always asks "how much do I want this in my feed,"
not "how close are we."

### Range and polarity

Every actor-edge dimension is bipolar in `[-1.0, +1.0]`:

- `0` = no opinion / no interest / neutral.
- Positive = the "forward" meaning (like, approve, want-to-see, relevant).
- Negative = the **active opposite**, not merely the absence.

The polarity matters most where the forward meaning sounds like a one-sided
scale — most notably **interest**. An interest of `0` means "I don't
engage with this target's content"; a negative interest means "I am actively
avoiding this content / output" (muted, blocked, ghosted). The two are
distinct signals, and collapsing negative interest into `0` would discard
real information. The same reading extends to relevance (negative = "I
actively don't want this in my feed") and to approval dimensions on junction
nodes (negative = active rejection, not abstention).

Holding the full `[-1.0, +1.0]` range for every dimension also keeps the
ranking math uniform and avoids per-dimension clamping or branching logic.

### Negative `dim2` in the graph math vs. as a frontend filter

The graph math uses negative `dim2` as a **continuous taint signal**
(see [feed-ranking.md §3.4](feed-ranking.md)): a path that crosses an
avoided connection has its interest signal flipped negative, but its
magnitude is the natural product of `|dim2|` along the path —
proportional to the rest of the path's strength. Negative `dim2` is
*not* snapped to zero in the math.

Hard "never show me X" exclusion is a separate, **frontend concern**
— applied as a post-ranking filter, not encoded in the math. The math
stays smooth and the frontend layers exclusions on top. This
separation lets users tune their feed in two distinct ways: by
expressing graduated stances (which the math reads continuously) and
by enforcing absolute exclusions (which the UI applies after the
fact).

### Independence of dimensions

The two dimensions are independent. Examples:

- **High sentiment, low relevance**: I'm glad a foreign dictator was removed
  from power (+0.75 sentiment), but I have no ties to that country and I'm not
  into politics (-0.5 relevance).
- **Low sentiment, high relevance**: I don't have strong feelings about a new
  tax law (0 sentiment), but it directly affects my business (+0.9
  relevance).
- **User → User**: I love my childhood best friend (+0.9 sentiment), but our
  hobbies have diverged completely; their posts are not what I want in my
  feed (-0.5 interest). Personal closeness is real; interest in their
  content is not.

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
Jakob -[sentiment: +0.8, interest: +0.9]-> Alice
Alice -[sentiment: +0.7, interest: +0.9]-> Jakob
```

Two independent edges. Removing one does not remove the other.

---

## 8. Append-Only History (edges)

This section covers the edge-specific shape of append-only history.
For the project-wide principle — including node properties and
Postgres-side display content — see [layers.md](layers.md).

Each edge is not a single value but a stack of layers:

```
Jakob → Post_X:
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

**Layers are metadata for audit, history, and UI** — not a ranking
input. Ranking sees only the **top layer** of each edge: the user's
current expressed stance. Layer count, layer timestamps, and the
sequence of past values are available to UI surfaces (e.g., a "this
edge has been revised N times" indicator, or a stale-edge prompt
suggesting review) and to anyone auditing the graph's history, but
they do not amplify or attenuate the ranking math.

This follows from **stances-not-events** (§3): the graph trusts
the user's last-expressed stance until they change it. Most users
update reactively — when they notice their feed reflecting
connections they no longer care about — rather than actively
maintaining edges, similar to pruning a stale subscription list.
The system does not infer intent from interaction frequency; it
reflects what the user last said.

---

## 9. Relationship to feed ranking

The [feed ranking algorithm](feed-ranking.md) is a general rule for
ordering target nodes in any graph with 2D-tensor edges from a root
node's perspective. It is deliberately layer-agnostic — the math
applies regardless of what the dimensions represent.

This document defines the **concrete inputs** the ranking algorithm
operates on in CoGra:

- Node categories (actor, content, junction) — §2.
- Edge categories (actor, structural) — §3.
- The uniform 2-dimensional `[-1.0, +1.0]` tensor shape — §4.
- Directional semantics — §7 (inbound edges don't affect the viewer's
  feed).
- Append-only layer stacks — §8 (the current state is the top layer).

The ranker composes these inputs along each path via **parallel
tracks**: `dim1` chain via signed multiplication (signed-graph
balance), `dim2` chain via taint sign × magnitude product (no
transitivity for connection-weight). Each metric (`h`, `i`, `j`,
`k`) is itself a 2-tuple `(sentiment-component, interest-component)`,
collapsed to a scalar (default: sum) only at sort time. See
[feed-ranking.md §3-§4](feed-ranking.md) for the full rule.
