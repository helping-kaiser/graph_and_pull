# Edges

The full catalog of edge types in CoGra, plus the relationship-label
scheme used at the graph database layer.

For the conceptual model — what edges are, their dimensions, their
directionality, the append-only rule — see
[graph-model.md](graph-model.md).

---

## 1. Actor edges

All actor edges are created by actors (User or Collective) toward
other nodes. The 2 dimensions are set by the actor and follow the
uniform `[-1.0, +1.0]` range described in
[graph-model.md](graph-model.md).

Actor edges from a Collective are initiated by an authorized
CollectiveMember per the Collective's social contract, but the
on-graph edge belongs to the Collective itself — there is no
per-edge record of the acting member. See
[collectives.md "Acting through the Collective"](../instances/collectives.md#2-acting-through-the-collective).

**Invariant:** Edges attributed to a Collective carry no per-edge
record of the acting member. The Collective IS the actor at the
graph layer; member-level accountability lives in the Collective's
social contract, not in edge attribution. Deliberate non-feature —
not a gap to be filled by a future `acting_user` dimension.
Enforcement is **by absence**: the schema does not declare such a
property, and the service-layer write path never sets one. The
"rule" is the omission itself, not an unenforced convention.

Across every actor-edge type, `dim1` is signed valence and `dim2`
is signed connection-weight — the labels in the tables below
(sentiment, interest, relevance, importance) highlight the relevant
aspect per edge type; the math role of each dimension is uniform.
See [graph-model.md §6](graph-model.md#6-dimension-semantics) for
the full grammar.

### User as actor

| Edge type | Dimension 1 | Dimension 2 |
|-----------|-------------|-------------|
| User → User | **Sentiment** (love to hate) | **Interest** (how interested I am in their content / output — distinct from how well I know them) |
| User → Collective | **Sentiment** (love to hate) | **Interest** (how interested I am in this collective's output) |
| User → Post | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User → Comment | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User → Chat | **Sentiment** (like to dislike) | **Relevance** (how important is this chat to me) |
| User → ChatMessage | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User → ChatMember | **Sentiment** (endorse to reject) | **Importance** (how important is this membership to me) |
| User → CollectiveMember | **Sentiment** (endorse to reject) | **Importance** (how important is this membership to me) |
| User → ItemOwnership | **Sentiment** (endorse to reject) | **Importance** (for the initial owner, how important holding this item is; for transfer recipients, how important receiving the transfer is) |
| User → Item | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User → Proposal | **Sentiment** (support to oppose) | **Importance** (how strongly I want this change) |

### Collective as actor

| Edge type | Dimension 1 | Dimension 2 |
|-----------|-------------|-------------|
| Collective → User | **Sentiment** | **Relevance** (how valuable is this user to the collective) |
| Collective → Collective | **Sentiment** | **Relevance** |
| Collective → Post | **Sentiment** | **Relevance** |
| Collective → Comment | **Sentiment** | **Relevance** |
| Collective → Chat | **Sentiment** | **Relevance** |
| Collective → ChatMessage | **Sentiment** | **Relevance** |
| Collective → ChatMember | **Sentiment** (endorse to reject) | **Importance** |
| Collective → CollectiveMember | **Sentiment** (endorse to reject) | **Importance** |
| Collective → ItemOwnership | **Sentiment** (endorse to reject) | **Importance** (for the initial owner, how important holding this item is; for transfer recipients, how important receiving the transfer is) |
| Collective → Item | **Sentiment** | **Relevance** (how important is this product) |
| Collective → Proposal | **Sentiment** (support to oppose) | **Importance** (how strongly the collective wants this change) |

Actor edges to Hashtag are deliberately excluded from both tables.
Hashtag has no outgoing edges (see
[hashtag.md §4 "As source"](../instances/hashtag.md#as-source-outgoing))
so any actor edge ending at a Hashtag would terminate the path
without contributing to traversal further on. The cosmetic-only
rule — `:TAGGING` is the only edge ranking ever needs to touch on
the hashtag side, and it stays pure topology — lives in
[hashtag.md §4](../instances/hashtag.md#4-edges).

---

## 2. Structural edges

System-created. Dimensions default to `(0, 0)` unless the edge
participates in a **state-bearing pattern**. Two such patterns
exist:

- Junction approval pairs (claim / approval edges on `ChatMember`,
  `CollectiveMember`, `ItemOwnership`) — see
  [graph-model.md §5](graph-model.md#5-junction-node-flows).
- `:REFERENCES`, with a 2D tensor in `[-1, +1]` and a fanout-budget
  invariant — see "Reference" below.

For a matrix and diagram view of every structural edge in the
catalog see [structural-edge-map.md](structural-edge-map.md).

**Invariant:** A given `(source, target)` pair carries at most one
edge **label**. Layers within that single label are how the pair
accumulates history; a second label between the same endpoints —
actor or structural — is forbidden. When a relationship is already
covered by one label (e.g. `Post → Hashtag` via `:TAGGING`), the
same pair never gets a second label of a different type (no
parallel `Post → Hashtag` via `:REFERENCES`). This is what drives
the `:TAGGING` / `:REFERENCES` carve-out below.

The rule covers actor edges too: a Collective that is the parent
of a CollectiveMember cannot carry both `:APPROVAL` and `:ACTOR`
toward that member — `:APPROVAL` is sufficient and wins, since
that pair already exists by the time any sentiment edge would be
written. Cases where a `Collective → CollectiveMember` actor edge
*is* legal — sub-Collective Shape A self-claim (the bearer's own
edge), or sentiment from a third-party Collective — are
different `(source, target)` pairs and unaffected. The catalog
row in §1 stays, scoped by this rule.

**Enforcement.** Phrased as a single insert-time check — when
writing an edge between `(A, B)` with label L, abort if any
existing edge between `(A, B)` has label L' ≠ L.

The service layer is the primary check: before inserting a new
edge it queries existing edges between the same endpoints and
rejects the write if their label differs. A Memgraph trigger on
edge create (reading `startNode(e), endNode(e), type(e)` and the
existing `(A)-[*]->(B)` relationships) is the storage-level
backstop for code paths that bypass the service layer. The
service layer is primary because it returns a meaningful error to
the caller; the trigger is the safety net. See
[graph-data-model.md "Single-edge-label enforcement"](../implementation/graph-data-model.md#single-edge-label-enforcement)
for the Cypher.

### Containment / belonging

| Edge type | Meaning |
|-----------|---------|
| Comment → Post | This comment is on this post |
| Comment → Comment | This comment is a reply to that comment |
| Comment → Chat | This comment is on this chat as a whole |
| Comment → ChatMessage | This comment is on this specific message |
| Comment → Item | This comment is on this item |
| ChatMessage → Chat | This message belongs to this chat |
| ChatMember → Chat | This membership claims to be about this chat (claim) |
| CollectiveMember → Collective | This membership claims to be about this collective (claim) |
| ItemOwnership → Item | This ownership claim relates to this item (claim) |

### Approval completion

Paired with the claim edges above — see
[graph-model.md](graph-model.md) for the two-edge approval pattern.

| Edge type | Meaning |
|-----------|---------|
| Chat → ChatMember | This chat has accepted this member |
| Collective → CollectiveMember | This collective has accepted this member |
| Item → ItemOwnership | This item's ownership transfer to this claim is complete |

### Bearer binding

System-created at junction creation to bind the junction to the
actor it represents. Distinct from the claim/approval pair: claim
and approval encode the relationship between the junction and its
*parent* (Chat / Collective / Item); the bearer edge encodes the
relationship between the junction and its *bearing actor*.

| Edge type | Meaning |
|-----------|---------|
| ChatMember → User \| Collective | This chat membership represents this actor |
| CollectiveMember → User \| Collective | This collective membership represents this actor |
| ItemOwnership → User \| Collective | This ownership belongs to this actor |

The edge is written once at junction creation and never re-pointed
— a junction's bearer is its identity. Changing who a junction
represents is conceptually a *different* junction; ownership
transfers (`ItemOwnership`) and impersonation defenses all rely on
this — a new bearer means a new junction node with a new `:BEARER`
edge, not a re-pointing of the existing one. Key rotation on the
bearing actor doesn't shift `:BEARER` either: the edge points at
the actor's graph identity (User or Collective node), which is
stable across key changes. The actor's Shape A self-claim that
activates the junction must originate from the actor at the other
end of `:BEARER`; the API rejects mismatched self-claims. This is
what enables invite-only flows
([chats.md §11](../instances/chats.md#11-joining-and-leaving-a-chat)):
the inviter creates the junction with a known bearer before the
invitee acts.

**Invariant: bearer/self-claim validation is atomic.** The
service-layer transaction that accepts a self-claim reads the
existing `:BEARER` edge and refuses to commit if the claimer is
not the bearer. The validation + the claim write happen in one
transaction; mismatches never partially land. The `:BEARER`
edge itself may predate the self-claim (the invite-only window
above is intentional); the atomicity covers the check + claim,
not the creation of both.

Common queries:
- "What junctions am I bearer of (active and pending)?" — single
  inbound traversal from the actor along `:BEARER`.
- "Who is the bearer of this junction?" — single outbound
  traversal from the junction along `:BEARER`.

### Tagging

| Edge type | Meaning |
|-----------|---------|
| Post → Hashtag | This post is tagged with this hashtag |
| Comment → Hashtag | This comment is tagged with this hashtag |
| Item → Hashtag | This item is tagged with this hashtag |

### Subject targeting

System-created when a Proposal node is created. Records which node
the Proposal targets (the node whose property is proposed for
change — see [governance.md §2.1](governance.md#21-subject)).

| Edge type | Meaning |
|-----------|---------|
| Proposal → Target Node (any node) | This proposal targets this node's property |

The property name and proposed value are properties on the Proposal
node, not on this edge — the change is intrinsic to the Proposal,
not to the relationship.

### Reference

System-created when a content node embeds another node — a
ChatMessage sharing a post into a chat, a Post quoting another
Post, a Comment citing the original of a re-uploaded image, any
of the three mentioning an actor, pointing at a proposal to vote
on, etc. The **carrier** is a content node —
ChatMessage, Post, or Comment — with its own author, timestamp,
and Postgres body. The referenced node is what the carrier points
at. See [chats.md](../instances/chats.md) for the worked-out
ChatMessage usage patterns (including the personal-newsfeed shape
that replaces the old container-chat hack); per-carrier specifics
live in the respective [post.md](../instances/post.md) and
[comment.md](../instances/comment.md) edge sections.

| Edge type | Meaning |
|-----------|---------|
| ChatMessage → any node | This message references this node |
| Post → any node (except Hashtag) | This post references this node |
| Comment → any node (except Hashtag) | This comment references this node |

Targets span every node category: actor (User, Collective), content
(Post, Comment, Chat, ChatMessage, Item, Hashtag, Proposal), and
junction (ChatMember, CollectiveMember, ItemOwnership). A carrier
can point at anything with a graph identity.

**`:REFERENCES` is not written when another structural edge
already encodes the same `(source, target)` pair.** `:REFERENCES`
is the only structural edge with an open target set, so it is the
only one that can structurally collide with another label on the
same pair. The single-structural-edge invariant in §2 resolves
the collision in favor of the more specific edge; `:REFERENCES`
yields.

Two cases follow from the rule:

1. **Hashtag.** Post and Comment already attach to Hashtag via
   `:TAGGING`. `Post → Hashtag` and `Comment → Hashtag` therefore
   carry `:TAGGING` only — never a parallel `:REFERENCES`.
   `ChatMessage → Hashtag` does use `:REFERENCES` because
   ChatMessage has no `:TAGGING` edge type, so there is nothing
   to collide with.
2. **Own `:CONTAINMENT` parent.** A Comment whose body
   quotes/embeds the very Post / Comment / Chat / ChatMessage /
   Item it is posted on does not write a parallel `:REFERENCES`
   edge to that parent — the `:CONTAINMENT` edge already encodes
   the pair. The same applies to a ChatMessage whose body embeds
   its own home Chat. The containment edge is the one structural
   record of the pair; the frontend renders the embed from that
   edge (plus the body markup) without a second graph edge.

`:REFERENCES` is a **state-bearing structural edge** — the second
member of that pattern after junction approval pairs. The edge
carries a 2D tensor `(dim1, dim2)` in `[-1, +1]`, the same shape
as actor edges.

**Invariant: `:REFERENCES` fanout-budget.** Across all outbound
`:REFERENCES` edges from a single content node, top-layer values
must satisfy:

```
sum of |dim1| ≤ 1
sum of |dim2| ≤ 1
```

independently on each dimension. Default values (when none are
explicitly set): uniform `(1/N, 1/N)` on the top layer, where `N`
is the source node's outbound `:REFERENCES` count. The source
node's author may set explicit values within the budget — e.g.
`(0.9, 0.5)` on one reference and `(0.1, 0.5)` on another, per
dimension independent. The constraint applies to top layers only;
historical layers contribute nothing to ranking. Updating one
reference's weight may require re-balancing siblings to stay
within budget. The total may be less than 1 — a single weak
reference at `(0.2, 0.1)` is valid; the budget need not be fully
spent. Negative weights are allowed within the magnitude budget
for "I'm quoting this to disavow it" semantics.

Feed-ranking traversal rules for `:REFERENCES` (endpoint
restrictions, fanout-budget composition) live in
[feed-ranking.md §3.5 "Traversal restrictions"](feed-ranking.md#35-traversal-restrictions)
rules 4 and 5.

**Enforcement.** Every `:REFERENCES` write originating from a
given source node runs inside one service-layer transaction
(see
[architecture.md "Service-layer transactions"](../implementation/architecture.md#service-layer-transactions)).
Inside that transaction the service layer reads the existing
sibling top-layer weights, computes the post-write sum, and
rejects the write if either dimension would exceed `1`. If a
new reference requires lowering existing siblings to fit (e.g.
the author shifts emphasis), the rebalancing layers on each
sibling write inside the same transaction — the budget is
never observably breached at any commit point.

### Voting (Shape B)

System-created when a voter casts a Shape B vote (see
[governance.md §3](governance.md#3-the-two-vote-shapes)). The
edge runs from the voter's **eligibility junction** to the
subject; `dim1` carries vote direction (`+1` support, `-1`
oppose, intermediate values allowed), `dim2` is `0`.

| Edge type | Meaning |
|-----------|---------|
| ChatMember → ChatMember | Admission (`dim1 > 0`) vote on another chat member's membership at join time. Stance flips on this edge happen only during the open admission period; disavowal of an active member routes through a Proposal — see [chats.md §10](../instances/chats.md#10-moderation) Level 2. |
| ChatMember → Proposal | A chat-eligible vote on any Proposal targeting a chat-internal subject — chat property / role change, Level 1 message disavowal, or Level 2 member disavowal |
| CollectiveMember → CollectiveMember | Approval (`dim1 > 0`) or removal (`dim1 < 0`) vote on another collective member's membership |
| CollectiveMember → Proposal | A collective-eligible vote on a proposed property/role change |
| ItemOwnership → ItemOwnership | Current owner's approval vote on a transfer to a new ItemOwnership for the same Item |

**Same edge serves the membership lifecycle (CollectiveMember,
ItemOwnership).** For `CollectiveMember → CollectiveMember`
and `ItemOwnership → ItemOwnership`, the same edge that admits
a junction holder can later be re-layered to remove them —
`dim1 > 0` admits at layer-1, a later `dim1 < 0` layer is the
removal vote. One edge, layered history of one voter's stance
toward one membership / ownership. See
[graph-model.md §5](graph-model.md#5-junction-node-flows)
"Revocation and state transitions".

**ChatMember → ChatMember is admission-only.** The chat
case diverges: the edge supports stance flips during the open
admission period only. Once a membership is active, disavowal
flows through a Proposal targeting the member's ChatMember
junction (see [chats.md §10](../instances/chats.md#10-moderation)
Level 2), not through a re-layering of this edge.

**Network-scope governance uses Shape A, not Shape B.** Votes
on Network-wide Proposals (moderator role changes, content
moderation, `:Network` parameter amendments) are mechanically
the existing `User → Proposal` **actor edge** from §1, since
the Network has no per-member junction to vote from. The actor
edge keeps its normal `(sentiment, importance)` meaning; the
tally reads `sign(sentiment)` for the binary outcome. See
[governance.md §3](governance.md#3-the-two-vote-shapes)
"Shape A" and
[proposal.md §4](../instances/proposal.md#4-edges).

---

## 3. Edge labels at the graph layer

In Memgraph (and Cypher generally), every relationship carries
exactly one **type label**. Labels let queries filter relationships
efficiently without scanning properties or walking every incident
edge.

The trick is naming them at the right granularity. Too few labels
and every query has to filter by endpoint type too. Too many labels
and the schema explodes every time a node type is added.

### Base categories

| Label | Applies to | Description |
|---|---|---|
| `:ACTOR` | All actor edges | Created by actors (User or Collective); carries the 2-dimensional opinion tensor. Uniform across every actor-edge type — specific meaning (sentiment-toward-post vs interest-in-user, etc.) derives from endpoint node labels. |
| `:STRUCTURAL` | All structural edges not otherwise labeled | System-created edges expressing containment or belonging. Dimensions typically `(0, 0)` unless they participate in a state-bearing pattern. |

### Sub-category labels

Sub-labels exist for edges whose query patterns differ enough
that the endpoint-label-filter approach adds cost or noise. Most
are structural; `:AUTHOR` is the one actor-edge sub-label.

| Label | Applies to | Rationale |
|---|---|---|
| `:AUTHOR` | User \| Collective → authored content node (Post, Comment, Chat, ChatMessage, Item, Proposal) | The author's authoring actor edge — the first outgoing actor edge from author to authored content per [authorship.md](authorship.md). Frequently queried as "what did X author?" — a single-label scan instead of a scan-and-timestamp-compare across all of X's outgoing actor edges. Also used by the feed-ranking author-hop traversal rule ([feed-ranking.md §3.5](feed-ranking.md#35-traversal-restrictions)). Same 2D tensor and `[-1, +1]` range as `:ACTOR`; label is permanent across layers (re-layering updates `(dim1, dim2)` only). |
| `:CLAIM` | Junction → Parent (e.g. `ChatMember → Chat`) | The claim side of the two-edge approval pattern. Frequently queried as "what is this actor a member of (including pending)?" |
| `:APPROVAL` | Parent → Junction (e.g. `Chat → ChatMember`) | The approval side. "Is this relationship currently active?" queries scan only `:APPROVAL` edges with positive top-layer `dim1`. |
| `:BEARER` | Junction → User \| Collective (e.g. `ChatMember → User`) | Identity binding for a junction. "What junctions does this actor bear?" and "who is this junction's bearer?" run as single one-hop traversals along `:BEARER`. |
| `:CONTAINMENT` | Comment → Post, Comment → Comment, ChatMessage → Chat, Comment → Chat, Comment → ChatMessage, Comment → Item | Content containment and reply structure. Queried for feed assembly and thread rendering. |
| `:TAGGING` | Post → Hashtag, Comment → Hashtag, Item → Hashtag | Tag associations. Queried by hashtag-centric browsing. |
| `:TARGETS` | Proposal → Target Node | The proposal-to-subject relationship. Common query: "what proposals target this node?" needed by the governance cascade. |
| `:REFERENCES` | ChatMessage → any node; Post / Comment → any node except Hashtag | The "this carrier embeds X" relationship. Common query: "what nodes reference this one?" — feeds embed-rendering and inbound-attention surfaces. |

All sub-category labels **replace** their base label (`:ACTOR`
for `:AUTHOR`, `:STRUCTURAL` for the rest), not add to it — a
relationship has exactly one label in Memgraph.

### What about the rest of actor edges?

Non-`:AUTHOR` actor edges stay uniform at `:ACTOR`. The 2D tensor
treats all actor edges the same math-wise; splitting them by
tuple would multiply labels (User-Post, User-User,
Collective-Post, ...) without improving ranking efficiency — the
ranking algorithm iterates over actor edges regardless of tuple.

Endpoint node labels (`:User`, `:Post`, `:Chat`, etc.) already let
queries filter by meaning: `(u:User)-[:ACTOR]->(p:Post)` binds the
semantics without needing a `:USER_POST` label.

`:AUTHOR` is the exception because authorship has *both* a
query-shape distinct enough to merit its own scan (the
"earliest-incoming-edge" lookup it replaces is awkward in pure
Cypher) and a load-bearing role in feed-ranking traversal (the
author-hop rule). Other actor edges have neither.

---

## 4. Extension policy

Add a new edge type to the catalog when a new semantic relationship
is needed — these additions should always be discussed as part of
the broader design, not added silently.

Add a new **label** (not a new edge type) only when a query pattern
proves both **common** and **awkward to express** with the current
scheme. A per-tuple label for every combination was considered and
rejected for schema churn. A label change is a schema migration, so
the bar to adding one is deliberately higher than adding new nodes
or edge dimensions.

