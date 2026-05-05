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

## Caching

Looking up "who authored this?" by scanning all incoming layer 1
timestamps on every view would be expensive. The author ID should be
cached:

- **On the node itself** as a property (`author_id`) — keeps the info
  in the graph for traversal queries.
- **In Postgres metadata** (e.g. `posts.author_id`) — for display
  queries that don't need the graph.

Both are derived caches. The graph (earliest incoming layer 1) is the
source of truth; if the caches ever disagree with the graph, the graph
wins and the caches should be rebuilt.
