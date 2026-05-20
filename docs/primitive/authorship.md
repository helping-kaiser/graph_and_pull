# Authorship

Authorship in CoGra is a **derived fact**, not a stored field. The
author of a node is the actor whose incoming edge has the earliest
layer 1 timestamp. A node cannot exist without someone creating it, so
the very first edge ever created toward a node identifies the author.

The dimension values on the author's edge are just normal opinion
values — the author's initial feelings about their own content
(typically high positive sentiment and relevance).

## Example

Jakob creates a post. His actor edge `Jakob → Post_X` is layer 1, with
the earliest timestamp of any incoming edge on Post_X. That makes Jakob
the author. Later, Alice likes the same post — her edge
`Alice → Post_X` also has a layer 1, but its timestamp is later than
Jakob's. The author is always the earliest.

## Collective-authored content

When a Collective is the author, the rule is unchanged: the
on-graph author is the actor whose incoming edge has the
earliest layer-1 timestamp, and that actor is the Collective
itself. The gesture that produced the edge is initiated
off-graph by an authorized CollectiveMember per the Collective's
social contract (see
[user.md §1](user.md#1-user-vs-collective) and
[collectives.md "Acting through the Collective"](../instances/collectives.md#2-acting-through-the-collective)),
but no acting-member identity is recorded on the authorship
edge. Querying "who authored this?" returns the Collective; the
member who initiated the gesture is not derivable from the
authored node.

## Graph-layer label — `:AUTHOR`

The authoring edge is the one actor edge that carries a sub-label
distinct from `:ACTOR` — `:AUTHOR` — per
[edges.md §3 "Sub-category labels"](edges.md#sub-category-labels).
The label is the system's mechanical implementation of the
"first outgoing edge" rule above: created at the same gesture as
the authoring edge, permanent across re-layerings, and
queryable in a single label scan.

`:AUTHOR` is also load-bearing for the feed-ranking author-hop
traversal rule ([feed-ranking.md §3.5](feed-ranking.md#35-traversal-restrictions)),
which terminates `:REFERENCES`-to-actor paths after exactly one
outgoing `:AUTHOR` hop.

Same tensor shape, same `[-1, +1]` range, same append-only layer
semantics as any actor edge — only the label differs.

## Caching

Display queries that don't touch the graph need a fast author
lookup, so `author_id` is cached on the Postgres `posts`,
`comments`, and `chat_messages` rows — see
[data-model.md](../implementation/data-model.md).

The graph (earliest incoming layer-1 edge) is the source of truth.
The `:AUTHOR` label is itself derivable from that rule; the
Postgres `author_id` is in turn derivable from the graph. If
either disagrees with the source of truth, rebuild from the graph.
