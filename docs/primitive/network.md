# Network

The **Network** is the global community of every User on a CoGra
instance — the body that backs platform-wide governance: content
moderation, dispute resolution, and anything that affects the
whole instance rather than a specific chat or collective.

This doc is the per-node catalog for the `:Network` singleton: how
it is created, what it carries on the graph, what edges it
participates in, and how its lifecycle plays out as a graph
object. The governance applications the singleton hosts —
membership-and-roles structure, moderator role changes,
Network-wide governance, and parameter amendments — live in the
topical appendices after the per-node skeleton (§§8-11). The
governance primitive itself stays in
[governance.md](governance.md).

---

## 1. Distinct from Collective

A [Collective](../instances/collectives.md) is a small group with
a defined membership: a household, band, co-op, company.
Membership is explicit and approval-gated.

The Network is the opposite — the set of every User on the graph.
Membership is **automatic on registration** (see
[invitations.md](invitations.md)); there is no approval gate;
there is no "this band vs that band." It is one Network per
instance.

Federation across instances is a forward question — see
[open-questions.md Q15](../open-questions.md). Each instance has
its own Network until then.

---

## 2. Creation

The `:Network` singleton is brought into existence by the
**instance bootstrap migration** — a one-shot setup step that
runs once when an instance is created, alongside the database
schema migrations. The migration is the only path that writes
the singleton. Every subsequent change to the singleton's
parameters or to any user's role runs through governance.

The migration writes three nodes in a single atomic transaction:

1. The `:Network` singleton with the default property values
   listed in
   [graph-data-model.md](../implementation/graph-data-model.md).
2. The genesis User node, with `network_role = 'moderator'` —
   the bootstrap moderator (see §9). Identity (username,
   credentials) is supplied to the migration at run time: the
   central instance run by the project picks the project owner;
   a federated fork sets its own genesis.
