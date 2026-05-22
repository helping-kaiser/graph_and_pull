# Governance

CoGra uses **weighted role-based voting** as a recurring primitive.
Every governance decision — approving a new member, disavowing a
message, eventually electing a collective council — follows the same
shape: eligible actors cast weighted votes; a threshold policy
decides the outcome; the outcome is recorded as a state transition
on the graph.

This doc defines the primitive. Specific applications (junction
approval, chat moderation, future patterns) parameterize it for
their context.

---

## 1. Why a shared primitive

Voting recurs across the project. Instead of inventing a mechanism
per context, CoGra commits to one conceptual shape every governance
decision reuses:

- **One mental model.** Every governance flow is understandable as
  an instance of the same primitive.
- **Consistent append-only semantics.** Votes are layers; outcomes
  are state transitions on structural edges. No special storage, no
  special hidden logic.
- **No per-case re-invention.** A new governance need specifies
  four things (subject, eligibility, weights, threshold) and plugs
  into the primitive.

**Each application is a parameterization, not a new mechanism.**
Junction approval, chat-message disavowal, network moderator role
changes, content moderation, and `:Network` parameter amendments
all run on the same primitive — they differ only in the values
they pick for the components in §2 (subject, eligibility, weights,
threshold) and the carrier shape in §3. If a new governance need
arises, it gets parameters and slots in here, not its own
mechanism. The full list of current applications is in §8.

---

## 2. The five components

Every vote-based decision specifies the components below.

A single subject node can host **multiple coexisting governance
instances**, each scoped to a specific decision-type and
parameterized independently. A Collective may have one instance
for "fire worker" (1-of-1 from CEO) and a different one for
"remove board member" (2/3 of the board) — same node, different
instances, routed by the subject's role. See
[collectives.md](../instances/collectives.md) for the worked-out social-contract
patterns.

### 2.1 Subject

What's being decided. Always a graph object whose state can change:

- A junction relationship (e.g. a ChatMember approval).
- A structural edge's state-bearing dimension (e.g. a chat's stance
  toward a message).
