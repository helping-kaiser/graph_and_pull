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
(§2).

Collectives are **user-created nodes**: each begins with one
founding User and a written social contract (§1).

This doc is the per-node catalog for two related nodes — the
**Collective** actor node and the **CollectiveMember** junction
node. Topical mechanics live in their topical docs; this doc
links rather than duplicates.

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
founder's. The author-User is graph-derivable — see §6.

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

A Collective produces actor edges but has no credentials and
takes no gestures by itself. Every edge attributed to a
Collective is **initiated by an authorized member** — a User, or
a sub-Collective acting through its own authorized members. At
the graph layer the Collective is the source of the edge: no
`acting_user` dimension, no separate junction recording the
member, no on-graph trace back to the initiator.

**The lack of per-edge acting-member attribution is
deliberate.** Once a member is authorized to act for the
Collective, the Collective IS the actor for the graph's
purposes — accountability lives in the social contract (which
decides who can authorize what), not in per-edge attribution.
Whether and how the Collective then holds individual members
accountable internally is itself a matter for its social
contract.

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

**Invariant:** content-acts default permissive, governance-acts
default deny. The asymmetry reflects reversibility — a stray
Post is reversible by a counter-post, but a stray Proposal vote
or Item transfer binds the Collective externally — and mirrors
the broader governance primitive's stance that routine
gestures can be permissive while binding ones require explicit
eligibility ([governance.md](../primitive/governance.md)).

### Routing

When a member attempts to act-as a Collective C with a gesture
that would produce edge E, two Collective-specific steps run
ahead of the generic governance machinery:

1. **Classify** E as a content-act or governance-act (using the
   defaults above unless overridden).
2. **Look up the act-as rule** in C's social contract. If an
   explicit rule exists for E (by class or by specific edge
   type), its eligibility, weight, and threshold parameterize
   the act-as governance instance; otherwise the class default
   applies (allow for content-acts, deny for governance-acts).