3. The `bot-defense` Hashtag node, so its content-addressed
   UUIDv5 (per
   [data-model.md "Node identity strategies"](../implementation/data-model.md#node-identity-strategies))
   is present from network birth and every frontend can resolve
   it. See
   [feed-ranking.md](feed-ranking.md).

All three writes share one transaction; an observer never sees
the singleton without its moderator or the `bot-defense`
Hashtag. The migration is not a runtime flow — there is no
"first user to register" detection, no genesis-flag column, no
special branch in the registration endpoint. Subsequent Users
register exclusively through invitation per
[invitations.md](invitations.md).

This migration is the only step in the system that depends on
out-of-graph authority (see the global invariant in
[graph-model.md §1](graph-model.md#1-core-principles)); the
authority is confined to the migration.

Bitcoin analogy: someone has to mine the genesis block. From
there it is community-driven.

---

## 3. Graph-side properties

The `:Network` node carries the instance's configuration
parameters. All properties are **layered** per
[layers.md](layers.md), so every parameter change has a preserved
history. Each is amendable via a standard Proposal targeting the
property name (same primitive as everything else — see
[governance.md §2.1](governance.md#21-subject)), gated by one of
the amendment-rule pairs called out below.

Concrete property types and defaults live in
[graph-data-model.md](../implementation/graph-data-model.md);
this section names each parameter, its role, and which
amendment-rule pair gates changes to it.

### Eligibility-definition

- **`active_threshold_days`** — recency window that makes a User
  count as an "active member" for governance tallies (a User
  with at least one outgoing actor edge within the last N days).
  Composes with tally-time eligibility per §10. Gating bucket:
  baseline (§11).

Network-scope governance uses petition-style tally under a
dual-quorum gate (see
[governance.md §3](governance.md#petition-style-tally-and-dual-quorum-network-scope-only)).
Each instance below carries a **pair** of parameters: a
fractional bar (`P`, `*_quorum_fraction`) and an absolute bar
(`K`, `*_quorum_count`). The operative bar at tally time is
`min(P × |active|, K)`.

### Mod-role-change governance

- **`mod_role_change_quorum_fraction`**,
  **`mod_role_change_quorum_count`** — dual-quorum pair for
  the multi-sig Proposal that adds or removes a moderator
  (§9). Gating bucket: critical (§11).

### Content-moderation governance

- **`moderation_sensitive_quorum_fraction`**,
  **`moderation_sensitive_quorum_count`** — dual-quorum pair
  for `'sensitive'` classification Proposals. Gating bucket:
  baseline.
- **`moderation_illegal_quorum_fraction`**,
  **`moderation_illegal_quorum_count`** — dual-quorum pair
  for `'illegal'` classification Proposals. Gating bucket:
  critical.

### Platform-guidelines governance

- **`guidelines_version`**, **`guidelines_hash`** — the pinned
  version and SHA-256 of the current platform guidelines (see
  [platform-guidelines.md](../instances/platform-guidelines.md)).
  Amended together by the guidelines-amendment instance below,
  not by either property-change bucket.
- **`guidelines_change_quorum_fraction`**,
  **`guidelines_change_quorum_count`** — dual-quorum pair for
  the guidelines-amendment instance itself. Gating bucket:
  critical.

### Feed-ranking calibration

- **`time_decay_half_life_days`** — half-life of the reactor-edge
  time-decay factor `f(Δt)` used by the feed-ranking algorithm
  (see [feed-ranking.md §7.3](feed-ranking.md#73-shape--exponential-30-day-half-life-frontend-tunable)).
  The default seeded at genesis is 30 days; the property is
  amendable so the network can recalibrate freshness sensitivity
  as the graph matures. Frontend overrides remain available per
  §7.3; this property sets the network default. Gating bucket:
  baseline.

### Amendment-rule pairs (governance of governance)

The pairs that govern changes to the singleton's own parameters,
split by stakes (§11). Each amendment-rule pair is itself a
dual-quorum pair:

- **Baseline:** **`property_change_quorum_fraction`**,
  **`property_change_quorum_count`** — for low-stakes parameters
  (`moderation_sensitive_*`, `active_threshold_days`,
  `time_decay_half_life_days`, and the baseline pair itself).
- **Critical:** **`critical_property_change_quorum_fraction`**,
  **`critical_property_change_quorum_count`** — for parameters
  whose abuse has destructive or platform-wide reach
  (`mod_role_change_*`, `moderation_illegal_*`,
  `guidelines_change_*`, and the critical pair itself).

Each pair is self-amending: a baseline-pair amendment passes
under baseline rules; a critical-pair amendment passes under
critical rules. Defaults exist to bootstrap; they are not fixed
rules.

The singleton carries **no `moderation_status` property**. Like
junction nodes and the Proposal node, it has no user-input fields
to redact (see
[nodes.md "Universal: moderation_status"](nodes.md#universal-moderation_status));
the lifecycle consequence is §7.

The singleton exists so platform-wide governance has a graph
node to target. Without it, statements like "Network parameters
are amendable via Proposal" would be hand-waving — Proposals
need a node.

---

## 4. Postgres-side content

None. The `:Network` singleton is pure graph state. Every
parameter lives as a layered graph property on the node itself;
there is no `network` row, no display-content table, no
per-singleton data keyed by its UUID. The platform-guidelines
document that the singleton pins via `guidelines_version` and
`guidelines_hash` lives in the project repo, not in Postgres —
see
[platform-guidelines.md](../instances/platform-guidelines.md).

---

## 5. Edges

The `:Network` is a singleton parameter container. It is not an
actor; it has no Postgres-side display content; and it
participates in only a narrow set of edges.

### As source (outgoing)

The `:Network` authors no edges of any type. It is purely a
target that other parts of the graph point at.

### As target (incoming)

The `:Network` node receives:

- **`Proposal → Network` (`:TARGETS`)** when a Proposal
  targets one of the singleton's parameters
  (`mod_role_change_*`, `moderation_sensitive_*`,
  `moderation_illegal_*`, `guidelines_*`,
  `active_threshold_days`, or either amendment-rule pair). The
  amendment-rule pair that gates each property is named in §3.
  See
  [edges.md §2 "Subject targeting"](edges.md#subject-targeting).
- **`ChatMessage / Post / Comment → Network` (`:REFERENCES`)**
  when a content node mentions or embeds the singleton (e.g. a
  Post discussing platform governance). See
  [edges.md §2 "Reference"](edges.md#reference).

Network-scope governance instances do **not** create new
structural edges to the `:Network` node. Votes on Network-scope
Proposals — moderator role changes (§9), content moderation
classifications, and singleton parameter amendments (§11) — use
the existing `User → Proposal` **actor edge** as the Shape A
vote (see
[edges.md §1](edges.md#1-actor-edges) and
[governance.md §3](governance.md#3-the-two-vote-shapes)). The
Proposal itself targets the relevant subject (a User for role
changes, the `:Network` singleton for parameter amendments);
the votes themselves never carry an edge to or from the Network
node.

---

## 6. Authorship

There is no authorship section for the `:Network` singleton — by
§2 the Network is system-created at bootstrap and has no author
in the [authorship.md](authorship.md) sense. The
earliest-incoming-edge rule does not meaningfully apply: the
first edge the singleton receives is typically a `:TARGETS`
edge from a property-amendment Proposal or a `:REFERENCES`
edge from a content node mentioning the network, but those
edges' authors are authors of the *proposing* or *referencing*
node, not of the singleton. The Network is a system concept,
not authored content. (Same shape — though for a different
reason — as the Hashtag exemption in
[hashtag.md §5](../instances/hashtag.md#5-lifecycle).)

---

## 7. Lifecycle

The `:Network` singleton is **never deleted**. The append-only
rule applies as it does to every other node.

There is no redaction path either. The singleton carries no
user-input fields, so neither
[layers.md §5](layers.md#5-deletion-policy)'s in-place redaction
nor [retention-archive.md](retention-archive.md)'s archive
disposition has anything to act on. The node's UUID is stable
across the entire lifetime of the instance.

The singleton's **state changes** are exclusively parameter
amendments — passing Proposals targeting one of the layered
properties from §3, gated by the amendment-rule pair that §3
assigns. The full mechanism, threshold defaults, mod gate, and
the rationale for two stakes-tiered buckets live in §11. There
are no other lifecycle events the singleton has: no membership
changes (its eligibility set lives on User nodes, §8), no
transfer, no merge, no archive.

Federation across instances — whether two instances' singletons
can be reconciled — is the forward question already flagged in
§1, deferred to
[open-questions.md Q15](../open-questions.md).

---

## 8. Membership and roles

Every User has a `network_role` graph property:

- **`member`** — every registered user, automatically. The
  default.
- **`moderator`** — a small set of users who gate platform-wide
  governance actions (see
  [moderation.md](../instances/moderation.md) for the gating
  rule on content moderation; §9 below for the gating rule on
  mod role changes themselves).

`network_role` is a graph-side property on the User node,
layered per [layers.md](layers.md) — promotion and demotion
preserve full history. It is **not** a property on the Network
singleton, and the singleton has no incoming structural edge
representing membership: Network membership has no separate
gesture, so there is no `ChatMember`-/`CollectiveMember`-style
junction to bind a User to the Network. The eligibility-set is
"every User node on the graph," filtered by the recency window
from §3 (`active_threshold_days`).

Whether Collectives can be Network members or moderators is
deferred. For now, only Users carry `network_role`.

---

## 9. Mod role changes via multi-sig Proposal

Adding or removing a moderator uses the standard Proposal
mechanism
([governance.md §2.1](governance.md#21-subject)):

- **Subject:** A Proposal targeting `User.network_role` of the
  user being promoted or demoted, with `proposed_value` set to
  the new role.
- **Eligibility:** all active Network members.
- **Threshold:** multi-sig — **≥1 existing moderator's positive
  vote** plus the dual-quorum bar from §3:
  `positive_count ≥ min(Network.mod_role_change_quorum_fraction
  × |active|, Network.mod_role_change_quorum_count)`. Tally is
  petition-style (positive votes only) per
  [governance.md §3](governance.md#petition-style-tally-and-dual-quorum-network-scope-only).

The two gates implement a **separation of powers** — see
[governance.md §2.4](governance.md#24-threshold-policy)
"Multi-gate decisions". The mod-gate side of the pair —
"≥1 mod positive vote; mod weight = member weight = 1" — is
defined as a primitive in
[governance.md §7](governance.md#7-the-mod-gate) and reused here
and in §11. Each gate of the multi-gate pair counters a distinct
failure mode (sitting-mod coup vs community coordinated removal);
both required = both failure modes blocked.

Removal mirrors promotion mechanically: same Proposal
mechanism with `proposed_value = 'member'`, same dual-gate
rule. Two structural constraints sit on top of the mechanism:

- **Moderator floor.** The active moderator count cannot drop
  below **1**. A removal Proposal that would push the count
  below the floor is rejected at the dispatch check, regardless
  of vote tally. Without at least one moderator the mod-gate
  (§7,
  [governance.md §7](governance.md#7-the-mod-gate))
  cannot be opened, and every Network-scope Proposal would
  silently stall.
- **Bootstrap mod undemotable.** The genesis User installed by
  the bootstrap migration (§2) carries an undemotable
  `'moderator'` status: no Proposal can move them off
  `network_role = 'moderator'`. The dispatch check rejects the
  outcome write even on a passed tally. The exception exists
  for bot-defense — if every other moderator is compromised or
  removed, the bootstrap mod remains as the immovable floor of
  the mod-gate, blocking a coordinated full-takeover. The
  asymmetry is deliberate; this is the only mechanism in the
  system that exempts a graph object from governance reach.

---

## 10. Network-wide governance

The Network is the eligibility-and-voting body for any
platform-scoped governance instance:

- Adding and removing moderators (§9 above).
- Content moderation classifications — see
  [moderation.md](../instances/moderation.md).
- Tuning the `:Network` singleton's parameters themselves
  (governance of governance) — see §11.

Each runs as a Network-scope governance instance per
[governance.md §3](governance.md#3-the-two-vote-shapes). Two
consequences shared across all three:

- **The eligibility carrier is the User node itself**, not a
  junction. Network membership has no separate gesture (§8), so
  there is no `ChatMember`-/`CollectiveMember`-style junction to
  carry the vote. This is the natural Shape A case: the vote
  IS the existing `User → Proposal` actor edge from
  [edges.md §1](edges.md#1-actor-edges) — no separate structural
  edge is created. The actor edge keeps its normal `(sentiment,
  importance)` meaning; the tally reads `sign(sentiment)` for
  the binary outcome. See
  [governance.md §3](governance.md#3-the-two-vote-shapes)
  "Shape A".
- **Mod weight = member weight = 1; mod is a gate, not a
  weight.** Moderators do not outvote the community; the
  "≥1 mod positive vote" rule is a procedural gate, never a
  weighting. The rule is defined as a primitive in
  [governance.md §7](governance.md#7-the-mod-gate); it applies
  uniformly to every Network-scope Proposal (mod role changes,
  classifications, parameter amendments).

The `active_threshold_days` recency window from §3 composes
naturally with tally-time eligibility per
[governance.md §2.2](governance.md#22-eligibility): at the
moment a new or updated vote layer triggers a tally, the
eligible set is "Users with at least one outgoing actor edge
whose timestamp falls within the last
`Network.active_threshold_days` days as of *that* tally." A
voter whose latest activity has dropped out of the window
drops from the tally; a voter who becomes active again counts
the next time their renewed activity puts them back inside.
The window slides; eligibility is evaluated at a single point
per tally, not snapshotted at vote time.

No carve-out is needed for first-time voters or long-inactive
moderators. The Shape A vote is itself an outgoing
`User → Proposal` actor edge per
[edges.md §1](edges.md#1-actor-edges); the act of casting it
places that user inside `active_threshold_days` at the tally
that the vote triggers. A first-time voter who has never
otherwise interacted with the graph becomes active *by voting*,
in time to be counted. A moderator whose previous edges have
all aged out re-enters the window the same way the moment they
cast or update their gate-opening vote. Eligibility tracks
participation directly: the only way to be excluded is to not
participate.

---

## 11. Amending `:Network` parameters

Two amendment-rule pairs gate changes to the singleton's own
properties, separated by stakes:

| Bucket   | Dual-quorum pair                                  | `P` default | `K` default | Mod gate | Governs |
|----------|---------------------------------------------------|-------------|-------------|----------|---------|
| Baseline | `property_change_quorum_fraction`, `property_change_quorum_count` | `0.25` | `5000` | required | `moderation_sensitive_*`, `active_threshold_days`, the baseline pair itself |
| Critical | `critical_property_change_quorum_fraction`, `critical_property_change_quorum_count` | `0.50` | `10000` | required | `mod_role_change_*`, `moderation_illegal_*`, `guidelines_change_*`, the critical pair itself |

Pass condition for either pair is the dual-quorum form from
[governance.md §3](governance.md#petition-style-tally-and-dual-quorum-network-scope-only):
`positive_count ≥ min(P × |active members|, K)`.

`guidelines_version` and `guidelines_hash` are not in either
bucket — they are amended together by the guidelines-amendment
instance (`guidelines_change_*`, see
[platform-guidelines.md](../instances/platform-guidelines.md)).

The critical bucket holds parameters whose abuse has destructive
or platform-wide reach: stripping moderators, triggering the
redaction cascade, or shifting the normative frame for *all
future* moderation. Those amendments earn a supermajority. Soft
flags and eligibility windows move under the lighter baseline
pair so routine tuning isn't paralyzed.

A single uniform pair would lose the stakes split; a per-property
pair for each amendable property would double the singleton's
property count without adding meaningful differentiation. Two
buckets capture the gradient that matters in practice.

The mod gate uses the same bot-defense reasoning as content
moderation (§10,
[governance.md §7](governance.md#7-the-mod-gate)).
Without it, a coordinated push could drag a baseline-pair
threshold to trivially low values and weaponize the loosened
parameter.

Both pairs are **self-amending**: each bucket's own thresholds
are governed by that bucket's rule. Defaults exist to bootstrap;
they are not fixed rules.

---

## What this doc is not

- **Not the governance primitive.** Eligibility, weight
  functions, threshold policies, outcome semantics, the two
  vote shapes, and multi-gate decisions live in
  [governance.md](governance.md).
- **Not the moderation primitive.** Mechanics of moderation
  Proposals, the cascade outcomes, the content-side mod-gate
  rule, and the platform-guidelines reference live in
  [moderation.md](../instances/moderation.md).
- **Not the Proposal node spec.** Proposal creation, properties,
  edges, authorship, and lifecycle live in
  [proposal.md](../instances/proposal.md).
- **Not federation.** Cross-instance Network reconciliation is
  Q15-deferred ([open-questions.md](../open-questions.md)).
- **Not the User node spec.** See [user.md](user.md).
- **Not the Memgraph schema.** Concrete property types, defaults,
  and indexes live in
  [graph-data-model.md](../implementation/graph-data-model.md).
