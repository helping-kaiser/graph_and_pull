# Collectives

A **Collective** is an actor node on the graph — any group of people
that needs a single graph identity to act through. The term spans
the full range from informal to formal: a household, a band, a
co-op, a studio, a partnership, an NGO, a company. Collectives are
**fully equivalent to Users as actors**: they can do everything
Users can do.

- Author posts, comments, items, chats.
- Create outbound actor edges (sentiment, closeness) toward any other
  node — including other collectives and other users.
- Own items (via ItemOwnership).
- Be followed, liked, disliked.
- Appear in feeds and be ranked like any other actor.

A collective having sentiment toward another collective is perfectly
normal. A collective having sentiment toward a user, or vice versa,
is also normal. There is no asymmetry at the graph level between
Collective and User actors.

## Economic role — no preferential treatment

No actor type receives preferential treatment in ad-revenue
distribution. Revenue follows graph topology, not actor type:
whichever nodes have the most economic weight in a "rich" part of
the graph — an influencer with massive reach, a bridging user that
connects otherwise-disconnected communities, a niche collective in
a dense neighborhood — receives a share proportional to that
weight. See the fair-economics principle in
[CLAUDE.md](../CLAUDE.md). The graph decides — actor type does not.

This applies symmetrically: commercial collectives that buy ads do
not receive preferential placement, and non-commercial collectives
(households, hobby groups, co-ops) are not penalized for not buying
ads.

## Collectives always have members

Every collective has, or at some point had, at least one
[CollectiveMember](#membership-collectivemember). A collective with
**zero active members** is a collective that has dissolved — the
history is preserved (members come and go via state transitions on
the structural edges, per
[graph-model.md §5](graph-model.md)), but no one currently acts on
the collective's behalf.

## Membership: CollectiveMember

A `CollectiveMember` is a junction node (see
[graph-model.md §2](graph-model.md)) connecting **Collective to
User or Collective**. A collective can be a member of another
collective — subsidiaries, holdings, partner firms, coalitions of
bands under a label, households as members of a co-op.
CollectiveMember is not restricted to human members.

It carries **role** and role-attached quantities as properties on
the node itself (not in edge dimensions):

- `role` — one of `founder`, `shareholder`, `worker`, `band member`,
  `subsidiary`, `partner`, `member`, etc. Categorical, defined per
  collective.
- `ownership_pct` — when the role implies a stake (e.g. shareholder).
- Additional properties as needed (voting weight, vesting schedule,
  etc.).

Role properties stay on the junction node rather than being encoded
in edge dimensions — see [graph-model.md §2](graph-model.md) for the
reasoning.

## Approval flow

CollectiveMember uses the **two-edge approval pattern** described in
[graph-model.md §5](graph-model.md):

1. Actor (User or Collective) creates an actor edge toward a new
   **CollectiveMember** node.
2. System creates `CollectiveMember -> Collective` (claim).
3. Required approving actors create actor edges toward the same
   CollectiveMember node. Approval policy depends on the target
   role — a new shareholder may require approval from existing
   founders and/or a threshold of current shareholders; adding a
   worker may be at founder discretion; adding a household member
   may need only the existing members' approval.
4. Once the collective's approval policy is satisfied, the system
   creates `Collective -> CollectiveMember` (approval).
5. Actor is an active member.

Multi-sig approval thresholds are expressed as "N actor edges from
specific roles required," with role-weighted voting derived from
the properties on the approving actors' own CollectiveMember nodes.

## Leaving / being removed

Departures follow the general state-transition rule for junction
approval pairs — new layers on the structural edges encode the
flip, and the relationship is active iff both top layers have
`dim1 > 0`. See [graph-model.md §5](graph-model.md) for the formal
rule, and [chats.md §8](chats.md) for the chat-side version of the
same mechanism applied to ChatMember. For CollectiveMember: a
member voluntarily leaving adds a negative layer to their actor
edge and the system cascades to `CollectiveMember -> Collective`;
removal by the collective (firing, dismissal, expulsion) adds a
negative layer to the approving actors' actor edges and, once the
policy threshold is met, the system cascades to
`Collective -> CollectiveMember`.
