# Invitations

How a new User joins the graph and gets their first edges.
Invitations are the User onboarding mechanism that prevents new
Users from starting as isolated nodes with no path to the rest
of the graph.

Collectives are not invited — they come into existence through a
different mechanism. See
[collectives.md](../instances/collectives.md).

## The two-edge invitation pattern

When an existing actor invites a new actor to the platform, **two
actor edges** are created:

```
Inviter   -[sentiment: +0.5, interest: +0.5]-> New Actor   (layer 1: "I invited them")
New Actor -[sentiment: +0.5, interest: +0.5]-> Inviter    (layer 1: "they invited me")
```

Both are normal actor edges (see [edges.md](edges.md)). Neither
is special-cased in the graph model.

The `(+0.5, +0.5)` shown here is the **default** — see "Default
values and customization" below.

## Why two edges

The new actor must have **at least one outgoing edge** from the moment
they join. Without it, their node is an island — zero hops to anywhere
in the graph, no feed to calculate. The inviter edge gives them a
starting position.

Both directions are needed because edges are strictly directional (see
[graph-model.md §1](graph-model.md#1-core-principles)):

- The inviter's edge toward the new actor expresses the inviter's
  opinion (they liked this person enough to bring them in).
- The new actor's edge toward the inviter gives the new actor their
  first outbound connection, which the ranking algorithm can walk.

## Default values and customization

Both edges carry an initial `(dim1, dim2)` tensor — layer 1,
written when the invite is accepted. Defaults are `(+0.5, +0.5)`
on each direction.

**Both parties choose their own edge** during the invitation
flow. The defaults are a fallback for users who skip the choice,
*not* the recommended values. The two sides matter differently.

### Inviter side: shaping the new actor's reach

The inviter's edge `Inviter → New Actor` controls how the new
actor's eventual content traverses the inviter's network. Positive
values let their posts surface in the inviter's friends' feeds via
the path mechanics in
[feed-ranking.md §3](feed-ranking.md#3-per-edge-composition-along-a-path); weaker values are a softer
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
  collapser in [feed-ranking.md §4.3](feed-ranking.md#43-tuple-collapse-to-scalar).

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

## Link modes: single-use and multi-use

When generating an invite link, the inviter picks **single-use or
multi-use**. Both modes are time-gated.

- **Single-use.** Consumed on the first accepted registration.
  Best for targeted invites — sending a specific link to a
  specific person. Even if the link leaks, the worst case is one
  accidental join, with no broader bot-cluster exposure.
- **Multi-use.** Many invitees can register through the same link
  until its timer expires. Different invitees produce different
  User nodes. Influencers and public communities need this mode
  to onboard their audience through a single shared link —
  typically posted over messenger or social channels, where the
  inviter does **not know in advance who will accept**.

### Pre-committed inviter values

The inviter's outgoing edge values are **pre-committed when the
link is generated**, not per invitee — same mechanic for both
modes. Whoever accepts inherits those values. The invitee still
chooses their own outgoing edge at registration.

### Revocation and abandonment

The inviter can revoke a link explicitly at any time; otherwise it
expires when its timer runs out. A link that no one accepts simply
expires — no User node, no edges, no record beyond the link itself.
Implementation specifics — email verification, pending-registration
handling, and the atomic edge-creation step on verification — live
in [auth.md](../implementation/auth.md).

### The bot-cluster trade-off

Multi-use links shared publicly create an attack surface: a bot
cluster joining through an influencer's link turns the
influencer into a **bridge node into the cluster**. The same
mechanic that gives the inviter reach concentrates the cost of
mis-vouching onto them. (Single-use links sidestep this by
construction — at most one accidental join.)

The system tolerates this for the multi-use case because public
multi-use links are necessary for high-reach onboarding —
communities and influencers can't onboard their audiences
otherwise — and the abuse is self-correcting: the inviter's
network can sever the bridge through cascading severance
([feed-ranking.md §3.6–§3.7](feed-ranking.md#36-bot-resistance-via-the-0-0-severance-edge)),
at which point the entire cluster reachable through that bridge
is zero-jailed. Inviters learn to be more selective with where
they post their links.

The trade-off is intentional: restricting the mechanism would
deprive legitimate high-reach actors of a critical onboarding
tool; pushing the consequence onto the inviter aligns the
incentive with the actor most able to manage it.
