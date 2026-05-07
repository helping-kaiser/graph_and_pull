# Invitations

How a new actor (User or Collective) joins the graph and gets their first
edges. Invitations are the onboarding mechanism that prevents new
actors from starting as isolated nodes with no path to any other part
of the graph.

## The two-edge invitation pattern

When an existing actor invites a new actor to the platform, **two
actor edges** are created:

```
Inviter   -[sentiment: +0.5, interest: +0.5]-> New Actor   (layer 1: "I invited them")
New Actor -[sentiment: +0.5, interest: +0.5]-> Inviter    (layer 1: "they invited me")
```

Both are normal actor edges (see
[edges.md](edges.md) for the edge catalog).
Neither is special-cased in the graph model.

The `(+0.5, +0.5)` shown here is the **default** — see
"Default values and customization" below.

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

## Default values and customization

Both edges carry an initial `(dim1, dim2)` tensor — the layer 1
written when the invite is accepted. The defaults are
`(+0.5, +0.5)` on each direction.

**Both parties choose their own edge.** The inviter and the invitee
each pick the values on their own outgoing edge during the
invitation flow. The defaults exist as a fallback for users who
skip the choice; they are *not* the recommended values. The two
sides matter in different ways.

### Inviter side: shaping the new actor's reach

The inviter's edge `Inviter → New Actor` controls how the new
actor's eventual content traverses the inviter's network. Positive
values let their posts surface in the inviter's friends' feeds via
the path mechanics in
[feed-ranking.md §3](feed-ranking.md); weaker values are a softer
introduction. The inviter is signaling to their network how
strongly to weight this new person's voice. This influences the new
actor's **early popularity** in the graph.

### Invitee side: shaping their own first feed

The invitee's edge `New Actor → Inviter` is initially their *only*
outbound edge, so their entire first feed runs on traversal through
this single connection. Picking values deliberately matters more
once the invitee forms a **second** outbound edge — the two edges'
relative path products decide which neighborhood dominates the
feed.

**Worked example: invited by a friend with different interests.**
The invitee values the inviter as a person but does not share their
content tastes — different hobbies, different topics. Two natural
choices for the invitee's outbound edge:

- `(+1, +1)` — full strength on both axes. The inviter's
  neighborhood dominates the invitee's feed even after additional
  edges are formed, because the inviter-edge path products stay at
  full strength.
- `(+1, -1)` — high sentiment, negative interest. *"I love
  this person but their content is not what I want to see."* Once
  the invitee adds a second edge, e.g. `(+0.5, +0.5)` to a hashtag
  they care about, the second edge dominates the feed: the
  inviter-edge path products have positive sentiment chains and
  negative interest chains, which tend to cancel under the sum
  collapser in [feed-ranking.md §4.3](feed-ranking.md).

The broader lesson: invitation-edge values encode a relationship
stance the math respects until the edge gets a new layer. Picking
deliberately at invitation time avoids the trap of "I left it as
the default and now my feed is dominated by my inviter's network."

### About the defaults

`(+0.5, +0.5)` is moderate on both axes — enough to give the
invitee a walkable starting edge, enough that the inviter is making
a real endorsement, not so strong that uncustomized edges dominate
indefinitely. Frontends should make customization the primary path
during the invitation flow; the defaults only kick in if the user
explicitly skips the choice.

## Open invite links

Invitations are typically shared through messenger apps and other
social channels rather than over email — the inviter generates a
link and posts it where their community will see it. This means
the inviter does **not know in advance who will accept**.

The link is **time-gated and multi-use**, not single-use.
Influencers and public communities need to onboard many people
through the same shared link, so consuming it on first use would
defeat the mechanism. Different invitees produce different User
nodes; the link expires by time, not by use.

### Pre-committed inviter values

Because the invitee is unknown at link-generation time, the
inviter's outgoing edge values are **pre-committed when the link
is generated**, not chosen per invitee. Whoever accepts inherits
the values the inviter set on the link. The invitee still chooses
their own outgoing edge during registration, per "Default values
and customization" above.

### The bot-cluster trade-off

Open invite links create an attack surface: a bot cluster can
join through an influencer's public link and turn the influencer
into a **bridge node into the cluster**. The same mechanic that
gives the inviter reach also concentrates the cost of mis-vouching
onto them.

The system tolerates this deliberately. Open invite links are
necessary for high-reach onboarding — public communities and
influencers cannot effectively invite their audiences without them
— and the abuse case is self-correcting: the inviter's network
can sever the bridge through the cascading-severance mechanism
described in
[feed-ranking.md §3.5–§3.6](feed-ranking.md), at which point the
entire cluster reachable through that bridge is zero-jailed from
the network's perspective. Inviters who learn this lesson become
more selective with where they post their links.

The trade-off is intentional: restricting the mechanism would
deprive legitimate high-reach actors of a critical onboarding
tool; pushing the consequence onto the inviter aligns the
incentive with the actor most able to manage it.
