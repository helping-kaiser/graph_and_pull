# Invitations

How a new actor (User or Collective) joins the graph and gets their first
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
[edges.md](edges.md) for the edge catalog).
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

## Default values

The initial dimension values on the new actor's edge toward the
inviter are an open design decision — they shape the first week of
the new user's experience. The question and the options considered
are tracked in [open-questions.md Q6](open-questions.md).
