# Companies

A **Company** is an actor node on the graph — a business, organization,
band, solo artist profile, or any other collective or professional
entity. Companies are **fully equivalent to Users as actors**: they can
do everything Users can do.

- Author posts, comments, items, chats.
- Create outbound actor edges (sentiment, closeness) toward any other
  node — including other companies and other users.
- Own items (via ItemOwnership).
- Be followed, liked, disliked.
- Appear in feeds and be ranked like any other actor.

A company having sentiment toward another company is perfectly normal.
A company having sentiment toward a user, or vice versa, is also
normal. There is no asymmetry at the graph level between Company and
User actors.

## Economic role — no preferential treatment

Companies are often the entities that pay the most for ads. They are
**not** the entities that receive most of the ad revenue. Revenue
distribution follows graph topology, not actor type: whichever nodes
have the most economic weight in a "rich" part of the graph — an
influencer with massive reach, a bridging user that connects
otherwise-disconnected communities, a niche company in a dense
neighborhood — receives a share proportional to that weight. See the
fair-economics principle in [CLAUDE.md](../CLAUDE.md). The graph
decides — actor type does not.

## Companies always have members

Every company has, or at some point had, at least one
[CompanyMember](#membership-companymember). A company with **zero
active members** is a company that has gone out of business — the
history is preserved (members come and go via state transitions on
the structural edges, per
[edge-tensor-model.md §6](edge-tensor-model.md)), but no one
currently acts on the company's behalf.

## Membership: CompanyMember

A `CompanyMember` is a junction node (see
[edge-tensor-model.md §2](edge-tensor-model.md)) connecting **Company
to User or Company**. A company can be a member of another company —
subsidiaries, holdings, partner firms, coalitions of bands under a
label. CompanyMember is not restricted to human members.

It carries **role** and role-attached quantities as properties on the
node itself (not in edge dimensions):

- `role` — one of `founder`, `shareholder`, `worker`, `band member`,
  `subsidiary`, etc. Categorical.
- `ownership_pct` — when the role implies a stake (e.g. shareholder).
- Additional properties as needed (voting weight, vesting schedule,
  etc.).

Role properties stay on the junction node rather than being encoded in
edge dimensions — see
[edge-tensor-model.md §2](edge-tensor-model.md) for the reasoning.

## Approval flow

CompanyMember uses the **two-edge approval pattern** described in
[edge-tensor-model.md §6](edge-tensor-model.md):

1. Actor (User or Company) creates an actor edge toward a new
   **CompanyMember** node.
2. System creates `CompanyMember -> Company` (claim).
3. Required approving actors create actor edges toward the same
   CompanyMember node. Approval policy depends on the target role —
   a new shareholder may require approval from existing founders
   and/or a threshold of current shareholders; adding a worker may be
   at founder discretion.
4. Once the company's approval policy is satisfied, the system creates
   `Company -> CompanyMember` (approval).
5. Actor is an active member.

Multi-sig approval thresholds are expressed as "N actor edges from
specific roles required," with role-weighted voting derived from the
properties on the approving actors' own CompanyMember nodes.

## Leaving / being removed

Departures follow the general state-transition rule for junction
approval pairs — new layers on the structural edges encode the flip,
and the relationship is active iff both top layers have `dim1 > 0`.
See [edge-tensor-model.md §6](edge-tensor-model.md) for the formal
rule, and [chats.md §8](chats.md) for the chat-side version of the
same mechanism applied to ChatMember. For CompanyMember: a member
voluntarily leaving adds a negative layer to their actor edge and the
system cascades to `CompanyMember -> Company`; removal by the company
(firing, dismissal) adds a negative layer to the approving actors'
actor edges and, once the policy threshold is met, the system
cascades to `Company -> CompanyMember`.
