# Network

The **Network** is the global community of every User on a CoGra
instance — the body that backs platform-wide governance: content
moderation, dispute resolution, and anything that affects the whole
instance rather than a specific chat or collective.

## 1. Distinct from Collective

A [Collective](../instances/collectives.md) is a small group with a
defined membership: a household, band, co-op, company. Membership is
explicit and approval-gated.

The Network is the opposite — the set of every User on the graph.
Membership is **automatic on registration**; there is no approval
gate; there is no "this band vs that band." It is one Network per
instance.

Federation across instances is a forward question — see
[open-questions.md Q15](../open-questions.md). Each instance has its
own Network until then.

## 2. The :Network singleton

The Network is represented on the graph by a **singleton `:Network`
node** that holds the instance's configuration parameters. There is
exactly one per instance.

It carries:

- Mod role-change quorum and threshold (`mod_role_change_quorum`,
  `mod_role_change_threshold`).
- Per-classification moderation quorums and thresholds
  (`moderation_sensitive_*`, `moderation_illegal_*`).
- Platform guidelines pointer and amendment thresholds
  (`guidelines_version`, `guidelines_hash`,
  `guidelines_change_quorum`, `guidelines_change_threshold`) —
  see [platform-guidelines.md](../instances/platform-guidelines.md).
- Eligibility-definition parameters (`active_threshold_days` — the
  recency window that makes a User count as an "active member" for
  governance tallies).
- Amendment-rule pairs that govern changes to the singleton's own
  parameters: a baseline pair (`property_change_quorum`,
  `property_change_threshold`) for low-stakes parameters and a
  critical pair (`critical_property_change_quorum`,
  `critical_property_change_threshold`) for parameters whose abuse
  has destructive or platform-wide reach. See §7.

All properties are layered, so every parameter change has a
preserved history. Each is amendable via a standard Proposal
targeting the property — same primitive as everything else (see
[governance.md §2.1](governance.md)); §7 below specifies which
amendment-rule pair gates which property. The full property list
and defaults live in
[graph-data-model.md](../implementation/graph-data-model.md).

The singleton exists so that platform-wide governance has a graph
node to target. Without it, statements like "Network parameters are
amendable via Proposal" are hand-waving — Proposals need a node.

## 3. Membership and roles

Every User has a `network_role` graph property:

- **`member`** — every registered user, automatically. The default.
- **`moderator`** — a small set of users who gate platform-wide
  governance actions (see [moderation.md](../instances/moderation.md) for the
  gating rule).

`network_role` is a graph-side property on the User node, layered
per [layers.md](layers.md) — promotion and demotion preserve full
history.

Whether Collectives can be Network members or moderators is
deferred. For now, only Users carry `network_role`.

## 4. Bootstrap

Each instance bootstraps with two pieces of out-of-graph state:

1. The **`:Network` singleton** is created with the default
   parameter values listed in
   [graph-data-model.md](../implementation/graph-data-model.md).
2. **One hardcoded genesis moderator** is set —
   `User.network_role = 'moderator'` for a configured user.

For the central instance run by the project, the genesis moderator
is the project owner. A federated fork sets its own genesis. These
are the only steps in the system that depend on out-of-graph
authority — every subsequent change to the singleton's parameters
or to any user's role runs through governance.

Bitcoin analogy: someone has to mine the genesis block. From there
it is community-driven.

## 5. Mod role changes via multi-sig Proposal

Adding or removing a moderator uses the standard Proposal mechanism
([governance.md §2.1](governance.md)):

- **Subject:** A Proposal targeting `User.network_role` of the user
  being promoted or demoted, with `proposed_value` set to the new
  role.
- **Eligibility:** all active Network members.
- **Threshold:** multi-sig — **≥1 existing moderator's positive vote**
  plus **`Network.mod_role_change_quorum`** of cast eligible-member
  votes, with **`Network.mod_role_change_threshold`** in favor.

The multi-sig is the bot defense:

- Bots can flood the community side but cannot bypass the mod-vote
  gate without compromising a real moderator.
- Removal works the same way — mods cannot be unilaterally removed
  by community alone (which would let bots strip honest mods), nor
  by other mods alone (which would let mods purge each other).

## 6. Network-wide governance

The Network is the eligibility-and-voting body for any platform-
scoped governance instance:

- Adding and removing moderators (§5 above).
- Content moderation classifications — see
  [moderation.md](../instances/moderation.md).
- Tuning the `:Network` singleton's parameters themselves
  (governance of governance) — see §7.

Each is a Shape B governance instance per
[governance.md §3](governance.md). Two consequences:

- **The eligibility carrier is the User node itself**, not a
  junction. Network membership has no separate gesture, so there is
  no `ChatMember`-/`CollectiveMember`-style junction to carry the
  vote. Vote edges run from the voter's User node to the subject.
  This relaxes the Shape B carrier rule for the Network case
  specifically — see governance.md for the relaxation.
- **Mod weight = member weight = 1.** Mods do not outvote the
  community; the "≥1 mod positive vote" rule is a gate, not a
  weighting. (Same rule applied to content classifications in
  [moderation.md](../instances/moderation.md).)

## 7. Amending `:Network` parameters

Two amendment-rule pairs gate changes to the singleton's own
properties, separated by stakes:

| Bucket   | Pair                                  | Quorum (default) | Threshold (default) | Mod gate | Governs |
|----------|---------------------------------------|------------------|---------------------|----------|---------|
| Baseline | `property_change_quorum`, `property_change_threshold` | 5%  | ≥2/3 | required | `moderation_sensitive_*`, `active_threshold_days`, the baseline pair itself |
| Critical | `critical_property_change_quorum`, `critical_property_change_threshold` | 10% | ≥3/4 | required | `mod_role_change_*`, `moderation_illegal_*`, `guidelines_change_*`, the critical pair itself |

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
moderation ([moderation.md §3](../instances/moderation.md)).
Without it, a coordinated push could drag a baseline-pair
threshold to trivially low values and weaponize the loosened
parameter.

Both pairs are **self-amending**: each bucket's own thresholds
are governed by that bucket's rule. Defaults exist to bootstrap;
they are not fixed rules.

## What this doc is not

- **Not moderation.** Mechanics of moderation Proposals, the
  cascade outcomes, and the platform-guidelines reference all
  live in [moderation.md](../instances/moderation.md).
- **Not federation.** Cross-instance Network reconciliation is
  Q15-deferred ([open-questions.md](../open-questions.md)).
- **Not the User node spec.** See [nodes.md](nodes.md).
