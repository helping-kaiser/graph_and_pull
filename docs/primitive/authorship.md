# Authorship

Authorship in CoGra is a **derived fact**, not a stored field. The
author of a node is the actor whose incoming edge has the earliest
layer 1 timestamp. A node cannot exist without someone creating it, so
the very first edge ever created toward a node identifies the author.

**"Creator" is a synonym for "author"; "author" is canonical.**
Wherever a User or Collective is described as "creating" a node
— Item, Chat, Collective, Post, Comment, Proposal — the on-graph
fact is the same: they hold the earliest-layer-1 incoming edge,
the `:AUTHOR` label, and the rights and obligations that
authorship carries. "Founder" is *not* a synonym — it is the
CollectiveMember **role string** used inside a Collective (see
[graph-model.md §5 "Bootstrap"](graph-model.md#5-junction-node-flows)
and [collectives.md §1](../instances/collectives.md#1-creation)).

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

The graph is the source of truth. The `:AUTHOR` label is
derivable from the earliest-layer-1 rule; the Postgres
`author_id` is derivable from the graph. If either disagrees,
rebuild from the graph.

**Economics does not read the Postgres cache.** Ad-revenue
distribution, payouts, and any value-bearing computation walk
the graph directly. A stale `author_id` affects display ordering
at most; it never changes what the author is paid. The cache may
drift briefly without correctness risk.

**Rebuild trigger.** Stale cache entries self-heal
opportunistically: when a viewing user's feed-ranking pass
touches a post via a path that traverses its `:AUTHOR` edge, the
ranking step has the authoritative author UUID. If it disagrees
with `posts.author_id` (or the equivalent column), the API
enqueues a rebuild for that row. Rebuilds run out-of-band — the
read that detected the drift never blocks.

A user with no path to the post never triggers a rebuild, but
also has no display query depending on the column — so the
drift is invisible until someone with a path looks. Any
disagreement that matters to display gets corrected by someone
who would have seen the wrong value.
