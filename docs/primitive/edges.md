# Edges

The full catalog of edge types in CoGra, plus the relationship-label
scheme used at the graph database layer.

For the conceptual model — what edges are, their dimensions, their
directionality, the append-only rule — see
[graph-model.md](graph-model.md).

---

## 1. Actor edges

All actor edges are created by User or Collective nodes toward other
nodes. The 2 dimensions are set by the actor and follow the uniform
`[-1.0, +1.0]` range described in
[graph-model.md](graph-model.md).

Across every actor-edge type the two dimensions follow the same
underlying grammar (see [graph-model.md §6](graph-model.md)):
`dim1` is **signed valence** (sentiment / approval / affirmation);
`dim2` is **signed connection-weight** (interest / relevance /
importance). The labels in the tables below differ to highlight the
relevant aspect of each edge type, but the role each dimension plays
in the math is uniform.

### User as actor

| Edge type | Dimension 1 | Dimension 2 |
|-----------|-------------|-------------|
| User -> User | **Sentiment** (love to hate) | **Interest** (how interested I am in their content / output — distinct from how well I know them) |
| User -> Collective | **Sentiment** (love to hate) | **Interest** (how interested I am in this collective's output) |
| User -> Post | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User -> Comment | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User -> Chat | **Sentiment** (like to dislike) | **Relevance** (how important is this chat to me) |
| User -> ChatMessage | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |
| User -> ChatMember | **Sentiment** (approve to reject) | **Importance** (how important is this membership to me) |
| User -> CollectiveMember | **Sentiment** (approve to reject) | **Importance** (how important is this membership to me) |
| User -> ItemOwnership | **Sentiment** (approve to reject) | **Importance** (how important is this transfer to me) |
| User -> Item | **Sentiment** (want to avoid) | **Relevance** (how interesting to me) |
| User -> Hashtag | **Sentiment** (like to dislike) | **Relevance** (how interesting to me) |

### Collective as actor

| Edge type | Dimension 1 | Dimension 2 |
|-----------|-------------|-------------|
| Collective -> User | **Sentiment** | **Relevance** (how valuable is this user to the collective) |
| Collective -> Collective | **Sentiment** | **Relevance** |
| Collective -> Post | **Sentiment** | **Relevance** |
| Collective -> Comment | **Sentiment** | **Relevance** |
| Collective -> Chat | **Sentiment** | **Relevance** |
| Collective -> ChatMessage | **Sentiment** | **Relevance** |
| Collective -> ChatMember | **Sentiment** (approve to reject) | **Importance** |
| Collective -> CollectiveMember | **Sentiment** (approve to reject) | **Importance** |
| Collective -> ItemOwnership | **Sentiment** (approve to reject) | **Importance** |
| Collective -> Item | **Sentiment** | **Relevance** (how important is this product) |
| Collective -> Hashtag | **Sentiment** | **Relevance** |

---

## 2. Structural edges

System-created. Dimensions default to `(0, 0)` unless the edge
participates in a state-bearing pattern (junction approval pairs —
see [graph-model.md](graph-model.md) for the rule).

### Containment / belonging

| Edge type | Meaning |
|-----------|---------|
| Comment -> Post | This comment is on this post |
| Comment -> Comment | This comment is a reply to that comment |
| Comment -> Chat | This comment is on this chat as a whole |
| Comment -> ChatMessage | This comment is on this specific message |
| Comment -> Item | This comment is on this item |
| ChatMessage -> Chat | This message belongs to this chat |
| ChatMember -> Chat | This membership claims to be about this chat (claim) |
| CollectiveMember -> Collective | This membership claims to be about this collective (claim) |
| ItemOwnership -> Item | This ownership claim relates to this item (claim) |

### Approval completion

Paired with the claim edges above — see
[graph-model.md](graph-model.md) for the two-edge approval pattern.

| Edge type | Meaning |
|-----------|---------|
| Chat -> ChatMember | This chat has accepted this member |
| Collective -> CollectiveMember | This collective has accepted this member |
| Item -> ItemOwnership | This item's ownership transfer to this claim is complete |

### Tagging

| Edge type | Meaning |
|-----------|---------|
| Post -> Hashtag | This post is tagged with this hashtag |
| Item -> Hashtag | This item is tagged with this hashtag |

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
| `:ACTOR` | All actor edges | Created by User or Collective actors; carries the 2-dimensional opinion tensor. Uniform across every actor-edge type — specific meaning (sentiment-toward-post vs interest-in-user, etc.) derives from endpoint node labels. |
| `:STRUCTURAL` | All structural edges not otherwise labeled | System-created edges expressing containment or belonging. Dimensions typically `(0, 0)` unless they participate in a state-bearing pattern. |

### Sub-category labels

Sub-labels exist for structural edges whose query patterns differ
enough that the endpoint-label-filter approach adds cost or noise.

| Label | Applies to | Rationale |
|---|---|---|
| `:CLAIM` | Junction → Parent (e.g. `ChatMember -> Chat`) | The claim side of the two-edge approval pattern. Frequently queried as "what is this actor a member of (including pending)?" |
| `:APPROVAL` | Parent → Junction (e.g. `Chat -> ChatMember`) | The approval side. "Is this relationship currently active?" queries scan only `:APPROVAL` edges with positive top-layer `dim1`. |
| `:CONTAINMENT` | Comment → Post, Comment → Comment, ChatMessage → Chat, Comment → Chat, Comment → ChatMessage, Comment → Item | Content containment and reply structure. Queried for feed assembly and thread rendering. |
| `:TAGGING` | Post → Hashtag, Item → Hashtag | Tag associations. Queried by hashtag-centric browsing. |

All sub-category labels **replace** `:STRUCTURAL`, not add to it — a
relationship has exactly one label in Memgraph.

### What about actor edges?

Actor edges stay uniform at `:ACTOR`. The 2D tensor treats all
actor edges the same math-wise; splitting them by tuple would
multiply labels (User-Post, User-User, Collective-Post, ...) without
improving ranking efficiency — the ranking algorithm iterates over
actor edges regardless of tuple.

Endpoint node labels (`:User`, `:Post`, `:Chat`, etc.) already let
queries filter by meaning: `(u:User)-[:ACTOR]->(p:Post)` binds the
semantics without needing a `:USER_POST` label.

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

---

## What this doc is not

- **Not the conceptual model.** What edges are, their dimensions,
  directionality, append-only rule — see
  [graph-model.md](graph-model.md).
- **Not a storage tuning guide.** Operational concerns for
  performance live in a future storage/ops doc.
