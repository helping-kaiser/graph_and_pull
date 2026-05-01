# Collectives

A **Collective** is an actor node on the graph — any group of people
that needs a single graph identity to act through. The term spans
the full range from informal to formal: a household, a band, a
co-op, a studio, a partnership, an NGO, a company. Collectives are
**fully equivalent to Users as actors**: they can do everything
Users can do.

- Author posts, comments, items, chats.
- Create outbound actor edges (sentiment, interest) toward any other
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
[CLAUDE.md](../../CLAUDE.md). The graph decides — actor type does not.

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
[graph-model.md §5](../primitive/graph-model.md)), but no one currently acts on
the collective's behalf.

## Membership: CollectiveMember

A `CollectiveMember` is a junction node (see
[graph-model.md §2](../primitive/graph-model.md)) connecting **Collective to
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
in edge dimensions — see [graph-model.md §2](../primitive/graph-model.md) for the
reasoning.

## Approval flow

CollectiveMember uses the **two-edge approval pattern** described in
[graph-model.md §5](../primitive/graph-model.md):

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

## Governance — the social contract

A collective's **social contract** is its set of governance rules:
which decisions need votes, who can vote on each, with what
weights, and at what threshold. Different collectives have very
different rules — a corporation's CEO can fire workers
unilaterally; a household requires consensus for everything; a
co-op uses 2/3 majorities for major decisions. The graph supports
all of these without any primitive changes.

### Per-decision-type instances

Every decision-type in a collective is a separate governance
instance per [governance.md §2](../primitive/governance.md). Each instance has
its own:

- **Subject** — what's being decided (a CollectiveMember junction
  for member changes; a Proposal node for property changes).
- **Eligibility** — who can vote (`role = CEO`,
  `role = board_member`, all members, members weighted by
  `ownership_pct`, …).
- **Weights** — how each voter's contribution is computed (uniform,
  role-based, or property-derived).
- **Threshold** — quorum and pass-threshold.

Instances coexist on the same Collective. Hiring a worker and
removing a board member can use entirely different rules; the
system routes each decision to its instance based on the subject
and the subject's role.

### No primitive defaults

Unlike Chats — which default to community-vote moderation because
that fits informal communities — Collectives must explicitly
define their rules at creation. Creating a Collective is the act
of writing its social contract. The example configurations below
are starting templates, not enforced defaults.

### Hierarchical authority is just a parameter choice

The "no admin veto" stance from chat governance is a chat-specific
default, not a primitive principle. A collective whose social
contract gives the CEO `weight = ∞` (or just `threshold = 1` with
`eligibility = role = CEO`) for the "fire worker" decision IS
expressing CEO-unilateral authority — and the graph supports it.
The primitive doesn't pick a power structure; the collective does.

### Example configurations

#### Corporate hierarchy

A small company with founders, a CEO, board members, and workers.

| Decision-type            | Eligibility                                            | Threshold |
|--------------------------|--------------------------------------------------------|-----------|
| Hire / fire worker       | `role = CEO`                                           | 1 vote    |
| Promote worker to senior | `role = CEO`                                           | 1 vote    |
| Add board member         | `role = founder`, weighted by `ownership_pct`          | > 50%     |
| Remove board member      | `role IN (founder, board_member)`, excluding subject   | ≥ 2/3     |
| Remove CEO               | `role = board_member`                                  | ≥ 2/3     |
| Change `ownership_pct`   | `role IN (founder, shareholder)`, weighted by stake    | ≥ 75%     |
| Change `Collective.name` | All active members                                     | > 50%     |

A worker is fired by a single CEO vote; a board member is removed
only by board supermajority; a CEO is removed only by the rest of
the board.

#### Household (5 people)

| Decision-type            | Eligibility                                | Threshold                                 |
|--------------------------|--------------------------------------------|-------------------------------------------|
| Add a new member         | All active members                         | 100% of cast, 100% quorum                 |
| Remove a member          | All members except subject                 | ≥ 90% of cast, 100% quorum of remaining   |
| Routine spending (if tracked) | All active members                    | > 50%, ≥ 60% quorum                       |

Everyone has equal voice; consensus dominates.

#### Worker co-op

All members equal stake; some routine decisions delegated to
officers.

| Decision-type            | Eligibility                | Threshold |
|--------------------------|----------------------------|-----------|
| Add a new member         | All active members         | ≥ 2/3     |
| Remove a member          | All members except subject | ≥ 2/3     |
| Routine operations       | `role = officer`           | > 50%     |
| Major policy change      | All active members         | ≥ 2/3     |
| Change capital structure | All active members         | ≥ 75%     |

### Where governance rules live

Each decision-type's parameters are stored as a structured property
on the Collective node (e.g.,
`Collective.governance_rules.remove_worker = { eligibility, weights, threshold }`).
Changes to any rule follow the standard Proposal pattern with that
rule's **own** configurable parameters. The bootstrap rules are set
at collective creation; everything afterward is governance of
governance.

## Leaving / being removed

Two paths out of an active membership:

- **Voluntary leave.** The member adds a new negative layer on
  their actor edge toward the CollectiveMember junction. The system
  cascades to `CollectiveMember -> Collective` with `dim1 < 0`.
  Self-determined; no governance vote.
- **Removal via governance instance.** The collective applies its
  "remove member" instance — eligibility, weights, and threshold
  per the social contract above. When that instance's threshold is
  crossed, the system adds a new layer on
  `Collective -> CollectiveMember` with `dim1 < 0`. The shape of
  "removal" varies enormously across collectives — a 1-of-1 CEO
  firing instance and a 2/3-of-board expulsion instance are both
  valid configurations.

In both cases the relationship is active iff both edges' top
layers have `dim1 > 0`, and the full history — including the
votes that drove a removal — stays visible, as everywhere else in
the graph.
