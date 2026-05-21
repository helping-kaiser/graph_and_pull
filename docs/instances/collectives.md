# Collectives

A **Collective** is an actor node on the graph — any group of
people that needs a single graph identity to act through. The
term spans the full range from informal to formal: a household,
a band, a co-op, a studio, a partnership, an NGO, a company.

On the outbound side a Collective looks like a
[User](../primitive/user.md): it authors content, creates actor
edges toward other nodes, owns items (via ItemOwnership), is
followed / liked / disliked, and appears in feeds and is ranked
like any other actor. The full outgoing-edge catalog is in
[edges.md §1 "Collective as actor"](../primitive/edges.md#collective-as-actor).
A Collective having sentiment toward another Collective, or
toward a User, or vice versa, is perfectly normal — there is no
asymmetry between Collective and User as edge endpoints.

What makes a Collective different from a User is the off-graph
side: a Collective has **no credentials of its own** and takes
no gestures by itself. Every action attributed to a Collective
is initiated by an authorized member — a User, or a sub-Collective
acting recursively through its own authorized members — per the
Collective's social contract. The graph records the action as the
Collective's; **no per-edge record of the acting member is kept**
(§2). The mechanism is in §2.

This means Collectives are **user-created nodes**: each Collective
begins with one founding User and a written social contract (see
§1).

This doc is the per-node catalog for two related nodes: the
**Collective** actor node and the **CollectiveMember** junction
node. Mechanics those topics depend on stay in their topical
docs — this doc links rather than duplicates.

---

## 1. Creation

A Collective is brought into existence by a single founding
gesture from exactly one **User**:

1. The founding User writes the Collective's social contract
   (§8) — at minimum its initial decision-type rules and its
   act-as rules (§2).
2. The system atomically creates the `:Collective` node and the
   founder's `CollectiveMember` junction.

Because the founder's CollectiveMember is the bootstrap — there
is no prior membership to vote on it — the
[two-edge approval pattern](../primitive/graph-model.md#5-junction-node-flows)
collapses to its 1-of-1 special case: the founder's `User → CollectiveMember`
**Shape A self-claim** is the only required vote, and the
system writes both structural edges (claim and approval) plus
the `CollectiveMember → User` `:BEARER` identity edge atomically
alongside it. This is the same bootstrap pattern used for the
author's `ItemOwnership` in
[items.md §1](items.md#1-creation) and for the founder of a
Chat in [chats.md §2.1](chats.md#21-chat). See §7 for the
regular case where existing CollectiveMembers cast Shape B
approver votes.

The founder's role on their CollectiveMember junction is
whatever the social contract names for the inaugural role
(`founder`, `owner`, `partner`, …). There is no separate
"author" role and no uniqueness constraint on the inaugural
role: **additional founders are added afterward through the
regular CollectiveMember addition flow**, and their `founder`
(or equivalent) role carries the same weight as the bootstrap
founder's. The author-User is identifiable on the graph as the
earliest layer-1 timestamp among the Collective's incoming
CollectiveMember-claim edges — the same earliest-incoming-edge
rule that derives authorship for any other node (see
[authorship.md](../primitive/authorship.md) and §6).

### Sub-Collectives

A Collective creating another Collective follows the same
pattern: the founding Collective acts through one of its
authorized members (a governance-act per §2), producing the
bootstrap gesture, and the new sub-Collective's first
CollectiveMember junction is `parent Collective → new sub-Collective`.
The User who originated the gesture remains identifiable through
the parent Collective's own CollectiveMember chain, but is not
directly recorded on the sub-Collective's graph structure.

---

## 2. Acting through the Collective

A Collective produces actor edges, but has no credentials and
takes no gestures by itself. Every edge attributed to a
Collective is **initiated by an authorized member** — a User, or
a sub-Collective acting through its own authorized members. At
the graph layer the Collective is the source of the edge: there
is no `acting_user` dimension on the edge, no separate junction
recording which member produced the gesture, no on-graph trace
that links the edge back to its initiator.

**The lack of per-edge acting-member attribution is
deliberate.** Once a member is authorized to act for the
Collective, the Collective IS the actor for the graph's
purposes — accountability for a member's gestures lives in the
social contract (which decides who can authorize what), not in
per-edge attribution. A Collective whose authorized members
produce harmful gestures is accountable as a Collective;
whether and how it then holds individual members accountable
internally is itself a matter for its social contract.

### Content-acts vs governance-acts

Two coarse classes of gestures, with different defaults:

**Content-acts** — authoring [Posts](post.md) and
[Comments](comment.md), and creating sentiment/relevance actor
edges toward other nodes (likes, dislikes, follows, interest).
**Default: any active CollectiveMember may produce a content-act
on behalf of the Collective.** A Collective that wants to lock
content-acts down (e.g. "only the press officer posts") declares
an explicit act-as rule that overrides the default; otherwise
the any-active-member default applies.

**Governance-acts** — authoring [Proposals](proposal.md) on
behalf of the Collective, casting votes in governance instances
the Collective is eligible in, creating or approving
[ItemOwnership](items.md) junctions, and creating or approving
[CollectiveMember](#3-graph-side-properties) junctions on
other Collectives. **Default: no member can produce a
governance-act on behalf of the Collective.** An explicit act-as
rule in the social contract is required. Governance-acts have
external consequences (they bind the Collective to votes, to
owned items, to memberships in other Collectives); defaulting
them off forces the Collective to declare in writing who can
carry them out.

The two defaults reflect the same principle from the rest of the
governance primitive ([governance.md](../primitive/governance.md)):
routine, reversible-by-the-actor gestures can be permissive;
consequential, binding gestures require explicit eligibility.

**Invariant:** Collective content-acts default permissive (any
active member may produce them); governance-acts default deny (no
member can produce them without an explicit act-as rule in the
social contract). The asymmetry reflects reversibility: a stray
Post is reversible by a counter-post, but a stray Proposal vote or
Item transfer binds the Collective externally.

### Routing

When a member attempts to act-as a Collective C with a gesture
that would produce edge E:

1. The system classifies E as a content-act or governance-act.
2. The system looks up the act-as rule in C's social contract.
   If an explicit rule exists for E (by class or by specific
   edge type), eligibility, weight, and threshold come from
   that rule; otherwise the default for the class applies
   (allow for content-acts, deny for governance-acts).
3. If the rule's threshold is `1`, the gesture immediately
   produces C's actor edge.
4. If the threshold is greater than `1`, the gesture creates a
   pending state and waits for the required co-signatures from
   other eligible members — the same shape as a multi-sig
   junction approval per
   [graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows).
   Only when the threshold is satisfied does the system produce
   C's outgoing edge.

If the acting "member" is itself a sub-Collective, its own
social contract is consulted recursively before the parent
Collective's edge is produced — the sub-Collective must
authorize the gesture on its end before the parent Collective's
on-behalf-of step is reached.

---

## 3. Graph-side properties

### Collective

A Collective node carries only what the graph needs to traverse,
filter, rank, and route governance. Display content (profile
text, avatar, website) lives in Postgres (§4).

- **`name`** — the handle used for mentions and lookups,
  analogous to `User.username`. Layered per
  [layers.md §3](../primitive/layers.md#3-layers-on-nodes), so
  rename history is preserved. UNIQUE per instance.
- **`moderation_status`** — `'normal'` / `'sensitive'` /
  `'illegal'`, default `'normal'`, layered. Universal across all
  user-input-bearing nodes; per-node mechanics — set by a passing
  `'sensitive'` Proposal, auto-flipped to `'illegal'` by the
  redaction cascade — are described in
  [nodes.md "Universal: moderation_status"](../primitive/nodes.md#universal-moderation_status)
  and §9 below.
- **`governance_rules.*`** — structured properties holding the
  social contract: one entry per decision-type instance and one
  per act-as rule (e.g.
  `governance_rules.remove_worker = { eligibility, weights, threshold }`,
  `governance_rules.act_as_transfer_item = { … }`). **Each rule
  is itself a layered authored property** per
  [layers.md §3](../primitive/layers.md#3-layers-on-nodes), and
  changes go through the standard Proposal pattern with that
  rule's own parameters — governance of governance. See §8.

Concrete property types and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

### CollectiveMember

A `CollectiveMember` is a junction node (see
[graph-model.md §2](../primitive/graph-model.md#2-node-categories))
connecting **Collective to User or Collective**. A Collective
can be a member of another Collective — subsidiaries, holdings,
partner firms, coalitions of bands under a label, households as
members of a co-op. CollectiveMember is not restricted to human
members.

Per [user.md §3](../primitive/user.md#3-graph-side-properties),
**every authored property is layered**. CollectiveMember
properties accordingly accumulate layers on change; the
appropriate decision-type instance in the Collective's social
contract governs each change (promotions, equity adjustments,
weight changes — see §8).

- **`role`** — categorical: `'founder'`, `'shareholder'`,
  `'worker'`, `'band member'`, `'subsidiary'`, `'partner'`,
  `'member'`, etc. Open-ended per the social contract; the role
  vocabulary is **Collective-specific**, not a global enum.
  Layered.
- **`ownership_pct`** — when the role implies a stake (e.g.
  shareholder). Layered when present.
- **`voting_weight`** — optional direct weight override for
  Collectives whose weight is not tied to equity (one-member-one-vote
  with role-based multipliers, per-member negotiated weight,
  etc.). Layered when present. See
  [governance.md §2.3](../primitive/governance.md#23-weight-function).

Role properties stay on the junction node rather than being
encoded in edge dimensions — see
[graph-model.md §2](../primitive/graph-model.md#2-node-categories)
for the reasoning. Concrete property types and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 4. Postgres-side content

### Collective

A Collective's display content lives in Postgres, linked to the
graph node by UUID. Edits are append-only per
[layers.md §4](../primitive/layers.md#4-layers-on-postgres-side-display-content):
a new version row, no overwrite.

- **`name`** — required; the handle used for mentions and
  lookups, analogous to `users.username`. UNIQUE per instance.
  Stored on the `collectives` row alongside the graph-side
  `name` of the same value.
- **`display_name`** — required; the human-readable label
  surfaced in feeds and profile views.
- **`description`** — optional body text describing what the
  Collective is and what it does.
- **`avatar_id`** — optional 1:1 FK to `media_attachments`,
  analogous to `users.avatar_id`. See
  [data-model.md "Why parents point at attachments"](../implementation/data-model.md#why-parents-point-at-attachments).
- **`website_url`** — optional external link.

Concrete schema lives in
[data-model.md](../implementation/data-model.md).

### CollectiveMember

None. CollectiveMember is a pure graph-side junction node — no
Postgres-side display content, no author-bearing row.

---

## 5. Edges

This doc covers two nodes: the **Collective** actor node and the
**CollectiveMember** junction. Each gets its own subsection.
Dimension labels, sub-category labels, and traversal semantics
are not duplicated here — see
[edges.md](../primitive/edges.md).

Every outgoing edge from a Collective is **initiated through an
authorized member** per §2; the graph layer records the edge as
the Collective's own with no per-edge record of which member
produced the gesture.

### 5.1 Collective

#### As source (outgoing)

A Collective is an actor. Its outgoing **actor edges** are the
full row in
[edges.md §1 "Collective as actor"](../primitive/edges.md#collective-as-actor)
— Collective → User, Collective → Post, Collective → Item,
Collective → Proposal, etc. The `(dim1, dim2)` values are set by
the acting member under the act-as rule routed by §2.

It carries one outgoing **structural** edge type, system-created:

- **`Collective → CollectiveMember` (`:APPROVAL`)** — the
  approval side of the two-edge approval pattern. Created once
  the collective's approval policy for the new member's role is
  satisfied (§7). State transitions — member removal per §9 —
  append additional `dim1 < 0` layers per
  [graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows).
  See
  [edges.md §2 "Approval completion"](../primitive/edges.md#approval-completion).

#### As target (incoming)

A Collective receives:

- **Actor edges** from Users and Collectives per
  [edges.md §1](../primitive/edges.md#1-actor-edges) — sentiment
  toward the collective and interest in its output, used by
  [feed-ranking](../primitive/feed-ranking.md) and the follow /
  interest surface.
- **`CollectiveMember → Collective` (`:CLAIM`)** — the claim
  side of the two-edge approval pattern, paired with the
  outgoing `Collective → CollectiveMember` above. See
  [edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging).
- **`ChatMember / CollectiveMember / ItemOwnership → Collective`
  (`:BEARER`)** — identity-binding edges from junction nodes the
  Collective bears (chat memberships, sub-collective memberships,
  item ownerships). See
  [edges.md §2 "Bearer binding"](../primitive/edges.md#bearer-binding).
- **`ChatMessage / Post / Comment → Collective` (`:REFERENCES`)**
  when a content node mentions or embeds the Collective. See
  [edges.md §2 "Reference"](../primitive/edges.md#reference).
- **`Proposal → Collective` (`:TARGETS`)** when a Proposal
  targets a property on the Collective — `name`,
  `moderation_status`, or any `governance_rules.*` parameter
  (§8). See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting).

### 5.2 CollectiveMember

#### As source (outgoing)

A CollectiveMember is a junction, not an actor. It carries one
claim edge, one bearer-binding edge, plus the Shape B vote edges
its bearer casts as a collective-eligible voter:

- **`CollectiveMember → Collective` (`:CLAIM`)** — the claim
  side of the two-edge approval pattern, closed by the
  collective's `Collective → CollectiveMember` approval edge
  (§5.1) once the collective's approval policy is satisfied
  (§7). See
  [edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging).
- **`CollectiveMember → User/Collective` (`:BEARER`)** —
  identity-binding edge written at junction creation, pointing
  at the actor (User or sub-Collective) the membership
  represents. Never re-pointed; the Shape A self-claim that
  activates the membership must originate from this actor (§7).
  See
  [edges.md §2 "Bearer binding"](../primitive/edges.md#bearer-binding).
- **`CollectiveMember → CollectiveMember` (Shape B vote)** —
  approver / removal vote on another CollectiveMember of the
  same Collective. `dim1 > 0` admits or affirms; a later
  `dim1 < 0` layer on the same edge votes for removal. See
  [edges.md §2 "Voting (Shape B)"](../primitive/edges.md#voting-shape-b)
  and
  [governance.md §3](../primitive/governance.md#3-the-two-vote-shapes).
- **`CollectiveMember → Proposal` (Shape B vote)** —
  collective-eligible vote on a Proposal targeting a collective
  property, role change, or any decision-type instance defined
  in the social contract (§8). `dim1` carries vote direction.

#### As target (incoming)

A CollectiveMember receives:

- **Actor edges** from Users and Collectives per
  [edges.md §1](../primitive/edges.md#1-actor-edges). For the
  bearer themselves, the `User → CollectiveMember` (or
  `Collective → CollectiveMember` when a Collective is the
  bearer via sub-Collective membership) edge is the **Shape A
  self-claim** that initiates the membership (§7). For other
  actors, these edges are personal sentiment about that
  membership — they do not drive the approval vote, which uses
  Shape B (above).
- **`CollectiveMember → CollectiveMember` (Shape B vote)** —
  incoming approver / removal votes from other active
  CollectiveMembers of the same Collective (§7, §9).
- **`Collective → CollectiveMember` (`:APPROVAL`)** — the
  approval side of the two-edge pattern, paired with the
  outgoing `CollectiveMember → Collective` claim above. State
  transitions — removal per §9 — append `dim1 < 0` layers on
  this edge per
  [graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows).
- **`ChatMessage / Post / Comment → CollectiveMember`
  (`:REFERENCES`)** when a content node embeds the membership
  (e.g. spotlighting a co-op steward). See
  [edges.md §2 "Reference"](../primitive/edges.md#reference).
- **`Proposal → CollectiveMember` (`:TARGETS`)** when a
  Proposal targets a property on the CollectiveMember — `role`
  changes (hire / fire / promote per the social contract),
  `ownership_pct`, etc.

---

## 6. Authorship

### Collective

A Collective is the on-graph author of any node whose earliest
incoming actor edge originates from it — the same
earliest-incoming-edge rule that derives authorship for every
node type ([authorship.md](../primitive/authorship.md)). The
gesture that produced the edge is initiated off-graph by an
authorized CollectiveMember per the Collective's social contract
(§§2, 8), but **no acting-member identity is recorded on the
authorship edge or anywhere else on the authored node.**
Querying "who authored this?" returns the Collective; the member
who initiated the gesture is not derivable from the authored
node. This matches the framing in
[authorship.md "Collective-authored content"](../primitive/authorship.md#collective-authored-content)
and is the same omission described in §2 as a deliberate
non-feature.

A Collective is itself authored — its **author** is the User
identifiable as the earliest layer-1 timestamp among the
Collective's incoming CollectiveMember-claim edges (§1). The
author-User is a graph-derivable identity, not a stored
authorship pointer; the role they hold on their CollectiveMember
junction is whatever the social contract named for the inaugural
role (commonly `founder`).

### CollectiveMember

CollectiveMember is a junction node and has no authorship in the
[authorship.md](../primitive/authorship.md) sense — it
represents a membership relationship, not an authored piece of
content. Its bearer (the actor the `:BEARER` edge points at) is
the identity it represents; the actor whose gesture produced it
is whichever party initiated the two-edge approval, but neither
is an "author" in the graph's authorship rule.

---

## 7. Approval flow

CollectiveMember uses the **two-edge approval pattern** described
in
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows):

1. The **would-be member** (User or Collective) writes a
   `User/Collective → new CollectiveMember` actor edge — their
   **Shape A self-claim** to the membership. The system creates
   the `CollectiveMember → Collective` claim edge and the
   `CollectiveMember → User/Collective` `:BEARER` identity edge
   in response. (Approver-initiated flows mirror invite-only:
   the approver creates the junction and `:BEARER` first; the
   would-be member self-claims later.)
2. **Required approvers** — existing CollectiveMembers eligible
   under the social contract for the target role — each cast a
   **Shape B vote** from their own existing CollectiveMember to
   the new one (`CollectiveMember_approver → CollectiveMember_new`,
   `dim1 > 0`).
3. Once the social contract's threshold is crossed, the system
   creates the `Collective → CollectiveMember` approval edge.
   The membership is active.

Approval policy depends on the target role — a new shareholder
may require approval from existing founders and/or a threshold
of current shareholders; adding a worker may be at founder
discretion; adding a household member may need consensus.
Multi-sig thresholds are expressed as "N Shape B votes from
specific roles required," with role-weighted voting derived from
the properties on the approving CollectiveMembers (per
[governance.md §2.3](../primitive/governance.md#23-weight-function)).

The bootstrap case — the founder's CollectiveMember at
Collective creation — collapses this to its 1-of-1 form: only
the Shape A self-claim is required, no Shape B approver votes
exist because no prior CollectiveMembers exist. See §1.

---

## 8. Governance — the social contract

A collective's **social contract** is its set of governance rules:
which decisions need votes, who can vote on each, with what
weights, and at what threshold. Different collectives have very
different rules — a corporation's CEO can fire workers
unilaterally; a household requires consensus for everything; a
co-op uses 2/3 majorities for major decisions. The graph supports
all of these without any primitive changes.

### Per-decision-type instances

Every decision-type in a collective is a separate governance
instance per
[governance.md §2](../primitive/governance.md#2-the-five-components).
Each instance has its own:

- **Subject** — what's being decided (a CollectiveMember junction
  for member changes; a Proposal node for property changes).
- **Eligibility** — who can vote (`role = CEO`,
  `role = board_member`, all members, members weighted by
  `ownership_pct`, …). Per
  [governance.md §2.2](../primitive/governance.md#22-eligibility),
  eligibility is evaluated **at tally time**: a vote from
  someone who becomes ineligible afterward drops out; a vote
  from someone who becomes eligible later (e.g. a newly-approved
  member) counts once their status flips.
- **Weights** — how each voter's contribution is computed (uniform,
  role-based, or property-derived).
- **Threshold** — quorum and pass-threshold.

Instances coexist on the same Collective. Hiring a worker and
removing a board member can use entirely different rules; the
system routes each decision to its instance based on the subject
and the subject's role.

### Act-as rules

Act-as rules are a second family of rules in the social
contract, sitting alongside the decision-type instances above.
They govern the on-behalf-of mechanism described in §2: which
members can produce which classes of gestures as the Collective.

An act-as rule has the same parameter shape as a decision-type
instance — eligibility, weights, threshold — but its outcome is
the production of the Collective's outgoing edge itself, not a
state transition on a separate subject. A single-signer rule
(threshold `1`) is the common case; a multi-sig rule
(threshold > `1`) delays the gesture until co-signers satisfy
the threshold, analogous to a multi-sig junction approval.

The defaults from §2 apply when no explicit rule covers a
gesture: content-acts default to any-active-member at threshold
`1`; governance-acts default to deny. Explicit rules override
these — content-acts can be locked down, governance-acts can be
opened up. The example configurations below include illustrative
act-as rules alongside the existing decision-type rules.

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

The roles used in the configurations below (`CEO`, `founder`,
`board_member`, `shareholder`, `worker`, etc.) are
**collective-specific** — each collective's social contract
defines its own role vocabulary. Roles are not a global enum;
the primitive only requires that a collective name them
consistently for its own eligibility/weight rules.

#### Corporate hierarchy

A small company with founders, a CEO, board members, and workers.

| Decision-type / Act-as rule        | Eligibility                                            | Threshold |
|------------------------------------|--------------------------------------------------------|-----------|
| Hire / fire worker                 | `role = CEO`                                           | 1 vote    |
| Promote worker to senior           | `role = CEO`                                           | 1 vote    |
| Add board member                   | `role = founder`, weighted by `ownership_pct`          | > 50%     |
| Remove board member                | `role IN (founder, board_member)`, excluding subject   | ≥ 2/3     |
| Remove CEO                         | `role = board_member`                                  | ≥ 2/3     |
| Change `ownership_pct`             | `role IN (founder, shareholder)`, weighted by stake    | ≥ 75%     |
| Change `Collective.name`           | All active members                                     | > 50%     |
| Act-as: post / comment             | `role = press_officer` *(override of the any-member default)*   | 1 signer  |
| Act-as: author external Proposal   | `role = CEO`                                           | 1 signer  |
| Act-as: cast vote in external Proposal | `role = CEO` or `role = board_member`              | 1 signer  |
| Act-as: transfer Item (acquire / release) | `role IN (founder, board_member)`, weighted by stake | ≥ 50% signers |

A worker is fired by a single CEO vote; a board member is removed
only by board supermajority; a CEO is removed only by the rest of
the board. Routine PR posting is delegated to a single press
officer (locking down the otherwise any-member default for
content-acts), while consequential moves — proposing, voting,
and transferring company assets — are routed to leadership and
the board.

#### Household (5 people)

| Decision-type / Act-as rule    | Eligibility                                | Threshold                                 |
|--------------------------------|--------------------------------------------|-------------------------------------------|
| Add a new member               | All active members                         | 100% of cast, 100% quorum                 |
| Remove a member                | All members except subject                 | ≥ 90% of cast, 100% quorum of remaining   |
| Routine spending (if tracked)  | All active members                         | > 50%, ≥ 60% quorum                       |
| Act-as: vote in HOA Proposal   | All active members                         | > 50% signers                              |
| Act-as: acquire shared Item    | All active members                         | > 50% signers                              |

Everyone has equal voice; consensus dominates. Content-acts
(posting to the household feed, reacting on shared content) are
left at the any-member default — no override.

#### Worker co-op

All members equal stake; some routine decisions delegated to
officers.

| Decision-type / Act-as rule         | Eligibility                | Threshold       |
|-------------------------------------|----------------------------|-----------------|
| Add a new member                    | All active members         | ≥ 2/3           |
| Remove a member                     | All members except subject | ≥ 2/3           |
| Routine operations                  | `role = officer`           | > 50%           |
| Major policy change                 | All active members         | ≥ 2/3           |
| Change capital structure            | All active members         | ≥ 75%           |
| Act-as: vote in federation Proposal | All active members         | > 50% signers   |
| Act-as: transfer co-op-held Item    | All active members         | ≥ 2/3 signers   |

### Where governance rules live

Each decision-type's and act-as rule's parameters are stored as a
structured property on the Collective node (e.g.,
`Collective.governance_rules.remove_worker = { eligibility, weights, threshold }`,
`Collective.governance_rules.act_as_transfer_item = { eligibility, weights, threshold }`).
**Each such property layers per
[layers.md §3](../primitive/layers.md#3-layers-on-nodes); changes
to any rule follow the standard Proposal pattern with that
rule's own configurable parameters.** The bootstrap rules are set
at collective creation; everything afterward is governance of
governance.

---

## 9. Lifecycle

### Collective

Collective nodes are **never deleted**. Per
[layers.md §5](../primitive/layers.md#5-deletion-policy), the
only permitted "removal" is in-place layer redaction on graph
properties or a tombstone version row on Postgres-side display
content; both preserve a visible record that the change occurred.

**Invariant — always had a member:** Every Collective has, or at
some point had, at least one active CollectiveMember. The
founding gesture (§1) creates the founder's CollectiveMember
atomically with the Collective node, so a Collective cannot
come into existence empty. A Collective with **zero active
members** is one that has **dissolved** — every member has left
or been removed and no one currently acts on the Collective's
behalf. The history is preserved: past members come and go via
state transitions on the structural edges per
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows),
and the chain of CollectiveMembers remains visible on the
graph. A dissolved Collective node persists; only its acting
capacity is gone.

Two redaction triggers apply to a Collective today:

- **Moderation: `'sensitive'` classification.** A passing
  `'sensitive'` Proposal flips the top layer of `moderation_status`
  to `'sensitive'`. No redaction; display content stays. Each
  viewing user's `content_filtering_severity_level` (see
  [data-model.md](../implementation/data-model.md) "User
  preferences") decides how aggressively the frontend filters
  the Collective. Reversible by a counter-Proposal back to
  `'normal'`. See
  [moderation.md §1](moderation.md#1-the-two-classification-paths).
- **Moderation: `'illegal'` classification.** A passing
  `'illegal'` Proposal targets one of the Collective's
  user-input fields — `name`, the Postgres-side `display_name` /
  `description` / `website_url`, the `avatar`, or the literal
  `'full'` shorthand per the per-node field list in
  [moderation.md §5](moderation.md#5-scope) — and fires the
  redaction cascade per
  [moderation.md §1](moderation.md#1-the-two-classification-paths):
  the affected graph-property layer or Postgres row is replaced
  with a redaction marker / tombstone, the redacted originals
  are written to the
  [retention archive](../primitive/retention-archive.md) under
  per-row legal hold, and the Collective node's
  `moderation_status` is auto-flipped to `'illegal'`. The
  cascade does **not** propagate to descendants — a Collective
  classified illegal does not redact the Posts and Comments it
  has authored, its CollectiveMembers, or items it owns. Each
  requires its own classification. `governance_rules.*` are
  in scope for `'illegal'`-targeted redaction in principle, but
  this is an unusual case; the typical redaction targets the
  identity fields.

A redacted Collective is an anonymized but still-graph-resident
actor, not a removed one. The Collective's UUID is stable
across every redaction. CollectiveMember chains, authored
content's authorship edges, owned items' ItemOwnership chains,
and incoming references all remain valid pointers.

### CollectiveMember

CollectiveMember nodes are also **never deleted**. Membership
changes follow the primitive — see
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows)
("Revocation and state transitions"):

- **Voluntary leave.** The bearer adds a negative-`dim1` layer
  on their own Shape A self-claim
  (`User/Collective → CollectiveMember`). The system appends a
  `dim1 < 0` layer on the claim-side structural edge. The
  CollectiveMember junction stays on the graph; the relationship
  is revoked.
- **Removal.** Eligible voters per the social contract's removal
  instance lay `dim1 < 0` layers on their existing
  `CollectiveMember_voter → CollectiveMember_target` Shape B
  edges (the same edges that previously approved the membership,
  if they voted in the original approval). When the threshold is
  crossed the system appends a `dim1 < 0` layer on the
  approval-side `Collective → CollectiveMember` edge.

The shape of "removal" varies enormously across collectives — a
1-of-1 CEO firing instance and a 2/3-of-board expulsion instance
are both valid configurations parameterized in the social
contract (§8). The Shape B edge mechanics are uniform; only the
threshold differs.

---

## 10. Economic role — no preferential treatment

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

---

## What this doc is not

- **Not the edge catalog.** Per-target-type edges with dimension
  labels live in [edges.md](../primitive/edges.md).
- **Not the governance primitive.** The five components, two
  vote shapes, tally-time eligibility rule, and weight-at-tally-time
  rule live in [governance.md](../primitive/governance.md).
- **Not the moderation primitive.** The Proposal mechanism, the
  mod gate, eligibility, thresholds, and the redaction cascade
  live in [moderation.md](moderation.md).
- **Not the deletion mechanism.** The redaction primitive lives
  in [layers.md §5](../primitive/layers.md#5-deletion-policy);
  the per-row legal hold and archive disposition live in
  [retention-archive.md](../primitive/retention-archive.md).
- **Not the Memgraph or Postgres schema.** Concrete property
  types, columns, indexes, and the `collectives` row shape live
  in
  [graph-data-model.md](../implementation/graph-data-model.md)
  and [data-model.md](../implementation/data-model.md).
- **Not the auth path for member gestures.** How a User's session
  authenticates a request that produces a Collective edge lives
  in [auth.md](../implementation/auth.md);
  [user.md §1](../primitive/user.md#1-user-vs-collective) is the
  short version.