- A node property (e.g. a chat's own `disavowal_threshold`) —
  governance of governance is in scope.

**What governance does NOT cover — actor sovereignty.** A User's
own node properties (`username`, profile fields) and their
outgoing actor edges are sovereign: the User changes them
themselves, with no vote and no eligibility check. Governance
applies to **shared** state (junctions, structural edges, and
properties on nodes that represent more than one actor — Chats,
Collectives, Items, Proposals). The
[redaction exception in layers.md §5](layers.md#5-deletion-policy) is the only path
by which someone outside the actor can alter sovereign content,
and only for illegal material with a visible trace.

#### How subjects are addressed

A vote edge needs a node endpoint — edges can't point at edges or
at properties. Each subject type has a natural node to address:

- **Junction relationship** — the junction IS a node. Votes point
  directly at it.
- **Structural edge state** — the edge's target is a node. Votes
  point at that target; when the tally crosses threshold, the
  system writes a new state layer on the edge. The edge itself is
  never the target of a vote.
- **Node property** — a **Proposal** node (see
  [nodes.md §2](nodes.md#2-content-nodes)) is created as the subject. It carries
  `target_property` and `proposed_value` as node properties, and a
  `:TARGETS` structural edge to the target node (see
  [edges.md §2](edges.md#2-structural-edges)). Votes point at the Proposal; when the
  tally crosses threshold, a cascade (see
  [graph-model.md §5](graph-model.md#5-junction-node-flows)) writes a new layer on the
  target property with `proposed_value`. Multiple Proposals
  targeting the same property coexist; each passes or fails on its
  own votes. Reverting a passed change requires a counter-Proposal —
  consistent with §6.

Whether the vote edge uses Shape A or Shape B (§3) is a separate,
per-application choice about whether personal sentiment stays
coupled to the vote. It is not determined by the subject type.

### 2.2 Eligibility

Who can vote. Always expressed as a condition on existing graph
state:

- "Active ChatMembers of Chat Y" — membership + approval pair
  active.
- "CollectiveMembers of Collective Z with role `shareholder`."
- "Any actor with an outgoing edge to X" — permissive.

Eligibility is evaluated at **tally time**, not vote time. A vote
from someone who becomes ineligible afterward drops out; a vote
from someone who becomes eligible later (e.g. a newly-approved
member) counts once their status flips.

### 2.3 Weight function

How each vote's contribution is scaled. Derived from properties on
the voter's eligibility junction:

- ChatMember: `role` (`admin` / `chat_mod` / `member`). Optionally
  a direct `voting_weight` property when the chat sets per-member
  weight explicitly instead of deriving it from `role`. The
  `chat_mod` label is chat-scope; do not confuse with the
  Network-scope moderator role (`User.network_role = 'moderator'`).
- CollectiveMember: `role` + `ownership_pct` combine into a
  composite. Optionally a direct `voting_weight` for collectives
  whose weight is not tied to equity (e.g. one-member-one-vote
  with role-based multipliers, or per-member negotiated weight).
- Future cases: whatever properties the junction exposes.

`voting_weight` is the escape hatch for any junction whose weight
doesn't naturally fall out of role + ownership. When present it is
read directly as the voter's weight; when absent the instance falls
back to whatever rule it defines over `role` and other properties.
See [nodes.md §3](nodes.md#3-junction-nodes) for the property declaration.

### 2.4 Threshold policy

What tally triggers the outcome. Possible shapes:

- Simple count (N or more affirmative votes).
- Percentage of eligible voting weight.
- Supermajority for irreversible decisions.
- Quorum + percentage (M% of eligible weight participates, N% of
  cast weight agrees).
- **Multi-gate** — two or more independent eligibility groups
  voting on the same subject; each gate has its own threshold,
  and the outcome triggers only when **all** gates cross.

Percentages scale with the voter pool; fixed counts don't. An
instance that picks fixed numbers has to defend why it won't need
re-tuning as the pool grows.

**Multi-gate decisions are a separation of powers.** When a single
subject is gated by two or more distinct eligibility groups —
neither alone can pass it — the structure is intentional: each
gate counters a failure mode the others cannot. The canonical
instance is Network moderator role changes
([network.md §9](network.md#9-mod-role-changes-via-multi-sig-proposal)):
a moderator gate (≥1 existing moderator's positive vote) prevents
community-only purges by bot floods or coordinated targeting; a
community gate (quorum + supermajority of active members) prevents
mod-only coups in which sitting moderators strip honest peers.
Either gate alone leaves a hole; both gates together close it.
Future decisions adopt the multi-gate shape when the trust model
demands more than one veto-bearing group.

**All numeric parameters are tunable via this same primitive.**
Role weights, quorum %, threshold % — every number is a node
property on the subject, not a hardcoded constant. Changing any of
them is done via a Proposal (see §2.1), voted on by the same
eligibility rules. Defaults exist to bootstrap; they are not fixed
rules.

### 2.5 Outcome

What state change happens when the threshold is crossed. Always a
new layer on a structural edge (state-bearing) or a new layer on a
node property. Never a deletion; always append-only. Cascades are
allowed — see [graph-model.md §5](graph-model.md#5-junction-node-flows).

---

## 3. The two vote shapes

Two edge shapes carry votes. Both are append-only; they differ only
in carrier.

### Shape A — actor edge from voter to subject

Used when the voter has **no eligibility junction** to vote
through. The voter creates an actor edge from their `User` (or
`Collective`) node directly to the subject; `dim1` carries the
position (positive = support, negative = oppose).

Two cases use Shape A, both because the voter has no junction
to vote through:

**Would-be bearer's self-claim to a new junction.** The bearer
of a new ChatMember / CollectiveMember / ItemOwnership has no
junction of that type yet — their own junction is the very
thing they're claiming. Their gesture is necessarily a
`User → junction` actor edge.

```
User_Bob -[dim1: +0.9, dim2: +0.8]-> ChatMember_Bob_ChatY
```

**Network-scope governance.** The Network has no per-member
junction — every User is a member by virtue of being on the
graph (see [network.md](network.md)) — so every member votes on
Network-wide Proposals from their User node directly. The
`User → Proposal` actor edge from
[edges.md §1](edges.md#1-actor-edges) carries the vote: `dim1`
is the voter's stance, the tally reads `sign(dim1)` for binary
outcomes. See [proposal.md §4](../instances/proposal.md#4-edges).

In both cases the vote IS the actor's own stance toward the
subject. Other actors may also write actor edges to the same
subject (e.g. `User_Alice → ChatMember_Bob_ChatY` for personal
sentiment about Bob's membership) — these are not approval
votes, just personal sentiment.

### Shape B — structural edge from eligibility junction to subject

Used when the voter has an **existing eligibility junction** to
vote through. The voter triggers; the system creates a
structural edge from the voter's junction to the subject,
`dim1` carrying vote direction.

This is the workhorse shape for chat-internal and
collective-internal governance:

**Junction approval and removal.** Each required approver of a
new junction casts a Shape B vote from their existing junction
of the same type for the same parent. For `CollectiveMember`
and `ItemOwnership`, the same edge serves the full lifecycle:
layer-1 with `dim1 > 0` admits, later layers shift stance, an
eventual `dim1 < 0` layer is the removal vote. See
[graph-model.md §5](graph-model.md#5-junction-node-flows).

```
CollectiveMember_Alice_CollZ -[dim1: +1, dim2: 0]-> CollectiveMember_Bob_CollZ    (admit, layer 1)
CollectiveMember_Alice_CollZ -[dim1: -1, dim2: 0]-> CollectiveMember_Bob_CollZ    (remove, layer 2)
```

`ChatMember → ChatMember` covers admission only. Chats made the
uniformity choice that all chat-internal disavowal routes
through a Proposal node (see "Disavowal" below and
[chats.md §10](../instances/chats.md#10-moderation)) — direct
disavowal edges would reinvent tally semantics and make
counter-Proposal reversal awkward. Collectives keep both shapes
available per the collective's social contract.

**Disavowal of content or members.** Chat-internal disavowal —
both Level 1 against a `ChatMessage` and Level 2 against a
`ChatMember` — routes through a Proposal. Votes flow from each
voter's `ChatMember` to the Proposal node as `ChatMember →
Proposal` Shape B edges. See
[chats.md §10](../instances/chats.md#10-moderation).

**Votes on Proposals targeting chat / collective properties.**
`ChatMember → Proposal`, `CollectiveMember → Proposal` — the
member casts their vote as an eligible chat / collective
member, not as a personal stance.

In all cases:

- Decouples the vote from the voter's personal sentiment. Their
  `User → User`, `User → ChatMessage`, or `User → ChatMember`
  edges are untouched.
- Voter identity is the eligibility junction, expressing "I
  vote as a member of this chat / collective," not "I
  personally dislike this content / person."
- Eligibility loss handled naturally: if the junction goes
  inactive, the vote drops from the tally (edge stays in
  history per §5 / [graph-model.md §8](graph-model.md#8-append-only-history-edges)).

### Choosing between A and B

Mechanically: use **Shape A** when the voter has no junction to
vote through (bearer self-claim to a new junction, Network-scope
governance); use **Shape B** when the voter has an existing
junction (every chat-/collective-internal vote, including the
approver votes that admit and later remove a junction holder).

A future case that doesn't fit either shape is a signal to add a
third shape to this doc, not to hack an existing one.

### Co-signed acts: threshold > 1 in either shape

When a graph-state change requires more than one party to
concur — N parties whose individual votes each contribute to the
same tally — the structure is **uniformly governance with
threshold > 1** in whichever vote shape fits. No additional
mechanism is introduced; the result is what we call a
**co-signed act**:

- The would-be change is materialized as a pending subject (a
  not-yet-active junction node, a not-yet-produced outgoing
  edge, a Proposal) so co-signers have something to vote on.
- Co-signers append vote layers in their respective shape until
  the threshold is reached.
- On threshold-cross, the outcome takes effect per §6: the
  junction activates, the outgoing edge is produced, or the
  Proposal's cascade fires.

The pattern recognizes that "N parties concur" and "governance
with threshold N" are the same primitive — there is no separate
"co-signature" concept. Three current consumers:

- **Multi-sig junction approval.** Shape A self-claim by the
  would-be bearer plus N Shape B approver votes from existing
  eligibility junctions of the same type. The junction stays
  pending until the threshold is reached; activation produces
  the approval edge. See
  [graph-model.md §5](graph-model.md#5-junction-node-flows).
- **Chat invitations.** A specific application of multi-sig
  junction approval: the chat's `join_policy` sets the threshold
  on the inviter / approver Shape B votes that admit a new
  `ChatMember`. See
  [chats.md §11](../instances/chats.md#11-joining-and-leaving-a-chat).
- **Collective act-as routing.** A member's gesture to produce
  the Collective's outgoing edge runs through the social
  contract's per-gesture rule; when the rule's threshold is `> 1`,
  the gesture is held in a pending state until enough authorized
  members co-sign, identical in shape to the multi-sig junction
  approval above. See
  [collectives.md §2](../instances/collectives.md#2-acting-through-the-collective).

A future actor that needs N-party authorization on its own
outgoing edges (a multi-sig bot, a council acting jointly)
parameterizes this same primitive rather than inventing a new
one.

---

## 4. Append-only throughout

- Votes are layers on their carrier edges. Never deleted.
- Changing your vote = new layer (same edge, new dimension values).
- Revoking = new layer with opposing `dim1`.
- History is always visible. An observer can see how vote
  distribution evolved over time.

---

## 5. Weight at tally time

When weights come from mutable junction properties (e.g. an admin
demoted to member), the question arises: does a past vote retain
its old weight or take the current one?

**CoGra's default: current weight at tally time.** Reasons:

- Consistent with "current state = top layer of underlying data"
  everywhere else.
- An ex-admin's past admin-weighted votes shouldn't retain leverage
  after demotion.
- Avoids snapshotting weights into each vote edge (duplicates data).

Specific applications can override this if they need vote-time
snapshot weights, but they carry the burden of explaining why.

---

## 6. When outcomes take effect

Outcomes are **triggered by new-vote threshold-crossings**. A tally
is computed only when a new or updated vote layer arrives on the
subject — not on any schedule, and not when the eligibility set
shifts in the background.

- Raw vote layers are written whenever any eligible actor casts or
  changes a vote.
- On each new vote event, the tally is computed over currently
  eligible voters' current top vote layers (§§2.2, 5). If the tally
  has crossed the threshold since the last outcome on this subject,
  a new state layer is written on the subject.
- Eligibility changes alone (members leaving, roles changing) do
  **not** trigger re-tallying. Past outcomes stand. Current
  eligibility only applies the next time someone actually votes on
  the subject.

### Cascade dispatch

When a tally crosses threshold the outcome layer is rarely the
only write. The triggering write may fan out into derived writes:

- A `dim1 < 0` layer on a `:APPROVAL` edge (a member disavowal
  outcome).
- Redaction markers across one or more property layers plus a
  Postgres-side archive row (the `'illegal'`-classification
  outcome — see [moderation.md](../instances/moderation.md)).
- A `(0,0)` layer on every active vote of an actor whose
  eligibility just dropped (eligibility-dropout cascade).
- A new property layer on a chat node (mid-epoch `Chat.epoch`
  rotation).
- A new ownership-edge layer (Item ownership transfer).

Generic to every cascade:

- **Synchronous.** The cascade fires inside the same
  service-layer transaction as the triggering write — never
  deferred to a worker, never a background sweep.
- **Archive-first.** When the cascade includes a Postgres-side
  archive write (the redaction cascade is the current case),
  the archive row is written **before** the graph mutation. A
  failed archive prevents the graph mutation; the inverse would
  leave the platform with a redacted layer and no archive copy,
  which the retention story in
  [retention-archive.md](retention-archive.md) refuses.
- **Full rollback.** Any step failure rolls back the entire
  transaction — including the triggering vote layer. From the
  graph's perspective the threshold cross never happened, and
  the voter who pushed it across sees the rollback as a write
  failure on their vote.

The dispatcher is one module in the API service layer; it
routes by trigger type to the correct fan-out sequence and
holds no Cypher or SQL of its own. See
[architecture.md "Service-layer transactions"](../implementation/architecture.md#service-layer-transactions)
for where the code lives.

### Tally serialization

A tally runs inside the same service-layer transaction as the
vote-layer write that triggers it, so the tally observes the
new layer and the cascade fan-out commits or rolls back as one
unit.

Two writers can target the same Proposal concurrently — two
voters pressing send at the same moment. **Tallies are
serialized per Proposal:** two tallies on the same Proposal
never run in parallel. The first acquires a per-Proposal lock,
runs to completion, observes the threshold (or not), and
commits; the second runs against the post-commit state. If the
first crossed threshold, the second sees the outcome layer
already written and does not fire the cascade twice.

The serialization primitive is Memgraph's per-node lock taken
on the Proposal node at the start of the tally transaction. If
a chosen Memgraph build turns out to lack the needed primitive,
a Postgres advisory lock keyed on the Proposal UUID is a valid
fallback — the service layer holds both pools and can take the
lock on either side. The doctrine (*tallies serialized per
Proposal*) fixes the requirement; the exact mechanism is an
implementation choice settled in
[architecture.md](../implementation/architecture.md).

True-simultaneous writes — two commits the chosen DB isolation
cannot serialize — are an open question for the storage
engines and are filed in
[open-questions.md](../open-questions.md). The lock primitive
above is sufficient under standard read-committed isolation;
weaker conditions would need re-spec.

### Eligibility-dropout cascade

When an actor's voting eligibility changes mid-flight — a
ChatMember demoted, a CollectiveMember whose junction is
revoked, a User whose `network_role` is downgraded — the
cascade handler appends a `(0,0)` layer on every **active vote
that actor cast on still-open Proposals** (Proposals whose
outcome has not yet crossed threshold).

The next tally on any affected Proposal reads `(0,0)` and
counts no contribution from this voter. The voter's prior
positive or negative stance stays preserved in the edge layer
history (§4) — the cascade neutralizes the live tally without
erasing the record of intent.

Scope: open Proposals only. Past outcomes already at threshold
are not revisited (per § "Why outcomes are sticky"). Votes
already at `(0,0)` get no redundant new layer.

The cascade fires from the same service-layer transaction that
flipped the eligibility, with the rollback semantics of §
"Cascade dispatch": if any neutralizing layer fails, the
eligibility flip rolls back too.

### Why outcomes are sticky, not continuously rendered

Consider a member who voted on 1000 past disavowals and then leaves
the chat. Under a naive "always match the current tally" model,
their exit could flip every past decision they were pivotal to —
and each of those thousand subjects would then need fresh votes
from remaining members to re-cross quorum. Governance would be
dominated by background churn, not by intent, and the graph's
history would be swamped by silent reverts.

CoGra's model instead: **once an outcome takes effect, the subject
stays in that state until a future vote event pushes it back across
the threshold.** To undo a decision, members actively cast new
votes — updating their existing vote edges or creating new ones for
members who hadn't voted before. Governance is an act, not a
background computation.

### No time-boxing

Votes stand until changed; there is no "voting ends at T". A
specific application that genuinely needs a time window is a new
design discussion (§11).

---

## 7. The mod-gate

A recurring component of Network-scope governance is the
**mod-gate**: for any Proposal in scope, at least one positive
vote from a User with `network_role = 'moderator'` must be
present in the tally before the outcome can take effect. The
mod-gate is a procedural validator, not a weighting.

**Invariant: mod weight = member weight = 1; mod is a gate, not
a weight.** A moderator's vote contributes exactly one
member's worth of weight to the tally — the same as every other
voter. The mod-gate sits *alongside* the member-weighted tally,
not on top of it.

**A moderator's positive vote counts in BOTH the mod-gate check
AND the member-tally arithmetic; it is not double-spent in any
other sense.** The mod casts `+1` once; that same vote opens the
gate *and* contributes its weight of `1` to the count tallied
against quorum and threshold. The "mod weight = 1" rule means
the moderator's contribution to the member arithmetic is
exactly one member's worth — nothing more — even though the same
vote also opened the gate.

The gate applies symmetrically in both directions of any
classification — setting `sensitive` or `illegal`, and
un-setting back to `normal` — and across every Network-scope
Proposal kind. Reasons each direction needs the gate:

- Without a mod gate on `sensitive`, a small coordinated group
  could flood-flag legitimate content, forcing endless
  re-moderation.
- Without a mod gate on `illegal`, bot networks could mass-vote
  redactions of legitimate content.
- Without a mod gate on un-classification, bots could strip
  moderation flags from legitimately-classified content.
- Without a mod gate on **moderator role changes**, the
  community alone could strip moderators — a coordinated push
  (bots or otherwise) removes honest mods at will.

The mod-gate is the specific instance of the §2.4 multi-gate
pattern that pairs a moderator-gate with a community-tally gate.
Either gate alone leaves a hole:

- **Mods alone** can purge each other — sitting-mod coup.
- **Community alone** can be coordinated against honest mods —
  flooded removal.

Both gates together close both failure modes. The full list of
Network-scope instances that share the mod-gate component is in
§8 — content moderation classifications, moderator role
changes, and `:Network` parameter amendments.

Substantive arithmetic (quorums, supermajority thresholds, the
exact pairs per instance) lives with each instance, not here.

### External demands enter as Proposals

A direct corollary of the mod-gate and the broader
governance-only authorization model: **there is no admin
escape-hatch.** Any external pressure on the platform —
court orders, regulator demands, law-enforcement notices,
next-of-kin requests, copyright takedown letters — enters the
graph through the same Proposal mechanism any User would use.
Typically a moderator files the Proposal (because they are the
party that received the demand and they understand the
classification), but the Proposal itself is an ordinary
moderation Proposal in [moderation](../instances/moderation.md)
or the relevant account-action path; the mod-gate and the
community-tally gate both still apply.

The platform commits to this *because* it is the property that
makes the transparency story load-bearing: every removal is
auditable on the graph, no party can edit silently, and a
pathologically captured moderation corps still requires
community participation. The federation/forking exit
([network.md](network.md)) is the final pressure release if a
particular instance's governance no longer reflects its
community.

---

## 8. Instances

### Existing

- **Junction approvals** — [graph-model.md §5](graph-model.md#5-junction-node-flows).
  Shape A self-claim by the would-be bearer plus N Shape B
  approver votes from existing eligibility junctions of the
  same type for the same parent. Threshold: N is per-policy
  (open = 0, single approver = 1, multi-sig = N). Same Shape B
  edges later carry removal votes (stance flipped).
- **Chat message disavowal** — [chats.md §10](../instances/chats.md#10-moderation).
  Shape B `ChatMember → Proposal` vote on a Proposal targeting
  the `ChatMessage` with `target_property = 'node'`,
  `proposed_value = 'disavowed'` (the whole-node-targeting
  sentinel, see
  [nodes.md "Whole-node targeting"](nodes.md#whole-node-targeting-the-node-sentinel)).
  Quorum + weighted-majority threshold. No separate outcome
  edge — the chat's stance is the existence of the passed
  Proposal.
- **Chat member disavowal** — [chats.md §10](../instances/chats.md#10-moderation).
  Shape B `ChatMember → Proposal` vote on a Proposal targeting
  the member's `ChatMember` junction with the same `'node'` /
  `'disavowed'` shape. Cascade writes a `dim1 < 0` layer on the
  existing `Chat → ChatMember` approval edge for the target.
- **Chat property and role changes** — [chats.md §10](../instances/chats.md#10-moderation).
  Shape B `ChatMember → Proposal` votes on `Chat.name`,
  `Chat.join_policy`, `Chat.epoch` (mid-epoch chat-key rotation,
  see [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism)),
  and `ChatMember.role`. Defaults vary by stakes; thresholds are
  themselves chat properties.
- **Collective governance (full social contract)** —
  [collectives.md](../instances/collectives.md). Membership
  changes (hire / fire / promote), property changes (`name`,
  `governance_rules`, `ownership_pct`), and any other
  decision-type the collective defines. A Collective hosts as
  many instances as its social contract specifies; each is
  parameterized for its own decision-type. Shape B
  `CollectiveMember → CollectiveMember / Proposal` for all
  internal votes.
- **Network moderator role changes** — [network.md §9](network.md#9-mod-role-changes-via-multi-sig-proposal).
  Shape A from the User node directly (no per-member Network
  junction exists). Multi-sig: ≥1 existing moderator's positive
  vote plus a community-quorum threshold. Two dispatch
  exceptions — the **moderator floor** of 1 and the
  **undemotable bootstrap mod** — refuse the outcome write even
  on a passed tally.
- **Content moderation classifications** — [moderation.md](../instances/moderation.md).
  Shape A from the User node directly. Mod-vote-required gate
  on every classification change (`sensitive` / `illegal` and
  un-classification back to `normal`); mod weight = member
  weight = 1.
- **`:Network` parameter amendments** — [network.md §11](network.md#11-amending-network-parameters).
  Shape A from the User node directly. Two amendment-rule pairs
  on the `:Network` singleton — a baseline pair for low-stakes
  parameters and a critical pair for parameters with destructive
  or platform-wide reach. Mod gate required for both.

Future cases get added here as they're designed.

---

## 9. Coexistence: multiple governance instances on a shared subject

A single node can be the subject of several governance instances
at once — each parameterized by its own eligibility, weight
function, threshold policy, and outcome. The instances act
independently, write to different state, and produce different
outcomes; they do not conflict because they aren't operating on
the same state.

The principle: **scope determines what the outcome writes.** A
chat-scope instance writes to chat-side state (a `Chat →
ChatMember` approval edge layer, or the existence of a passed
disavowal Proposal that the chat treats as its stance). A
Network-scope instance writes to node-level state (a
`moderation_status` property layer, or per-field redaction
markers under [layers.md §5](layers.md#5-deletion-policy)). The
two cannot conflict because the targets are different graph
objects even when the *node* is the same.

The canonical worked example is **chat-internal disavowal
alongside platform moderation**, both of which can apply to a
single `ChatMessage`:

- **Platform moderation** — Network-scope. Eligibility = every
  active Network member; mod-gate (§7) required.
  Classification-Proposal targets the message; `'illegal'`
  triggers the redaction cascade in
  [layers.md §5](layers.md#5-deletion-policy) (destructive in
  the sense that the message body's top layer is replaced with
  a redaction marker, leaving the message node and edges in
  place). See [moderation.md](../instances/moderation.md).
- **Chat-internal disavowal** — chat-scope. Eligibility =
  active `ChatMember`s of the chat hosting the message; weight
  by role; no mod-gate. Outcome is the chat's stance
  ([chats.md §10](../instances/chats.md#10-moderation)) — the
  chat moves away from the message; the message stays.

Both can pass independently, neither overrides the other:

- A chat-disavowed message is still `'normal'` at the platform
  level until the Network classifies it — the disavowal Proposal
  affects chat-side state only.
- A platform-redacted message stays in any chat that hasn't
  disavowed it — the redaction marker affects node-side state
  only.
- The two can also both pass; the message is then both
  redacted at the platform level *and* disavowed at the chat
  level. No collision; the writes go to different places.

A similar pattern holds for any future instance that operates on
a node already governed by another scope (e.g. a Collective
governing a member's `CollectiveMember` while the Network
classifies that member's profile — separate state, independent
outcomes). The shape generalizes: scope decides the state
written, so instances at different scopes never compete for the
same write.

---

## 10. Multi-candidate decisions

Decisions that pick from several candidates — council seats,
multiple property values to choose between, etc. — are expressed
as **N parallel binary Proposals**, one per candidate. Each
Proposal is voted on independently using the same governance
instance (same eligibility, weights, threshold). Every Proposal
that crosses threshold passes; that candidate takes office or
that property value is set.

Removal later (recall, term-end) is another Proposal targeting
the same role or property to revert it. No special lifecycle
machinery needed.

This pattern loses ranked-ballot information ("B over A"). Ranked
and multi-seat semantics aren't part of the primitive (§11). A use
case that genuinely needs them deserves its own design pass.

---

## 11. Out of scope

- **Secret ballots.** All votes are public on the graph. Privacy is
  achieved through content encryption elsewhere, not through hiding
  vote topology. A future case that genuinely needs secret voting
  is a new design discussion.
- **Time-boxed voting periods.** Votes today are open-ended; once
  cast they stand until changed. "Voting ends at T" is a new
  design.
- **Delegation / proxies.** No "proxy voter" mechanism. Adds a
  layer to eligibility rules and needs its own design.
- **Ranked, multi-seat, or budget-allocation ballots.** All votes
  are binary (support / oppose on a single subject). Ranked
  preferences ("B over A"), multi-seat allocations beyond parallel
  binary Proposals (§10), and proportional budget splits across N
  options aren't expressible in the current primitive. Use cases
  that genuinely need any of these deserve their own design pass.

These aren't refused — they're just not addressed by the current
primitive. Any of them would extend governance.md rather than
replace it.

---

## What this doc is not

- **Not a list of specific thresholds or weights.** Per-application.
- **Not an aggregation / caching spec.** How the system efficiently
  evaluates tallies is an implementation concern.
- **Not a roadmap.** When each governance feature ships is separate.
