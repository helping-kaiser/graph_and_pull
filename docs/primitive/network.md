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

## 2. Membership and roles

Every User has a `network_role` graph property:

- **`member`** — every registered user, automatically. The default.
- **`moderator`** — a small set of users who gate platform-wide
  governance actions (see [moderation.md](moderation.md) for the
  gating rule).

`network_role` is a graph-side property on the User node, layered
per [layers.md](layers.md) — promotion and demotion preserve full
history.

Whether Collectives can be Network members or moderators is
deferred. For now, only Users carry `network_role`.

## 3. Genesis moderator

Each instance starts with **one hardcoded genesis moderator** set at
instance bootstrap. For the central instance run by the project that
is the project owner; a federated fork sets its own genesis. This is
the only step in the system that depends on out-of-graph authority —
every subsequent mod change runs through the governance described
below.

Bitcoin analogy: someone has to mine the genesis block. From there
it is community-driven.

## 4. Mod role changes via multi-sig Proposal

Adding or removing a moderator uses the standard Proposal mechanism
([governance.md §2.1](governance.md)):

- **Subject:** A Proposal targeting `User.network_role` of the user
  being promoted or demoted, with `proposed_value` set to the new
  role.
- **Eligibility:** All active Network members.
- **Threshold:** Multi-sig — **≥1 existing moderator's positive vote**
  plus **N community-member votes** (N is a parameter — defaults to
  refine, e.g. ≥30% of active members at >50% in favor).

The multi-sig is the bot defense:

- Bots can flood the community side but cannot bypass the mod-vote
  gate without compromising a real moderator.
- Removal works the same way — mods cannot be unilaterally removed
  by community alone (which would let bots strip honest mods), nor
  by other mods alone (which would let mods purge each other).

## 5. Network-wide governance

The Network is the eligibility-and-voting body for any platform-
scoped governance instance:

- Adding and removing moderators (§4 above).
- Content moderation classifications — see
  [moderation.md](moderation.md).
- Future platform-wide policy and parameter tuning.

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
  [moderation.md](moderation.md).)

## What this doc is not

- **Not the moderation primitive.** Mechanics of moderation
  Proposals, the cascade outcomes, and the platform-guidelines
  reference all live in [moderation.md](moderation.md).
- **Not federation.** Cross-instance Network reconciliation is
  Q15-deferred ([open-questions.md](../open-questions.md)).
- **Not the User node spec.** See [nodes.md](nodes.md).