The rule's threshold then runs as a standard governance
instance per [governance.md](../primitive/governance.md):
threshold `1` produces C's outgoing edge immediately; threshold
`> 1` holds the gesture pending until enough authorized members
co-sign — the
[Co-signed acts pattern](../primitive/governance.md#co-signed-acts-threshold--1-in-either-shape).
Collective act-as routing is one of the three current consumers
of that pattern.

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
  rename history is preserved. UNIQUE per instance. Data;
  per-field status carried separately by `name_status`.

Per-field moderation-status properties cover each user-filled
profile field — **`name_status`** (companion to the data sibling
`name`), **`display_name`**, **`description`**, **`avatar`**,
**`website_url`** — plus the node-level `moderation_status`
cache. Universal mechanics in
[nodes.md](../primitive/nodes.md#universal-per-field-moderation-status);
Collective-specific cascade in §9.

- **`governance`** — a single layered map property holding the
  Collective's entire social contract, keyed by `action_key`
  string. Each entry is a `Rule` object carrying two triples —
  `exec` (eligibility, weights, threshold for the action) and
  `amend` (eligibility, weights, threshold for amending this
  entry). Layered per
  [layers.md §3](../primitive/layers.md#3-layers-on-nodes);
  amending an entry is a standard Proposal targeting
  `governance.<action_key>`, gated by that entry's own `amend`
  triple — governance of governance, scoped per rule. See §8.

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
  Layered. The vocabulary is **implicit** — it is the set of
  strings used anywhere in the Collective's `governance` map
  (§8) eligibility predicates plus the strings assigned to any
  active member's `role`. Typos are amendable like any other
  `role` change via a Proposal targeting `CollectiveMember.role`.
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

Per node — Collective in §5.1, CollectiveMember in §5.2.
Dimension labels, sub-category labels, and traversal semantics
are not duplicated here; see [edges.md](../primitive/edges.md).

Every outgoing edge from a Collective is initiated through an
authorized member (§2).

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
  targets a property on the Collective — `name`, any per-field
  moderation-status property (§3), or any `governance.<action_key>`
  entry (§8). See
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
node type ([authorship.md](../primitive/authorship.md)). On the
graph that edge carries the `:AUTHOR` sub-label and originates at
the Collective node itself. The gesture is initiated off-graph by
an authorized CollectiveMember (§§2, 8), but the acting member is
not recorded — querying "who authored this?" returns the
Collective. See
[authorship.md "Collective-authored content"](../primitive/authorship.md#collective-authored-content);
the omission is the deliberate non-feature from §2.

A Collective is itself authored — its **author** is the User
identifiable as the earliest layer-1 timestamp among the
Collective's incoming CollectiveMember-claim edges (§1). The
author-User is a graph-derivable identity, not a stored
pointer; the role they hold on their CollectiveMember junction
is whatever the social contract named for the inaugural role
(commonly `founder`).

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

For the bootstrap case (founder's CollectiveMember at creation),
this collapses to its 1-of-1 form — see §1.

---

## 8. Governance — the social contract

A collective's **social contract** is its set of governance rules:
which decisions need votes, who can vote on each, with what
weights, and at what threshold. Different collectives have very
different rules — a corporation's CEO can fire workers
unilaterally; a household requires consensus for everything; a
co-op uses 2/3 majorities for major decisions. The graph supports
all of these without any primitive changes.

### The `governance` map

The entire social contract lives in **one layered map property
on the Collective** — `governance`, keyed by `action_key`
string. Each entry is a `Rule` object:

```
governance: Map<String, Rule>
  where Rule = {
    exec:  { eligibility, weights, threshold, exclude_subject? },
    amend: { eligibility, weights, threshold }
  }
```

`exec` is the per-component governance instance per
[governance.md §2](../primitive/governance.md#2-the-five-components)
that governs executing the action. `amend` is almost the same
shape but **without `exclude_subject`** — the subject of an
amendment is the rule entry itself, not a CollectiveMember, so
there is no member to exclude. Amending a rule entry is a
standard Proposal with `value_kind = 'rule'`,
`target_property = 'governance.<action_key>'`, and
`proposed_value` set to the new `Rule` object; the Proposal is
gated by the entry's own `amend` triple — governance of
governance, scoped per rule.

**The `amend` triple is self-applying.** Amending the `amend`
half of a rule uses that same `amend` triple. Tightening the
amendment process requires using the current amendment process —
no separate meta-meta-rule, no infinite regress, no primitive
default.

**Schema is fixed; the action set is data.** `governance` is a
single map-typed property declared once in
[graph-data-model.md](../implementation/graph-data-model.md);
new action keys never require a schema change. Adding,
amending, or tombstoning an entry all flow through the same
`target_property = 'governance.<action_key>'` Proposal mechanism.

### Action keys

`action_key` strings follow conventions the dispatch layer
**constructs from a member's gesture** — they are not
arbitrary strings the Collective invents. The Collective
writes rules under keys matching those conventions; the
dispatch builds the candidate key the same way at gesture
time and walks the fallback chain (below) until it finds an
entry. Three reserved top-level namespaces:

- **`decision:<operation>[:<role>]`** — Proposals that change
  Collective-internal state. The `<operation>` enumerates
  what the cascade can produce on `:Collective` or
  `:CollectiveMember`: `add_member`, `remove_member`,
  `change_role`, `change_ownership_pct`,
  `change_voting_weight`, `set:<property>` (for Collective
  node properties like `name`, `description`, `avatar`,
  `website_url`). The optional `<role>` parameter refines
  member-related operations by the affected member's role.
  Composite Collective operations (`admit_shareholder`,
  `transfer_shares`, …) take their own operation key paired
  with a handler that knows the composite shape per
  [proposal.md §2 "Composite proposals"](proposal.md#composite-proposals).
- **`actas:<gesture-identifier>`** — gating Collective
  outgoing edges through an authorized member. The
  gesture-identifier is derived from the actor edge being
  produced — by convention `<gesture>:<target_type>`
  (e.g. `actas:author:Post`, `actas:vote:Proposal`,
  `actas:transfer:Item`) so the dispatch builds the key
  deterministically from the would-be edge. **Two fixed
  class fallback keys** are recognized:
  `actas:content_default` and `actas:governance_default` —
  so a Collective can override the §2 in-prose defaults at
  class granularity without enumerating every gesture.
- **`system:<key>`** — Collective-level meta keys when needed
  (rare; the per-rule `amend` triple covers most cases).

Within each namespace the Collective declares only the keys
it wants to govern explicitly — the fallback chain (below)
handles the rest.

### Fallback chain at dispatch

The dispatch layer constructs the most-specific applicable
`action_key` from the gesture per the construction conventions
above, then walks **most-specific to most-general** through
`governance` until it finds a matching entry:

1. Most-specific (with parameter): e.g. `actas:author:Post`
   for "author a Post as the Collective", or
   `decision:add_member:worker` for "admit a worker".
2. Class-general (parameter dropped): e.g.
   `actas:content_default` for any content-act, or
   `decision:add_member` for any member admission regardless
   of role.
3. **In-prose default from §2:** allow any active member for
   content-acts; deny for governance-acts and for
   `decision:*` gestures without a matching key.

A Collective only needs to declare rules where it wants to
override the §2 in-prose defaults — the rest fall through.

A single-signer entry (`exec.threshold = 1`) collapses the
co-signed-acts pattern (see
[governance.md §3](../primitive/governance.md#co-signed-acts-threshold--1-in-either-shape))
to its 1-of-1 case: the gesture executes immediately when the
acting member satisfies `exec.eligibility`. A multi-signer entry
(`exec.threshold > 1`) holds the gesture pending in a Proposal
until co-signers cross the threshold.

### Snapshot at author-time

Collective Proposals apply the snapshot pattern from
[governance.md §5 "Rule snapshot at author time"](../primitive/governance.md#rule-snapshot-at-author-time)
via the
[`rule_anchor`](proposal.md#2-graph-side-properties) field —
required on every Proposal. A Proposal authored under a
`governance[X]` entry sets:

```
rule_anchor = { node_id: <Collective.id>, as_of: <author-time T> }
```

Tally and cascade read `Collective.governance` as-of `T` and
index by `action_key` to recover the frozen Rule. Amendments
committed mid-flight do not retroactively change in-flight
Proposals' threshold, eligibility predicate, or weights. Voter
applicability against the frozen predicate is still evaluated
live at tally time — a voter who acquires the right role
mid-flight counts; one who loses it drops via the
eligibility-dropout cascade.

### Simple and composite actions

The `Rule` shape is uniform across all action keys. What
differs is the Proposal's `value_kind` and `proposed_value`
shape:

- **Simple** — single property change.
  `value_kind ∈ {'scalar:string', 'scalar:float',
  'scalar:integer', 'rule'}`, `target_property` names the
  property being changed. Examples: rename the Collective
  (`scalar:string`, `target_property = 'name'`); tighten a
  rule (`rule`, `target_property = 'governance.<action_key>'`);
  fire a worker (the cascade writes the
  `Collective → CollectiveMember` `:APPROVAL` state-transition
  layer per §9).
- **Composite** — multi-property atomic change across multiple
  nodes. `value_kind = 'composite:<action_key>'`,
  `proposed_value` is a handler-specific bundle of `_from` /
  `_to` entries, `:TARGETS` points at the Collective node (the
  owning entity). See
  [proposal.md §2 "Composite proposals"](proposal.md#composite-proposals).
  Examples: `composite:decision:admit_shareholder` (new member
  + redistribute existing percentages so the total stays at
  100%); `composite:decision:transfer_shares` (move N% from
  one shareholder to another).

The author-time invariant (e.g. "post-change percentages sum
to 100%") and the cascade re-validation against current state
are handler responsibilities per `action_key`, not primitive
machinery.

### No primitive defaults

Unlike Chats — which default to community-vote moderation
because that fits informal communities — Collectives must
explicitly define their rules at creation. Creating a
Collective is the act of writing its social contract. The
example configurations below are starting templates, not
enforced defaults.

### Hierarchical authority is just a parameter choice

The "no admin veto" stance from chat governance is a
chat-specific default, not a primitive principle. A collective
whose social contract gives the CEO `weight = ∞` (or just
`exec.threshold = 1` with `exec.eligibility = role = CEO`) for
the "fire worker" decision IS expressing CEO-unilateral
authority — and the graph supports it. The primitive doesn't
pick a power structure; the collective does.

### Example configurations

The roles below (`CEO`, `founder`, `board_member`,
`shareholder`, `worker`, etc.) are **collective-specific** per
§3 — each social contract defines its own role vocabulary; the
primitive only requires it to be used consistently for that
collective's eligibility and weight rules. Each table shows
`exec` only; the `amend` triple for each entry is the
Collective's choice (typically tighter than its `exec` — see
the corporate example below).

#### Corporate hierarchy

A small company with founders, a CEO, board members, and
workers.

| `action_key`                                | `exec.eligibility`                                        | `exec.threshold` |
|---------------------------------------------|-----------------------------------------------------------|------------------|
| `decision:add_member:worker`                | `role = CEO`                                              | 1 vote           |
| `decision:remove_member:worker`             | `role = CEO`                                              | 1 vote           |
| `decision:change_role:worker`               | `role = CEO`                                              | 1 vote           |
| `decision:add_member:board_member`          | `role = founder`, weighted by `ownership_pct`             | > 50%            |
| `decision:remove_member:board_member`       | `role IN (founder, board_member)`, `exclude_subject`      | ≥ 2/3            |
| `decision:remove_member:CEO`                | `role = board_member`                                     | ≥ 2/3            |
| `decision:admit_shareholder` *(composite)*  | `role IN (founder, shareholder)`, weighted by stake       | ≥ 75%            |
| `decision:transfer_shares` *(composite)*    | `role = shareholder`, weighted by `ownership_pct`         | ≥ 75%            |
| `decision:set:name`                         | All active members                                        | > 50%            |
| `actas:author:Post`                         | `role = press_officer` *(overrides any-member default)*   | 1 signer         |
| `actas:author:Proposal`                     | `role = CEO`                                              | 1 signer         |
| `actas:vote:Proposal`                       | `role IN (CEO, board_member)`                             | 1 signer         |
| `actas:transfer:Item`                       | `role IN (founder, board_member)`, weighted by stake      | ≥ 50% signers    |

A worker is hired or fired by a single CEO vote; a board
member is removed only by board supermajority; a CEO is removed
only by the rest of the board. Routine PR posting is delegated
to a single press officer (locking down the otherwise
any-member default for content-acts), while consequential
moves — proposing, voting, and transferring company assets —
are routed to leadership and the board. Shareholder admissions
and transfers are composite actions: the Proposal's bundle
covers both the new/changed CollectiveMember junction and the
redistributed `ownership_pct` values across affected members,
atomic at cascade.

Sample `amend` triples for the same Collective, showing how
amendment cost is calibrated per rule:

| `action_key`                              | `amend.eligibility`                                       | `amend.threshold` |
|-------------------------------------------|-----------------------------------------------------------|-------------------|
| `decision:add_member:worker`              | `role IN (founder, board_member)`                         | > 50%             |
| `decision:transfer_shares`                | `role = shareholder`, weighted by `ownership_pct`         | ≥ 90%             |
| `decision:set:name`                       | `role IN (founder, board_member)`                         | ≥ 2/3             |

The CEO-can-hire rule is cheap to amend (board majority);
share-transfer rules are expensive to amend (near-unanimous
shareholders); the Collective's name is moderately gated. Each
rule self-describes its mutability cost.

#### Household (5 people)

| `action_key`                              | `exec.eligibility`                  | `exec.threshold`                          |
|-------------------------------------------|-------------------------------------|-------------------------------------------|
| `decision:add_member`                     | All active members                  | 100% of cast, 100% quorum                 |
| `decision:remove_member`                  | All members, `exclude_subject`      | ≥ 90% of cast, 100% quorum of remaining   |
| `decision:routine_spending`               | All active members                  | > 50%, ≥ 60% quorum                       |
| `actas:vote:Proposal`                     | All active members                  | > 50% signers                             |
| `actas:transfer:Item`                     | All active members                  | > 50% signers                             |

Everyone has equal voice; consensus dominates. Content-acts
(posting to the household feed, reacting on shared content)
are left at the any-member default — no override.

#### Worker co-op

All members equal stake; some routine decisions delegated to
officers.

| `action_key`                              | `exec.eligibility`              | `exec.threshold` |
|-------------------------------------------|---------------------------------|------------------|
| `decision:add_member`                     | All active members              | ≥ 2/3            |
| `decision:remove_member`                  | All members, `exclude_subject`  | ≥ 2/3            |
| `decision:routine_operations`             | `role = officer`                | > 50%            |
| `decision:major_policy_change`            | All active members              | ≥ 2/3            |
| `decision:change_capital_structure`       | All active members              | ≥ 75%            |
| `actas:vote:Proposal`                     | All active members              | > 50% signers    |
| `actas:transfer:Item`                     | All active members              | ≥ 2/3 signers    |

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

Moderation is the only redaction trigger on a Collective node
itself ([moderation.md §1](moderation.md#1-the-two-classification-paths)) —
typically targeting identity fields. Entries inside `governance`
are in scope in principle but rarely the target.

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
