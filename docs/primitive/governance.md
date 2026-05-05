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
[redaction exception in layers.md §5](layers.md) is the only path
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
  [nodes.md §2](nodes.md)) is created as the subject. It carries
  `target_node_id`, `target_property`, and `proposed_value`. Votes
  point at the Proposal; when the tally crosses threshold, a
  cascade (see [graph-model.md §5](graph-model.md)) writes a new
  layer on the target property with `proposed_value`. Multiple
  Proposals targeting the same property coexist; each passes or
  fails on its own votes. Reverting a passed change requires a
  counter-Proposal — consistent with §6.

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

- ChatMember: `role` (admin / mod / member). Optionally a direct
  `voting_weight` property when the chat sets per-member weight
  explicitly instead of deriving it from `role`.
- CollectiveMember: `role` + `ownership_pct` combine into a
  composite. Optionally a direct `voting_weight` for collectives
  whose weight is not tied to equity (e.g. one-member-one-vote
  with role-based multipliers, or per-member negotiated weight).
- Future cases: whatever properties the junction exposes.

`voting_weight` is the escape hatch for any junction whose weight
doesn't naturally fall out of role + ownership. When present it is
read directly as the voter's weight; when absent the instance falls
back to whatever rule it defines over `role` and other properties.
See [nodes.md §3](nodes.md) for the property declaration.

### 2.4 Threshold policy

What tally triggers the outcome. Possible shapes:

- Simple count (N or more affirmative votes).
- Percentage of eligible voting weight.
- Supermajority for irreversible decisions.
- Quorum + percentage (M% of eligible weight participates, N% of
  cast weight agrees).

Percentages scale with the voter pool; fixed counts don't. An
instance that picks fixed numbers has to defend why it won't need
re-tuning as the pool grows.

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
allowed — see [graph-model.md §5](graph-model.md).

---

## 3. The two vote shapes

Two edge shapes carry votes. Both are append-only; they differ only
in carrier.

### Shape A — actor edge toward a junction

Used when the subject is **a relationship itself**. The voter
creates a regular actor edge toward the junction; `dim1` is their
position (positive = support, negative = oppose).

Example — approving a ChatMember:

```
User_Alice -[sentiment: +0.9, relevance: +0.8]-> ChatMember_Bob_ChatY
```

- The vote IS the actor's personal stance toward the junction.
- Fine because junctions rarely surface as feed content;
  conflating stance with sentiment has no cost.
- Used by: all existing junction approval flows.

### Shape B — system-created structural edge from eligibility junction to subject

Used when the subject is **content** (or anything the voter also
has a separate personal opinion about). The voter triggers; the
system creates a structural edge from the voter's eligibility
junction to the subject, `dim1` carrying vote direction.

Example — disavowing a ChatMessage:

```
ChatMember_Jakob_ChatY -[dim1: -1, dim2: 0]-> ChatMessage_X
```

- Decouples the vote from the voter's personal sentiment. Their
  `User → ChatMessage` edge is untouched.
- Voter identity is the eligibility junction, expressing "I vote as
  a member of this chat," not "I personally dislike this content."
- Eligibility loss handled naturally: if the junction goes inactive,
  the vote drops from the tally (edge stays in history).
- Used by: chat message disavowal, chat member disavowal.

### Choosing between A and B

Use **Shape A** when voting for/against the subject is the same as
liking/endorsing it. Junction approvals fit this.

Use **Shape B** when voting for/against shouldn't conflate with
personal sentiment. Content moderation fits this.

A future case that doesn't fit either shape is a signal to add a
third shape to this doc, not to hack an existing one.

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
design discussion (§9).

---

## 7. Instances

### Existing

- **Junction approvals** — [graph-model.md §5](graph-model.md).
  Shape A. Threshold: N actor edges from specified roles.
- **Chat message disavowal** — [chats.md §6](../instances/chats.md). Shape B.
  Quorum + weighted-majority threshold.
- **Chat member disavowal** — [chats.md §6](../instances/chats.md). Shape B.
  Higher quorum + weighted-supermajority threshold.
- **Chat property and role changes** — [chats.md §6](../instances/chats.md).
  Proposals on `Chat.name`, `Chat.content_privacy`,
  `Chat.join_policy`, and `ChatMember.role`. Defaults vary by
  stakes; thresholds are themselves chat properties.
- **Collective governance (full social contract)** —
  [collectives.md](../instances/collectives.md). Membership changes (hire /
  fire / promote), property changes (`name`, `governance_rules`,
  `ownership_pct`), and any other decision-type the collective
  defines. A Collective hosts as many instances as its social
  contract specifies; each is parameterized for its own
  decision-type. Eligibility, weights, and thresholds are all
  per-instance.

Future cases get added here as they're designed.

### Planned

- **Redaction authorization** — [open-questions.md Q9](../open-questions.md).
  When Q9 resolves, the takedown / redaction policy will land as
  another governance instance: subject = the layer or display row
  to redact, eligibility and threshold per the redaction policy.

---

## 8. Multi-candidate decisions

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
and multi-seat semantics aren't part of the primitive (§9). A use
case that genuinely needs them deserves its own design pass.

---

## 9. Out of scope

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
  binary Proposals (§8), and proportional budget splits across N
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
