# Invitations

How a new actor (User or Company) joins the graph and gets their first
edges. Invitations are the onboarding mechanism that prevents new
actors from starting as isolated nodes with no path to any other part
of the graph.

## The two-edge invitation pattern

When an existing actor invites a new actor to the platform, **two
actor edges** are created:

```
Inviter   -[sentiment: +X, closeness: +Y]-> New Actor   (layer 1: "I invited them")
New Actor -[sentiment: +X, closeness: +Y]-> Inviter    (layer 1: "they invited me")
```

Both are normal actor edges (see
[graph-model.md §5](graph-model.md) for the edge catalog).
Neither is special-cased in the graph model.

## Why two edges

The new actor must have **at least one outgoing edge** from the moment
they join. Without it, their node is an island — zero hops to anywhere
in the graph, no feed to calculate. The inviter edge gives them a
starting position.

Both directions are needed because edges are strictly directional (see
[graph-model.md §1](graph-model.md)):

- The inviter's edge toward the new actor expresses the inviter's
  opinion (they liked this person enough to bring them in).
- The new actor's edge toward the inviter gives the new actor their
  first outbound connection, which the ranking algorithm can walk.

## Default values (OPEN DESIGN QUESTION)

The initial dimension values on the new actor's edge toward the
inviter are a design decision. Tradeoffs:

- **High positive defaults** (e.g. sentiment +0.8, closeness +0.7) —
  you presumably like the person who invited you. But this biases the
  new user's feed heavily toward one person's graph neighborhood for
  their first days on the platform.
- **Moderate positive defaults** (e.g. +0.3, +0.3) — softer start,
  but the new user's feed will be thin until they build more edges.
- **Neutral defaults** (0.0, 0.0) — no bias but also not much of a
  foothold. May leave the feed nearly empty.

The new actor can update this edge over time like any other; these are
only initial values. Choosing them well matters because they shape the
first week of the new user's experience.
