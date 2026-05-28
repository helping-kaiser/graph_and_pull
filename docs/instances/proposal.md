# Proposal

The **Proposal** is a content node — the **subject carrier
for property-level governance votes**. Wherever the platform
needs to vote on changing a graph property (a Network
parameter, a User's `network_role`, a Chat's `name`, a
ChatMember's `role`, a content node's per-field
moderation-status property like `'bio'` or `'content'`), the
vote is cast on a Proposal that *targets* that node's
specific property, not on the underlying node directly. When
the tally crosses threshold, a cascade writes a new layer on
the target property with the Proposal's `proposed_value`.
Multi-property atomic changes — admitting a shareholder with
re-distribution, for instance — use **composite proposals**
(§2) which carry a structured bundle in place of a scalar
value and cascade atomically across all affected properties.

This doc describes the node; the **governance mechanics** it
hosts — eligibility, weight function, threshold policy, outcome
semantics, multi-candidate decisions — live in
[governance.md](../primitive/governance.md).

---

## 1. Creation

Any actor eligible for the governance instance the Proposal
serves can author one (see
[governance.md §2.2](../primitive/governance.md#22-eligibility)).
There is no second-party approval flow: like a Post (see
[post.md §1](post.md#1-creation)), the author's outgoing
vote edge is the only edge needed to bring the node into
the graph.

What the author specifies at creation:

- **The target node** — recorded as the system-created
  outgoing `:TARGETS` structural edge (§4). Fixed at
  creation; a Proposal cannot be re-targeted.
- **`target_property`**, **`proposed_value`**, **`value_kind`**,
  and **`rule_anchor`** — graph properties on the new Proposal
  (§2).

The system writes three records atomically: the
`:Proposal` node, the outgoing `:TARGETS` edge, and an
incoming vote edge from the authoring actor (§5).

---

## 2. Graph-side properties

- **`target_property`** — the name of the graph property
  on the target node being proposed for change (e.g. a
  per-field moderation-status property like `'bio'` or
  `'content'`, `'name'`, `'role'`, `'network_role'`,
  `'guidelines_version'`), or the reserved sentinel `'node'`
  for whole-node operations. The sentinel
  is defined in
  [nodes.md "Whole-node targeting"](../primitive/nodes.md#whole-node-targeting-the-node-sentinel)
  and has two consumers:
  - **Illegal-content classification** — every user-input
    field plus every attached media on the node (see
    [moderation.md §1](moderation.md#1-the-two-classification-paths)).
    `proposed_value = 'illegal'`.
  - **Chat-internal disavowal** — Level 1 against a
    `ChatMessage` or Level 2 against a `ChatMember` (see
    [chats.md §10](chats.md#10-moderation)).
    `proposed_value ∈ {'disavowed', 'normal'}`.
- **`proposed_value`** — the value to set on
  `target_property` if the Proposal passes. Shape depends on
  `value_kind` (below); values used with the `'node'` sentinel
  are listed in the two bullets above.
- **`value_kind`** — string discriminator on the shape of
  `proposed_value`, set at Proposal creation and consumed by
  frontends to render the right editor / display widget
  without needing out-of-band knowledge of every
  `target_property`. Enumerated:
  - `'scalar:string'` — `proposed_value` is a string
    (moderation classifications, role strings, `name`
    changes, …).
  - `'scalar:float'` — `proposed_value` is a Float
    (quorum fractions, `ownership_pct`, …).
  - `'scalar:integer'` — `proposed_value` is an Integer
    (absolute quorum counts, half-lives, …).
  - `'rule'` — `proposed_value` is a `Rule` object — the
    (eligibility, weights, threshold) triple used for
    governance rules. Consumer: governance-rule amendments
    on collectives (see
    [collectives.md §8](collectives.md#8-governance--the-social-contract)).
  - `'composite:<action_key>'` — `proposed_value` is a
    handler-specific structured bundle covering multiple
    properties across multiple nodes, applied atomically by
    the cascade. See "Composite proposals" below.
- **`rule_anchor`** — **required.** Every Proposal is grounded
  in a rule that lives in one or more layered properties on
  some node; this field identifies that node, per
  [governance.md §5 "Rule snapshot at author time"](../primitive/governance.md#rule-snapshot-at-author-time).
  The dispatcher reads each rule property on `rule_anchor`
  **as-of the Proposal's authorship-edge timestamp** per
  [authorship.md](../primitive/authorship.md) (the earliest
  incoming actor edge) at tally and cascade, so amendments
  committed mid-flight don't retroactively change in-flight
  Proposals' rule parameters.

  ```
  rule_anchor: String   // node ID hosting the rule property(ies)
  ```

  Covers every current consumer with a single value:
  - Collective Proposals (executions or amendments under
    `governance.<action_key>`) — `rule_anchor = <Collective.id>`;
    dispatcher reads `Collective.governance` as-of authorship
    and indexes by action_key.
  - Network dual-quorum moderation Proposals —
    `rule_anchor = <Network.id>`; dispatcher reads both
    `_quorum_fraction` and `_quorum_count` as-of authorship so
    the `min(P × |active|, K)` rule is fully frozen.

  Timestamp-based addressing on node-property layers is a
  forward dependency — see
  [layers.md §3](../primitive/layers.md#3-layers-on-nodes).

None of these properties layers — the Proposal's identity *is*
the specific change it proposes; mutating any of them
mid-lifecycle would change what voters are voting on. A revised
target, value, kind, or anchor requires a new Proposal.

### Composite proposals

A composite Proposal carries a structured `proposed_value`
bundle that atomically writes layers on several properties —
usually across several nodes that together encode one
invariant. The canonical case is shareholder admission:
creating the new `:CollectiveMember` junction with N% stake
and reducing existing shareholders' `ownership_pct` so the
100% total holds; either change passing alone would break the
invariant.

Three conventions hold across every composite kind:

1. **`:TARGETS` points at the owning entity.** For
   Collective-internal composites, the Collective node — not
   any one affected junction; the bundle inside
   `proposed_value` carries the per-node specifics.
2. **Bundle entries carry `_from` and `_to` for every
   property being changed.** At threshold-cross the cascade
   re-validates by checking each affected property's current
   value equals the entry's `_from`. Any mismatch — typically
   state drift between author-time and tally-time — causes
   the cascade to refuse; the Proposal records a terminal
   `passed_but_invariant_rejected` outcome and a fresh
   Proposal with refreshed numbers is needed. Straightforward
   compare-and-swap; voters see exactly what's being asserted
   about current state.
3. **Per-`action_key` handlers own bundle shape, author-time
   invariant validation, and the cascade transaction.** The
   primitive doesn't enumerate composite shapes — each
   application doc declares its own action keys.

Composite kinds in current use live in their application docs
— see
[collectives.md §8](collectives.md#8-governance--the-social-contract)
for `composite:decision:admit_shareholder` and
`composite:decision:transfer_shares`.

A Proposal does **not** carry any per-field moderation-status
properties: it has no user-input fields to redact (see
[nodes.md "Universal: per-field moderation status"](../primitive/nodes.md#universal-per-field-moderation-status),
which excludes Proposal alongside the junction nodes for the
same reason).

Concrete property types and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 3. Postgres-side content

None. The Proposal's full substance is `target_property` +
`proposed_value` + the `:TARGETS` edge — anything
human-readable a viewing user might want about the Proposal is
derivable from those plus the target node's current state.

The platform-guidelines amendment Proposal (see
[platform-guidelines.md §3](platform-guidelines.md#3-amendment-procedure))
is the one application where understanding the change
requires off-graph text (the new guidelines version,
published in the repo); even there, only the version number
and SHA-256 hash ride on the Proposal.

---

## 4. Edges

### As source (outgoing)

A Proposal carries exactly one outgoing structural edge,
system-created at creation and never re-targeted:

- **`Proposal → Target Node` (`:TARGETS`)** — identifies
  the node whose property is being changed. Targets span
  every node category: actor (User, Collective), content
  (Post, Comment, Chat, ChatMessage, Item, Hashtag), junction
  (`ChatMember.role`, `CollectiveMember.role`), and system
  (the `:Network` singleton — see
  [network.md §11](../primitive/network.md#11-amending-network-parameters)).
  The property name and proposed value live on the Proposal
  node (§2), not on the edge — the change is intrinsic to
  the Proposal, not to the relationship. See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting).

### As target (incoming)

A Proposal receives vote edges and (optionally) reference
edges. It does **not** receive `:CONTAINMENT` edges —
Comments attach only to Post, Comment, Chat, ChatMessage,
and Item, per
[edges.md §2 "Containment / belonging"](../primitive/edges.md#containment--belonging).

**Vote edges**, two shapes per
[governance.md §3](../primitive/governance.md#3-the-two-vote-shapes);
choice is per-application:

- **Shape A — actor edges** from Users and Collectives,
  `(sentiment, importance)` per
  [edges.md §1](../primitive/edges.md#1-actor-edges).
  `dim1` carries vote direction; `dim2` carries the
  voter's personal stake. Used for relationship-shaped
  subjects (junction approvals).
- **Shape B — structural vote edges** from the voter's
  eligibility junction. `dim1` carries vote direction,
  `dim2` is `0`. Per
  [edges.md §2 "Voting (Shape B)"](../primitive/edges.md#voting-shape-b):
  `ChatMember → Proposal` and
  `CollectiveMember → Proposal`.

For Network-scope governance (moderation, mod role changes,
`:Network` parameter amendments — see
[network.md §10](../primitive/network.md#10-network-wide-governance)),
the vote is Shape A: the `User → Proposal` actor edge from
[edges.md §1](../primitive/edges.md#1-actor-edges) carries
the vote. Network membership has no per-member junction, so
the User node is itself the eligibility carrier. The actor
edge keeps its normal meaning: `dim1` is the voter's
sentiment toward the change (positive = support, negative =
oppose), `dim2` is importance / personal stake. Network-scope
tally is petition-style: only `dim1 > 0` edges contribute
(`+1 × voter_weight` each); `dim1 ≤ 0` edges are valid
graph objects but contribute `0` to the tally. The pass
condition is dual-quorum:
`positive_count ≥ min(P × |active members|, K)` plus the
mod-gate. See
[governance.md §3 "Petition-style tally and dual quorum"](../primitive/governance.md#petition-style-tally-and-dual-quorum-network-scope-only).

**Reference edges:**

- **`ChatMessage / Post / Comment → Proposal` (`:REFERENCES`)**
  when a content node embeds the Proposal — a chat message
  surfacing it for chat members to vote on, a Post campaigning
  for support, a Comment citing it in debate. See
  [edges.md §2 "Reference"](../primitive/edges.md#reference).

A Proposal **never** receives a `:TARGETS` edge from
another Proposal: moderation can't target it (§2), and no
other governance application proposes changes to a
Proposal's own properties (neither layers, §2).

---

## 5. Authorship

The authoring gesture **is** the author's first vote on
the Proposal — a Proposal exists to be voted on, and there
is no separate personal-stance dimension to preserve apart
from the vote. The same edge serves both roles, so the
earliest-incoming-edge author derivation
([authorship.md](../primitive/authorship.md)) and the
first-voter identity coincide. See
[moderation.md §2](moderation.md#2-reports--proposals-on-the-graph)
for the worked example with reports.

---

## 6. Lifecycle

The governance mechanics that drive each transition stay in
[governance.md](../primitive/governance.md); what follows
is the node-level progression.

- **Open** — default state from creation. New eligible
  actors may cast vote edges at any time; existing voters
  change their position by appending a new layer to their
  existing vote edge
  ([governance.md §4](../primitive/governance.md#4-append-only-throughout)).
  **No time-boxing**: votes stand until changed and the
  Proposal stays open indefinitely
  ([governance.md §6 "No time-boxing"](../primitive/governance.md#no-time-boxing)).
- **Tally** — triggered only by a new or updated vote
  layer on the Proposal, not on a schedule or by background
  eligibility shifts
  ([governance.md §6](../primitive/governance.md#6-when-outcomes-take-effect)).
- **Cascade** — when a new-vote tally crosses threshold,
  the system writes a new layer on `target_property` of the
  target with the Proposal's `proposed_value`
  ([graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows)).
  Outcome semantics, cascade bounds, and the
  `'illegal'`-specific cascade behavior (per-field redaction
  marker, data-sibling write where applicable, Postgres
  tombstoning, archive disposition) live in
  [governance.md §2.5](../primitive/governance.md#25-outcome),
  [moderation.md §1](moderation.md#1-the-two-classification-paths),
  and
  [layers.md §5](../primitive/layers.md#5-deletion-policy).
  The `'node'` sentinel (§2) dispatches on the target's node
  type — re-layering `Chat → ChatMember` for a `ChatMember`
  target, or writing nothing on a `ChatMessage` target since
  the Proposal's pass-state is itself the chat's stance.
  Composite Proposals (§2 "Composite proposals") re-validate
  against current state at this point — if any bundle entry's
  `_from` no longer matches the affected property's current
  value, the cascade refuses and the Proposal terminates as
  `passed_but_invariant_rejected`. A fresh Proposal with
  refreshed numbers is the only path forward.
- **Outcome stickiness** — after the cascade, the target
  stays in its new state until a future vote event pushes
  it back across a threshold
  ([governance.md §6 "Why outcomes are sticky"](../primitive/governance.md#why-outcomes-are-sticky-not-continuously-rendered)).
  Reverting requires a counter-Proposal; multiple Proposals
  can coexist against the same property, each passing or
  failing on its own votes
  ([governance.md §2.1](../primitive/governance.md#21-subject),
  [§10](../primitive/governance.md#10-multi-candidate-decisions)).
- **No deletion** — per
  [layers.md §5](../primitive/layers.md#5-deletion-policy),
  graph structure is never removed; the Proposal node, its
  `:TARGETS` edge, and every incoming vote and reference
  edge stay on the graph as a permanent record. There is
  no redaction path for the Proposal itself (§2).

---

## What this doc is not

- **Not the governance primitive.** Eligibility, weight
  functions, threshold policies, outcome semantics, the
  two vote shapes, sticky outcomes, multi-candidate
  decisions — [governance.md](../primitive/governance.md)
  is canonical.
- **Not an enumeration of applications.** Application-side
  parameters (which property, which eligibility set, which
  threshold) live in each application doc:
  [moderation.md](moderation.md),
  [platform-guidelines.md](platform-guidelines.md),
  [network.md §§9, 11](../primitive/network.md#9-mod-role-changes-via-multi-sig-proposal),
  [chats.md §10](chats.md#10-moderation),
  [collectives.md](collectives.md).
- **Not the cascade mechanism.** The cascade and the
  redaction-cascade specifics live in
  [graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows),
  [layers.md §5](../primitive/layers.md#5-deletion-policy),
  and [moderation.md](moderation.md).
- **Not the edge catalog.** Per-source vote-edge types and
  the per-target `:TARGETS` enumeration live in
  [edges.md](../primitive/edges.md).
- **Not the Memgraph or Postgres schema.** Concrete
  property types and indexes live in
  [graph-data-model.md](../implementation/graph-data-model.md);
  Postgres has no Proposal shape.
