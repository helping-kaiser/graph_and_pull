# Open Questions

All unresolved design questions across the project, consolidated here.

Each entry is self-contained enough that a fresh reader (human or AI)
can engage without having to read every other doc. Pointers from the
origin docs link here; as questions are resolved, the answer moves
into the relevant design doc and the entry below is removed.

**Scope.** Design questions only — things we have not decided. Pure
implementation TODOs, known-outdated docs, and tasks on a roadmap are
out of scope. (`api-spec.md` is flagged as pending rewrite; it's not
a design question, it's a rewriting task, so it's not listed here.)

---

## Resolution order

The questions below are listed in **topic** order (roughly: ranking
primitives → onboarding → data model → chats → policy). The
**resolution** order is different — some questions genuinely can't be
answered until others are. Work them in roughly the order below;
within a phase, order is flexible.

| Phase | # | Question | Why here |
|:---:|:---:|:---:|---|
| 1. Collectives v1 | 1 | **Q21** | Collective role catalog — how role strings are introduced, scoped, and bound to powers. Current docs assume role strings exist without specifying a catalog mechanism. The granularity range (corporate hierarchy vs. household consensus) is the design constraint. Surfaced from the post-tightening audit; in active resolution. |
| 2. Economics workstream | 2 | **Q20** | Economics primitive — ad-revenue distribution mechanism, the ledger database home, and the "pull marketing" vocabulary anchor. The dedicated post-audit workstream. Settling it may also supply inputs to Q16 (S's derivation) and reopen the stake-gating option rejected for Q19. |
| 3. Sort fallback | 3 | **Q16** | Derivation of `S(t)`, the intrinsic per-node scalar that breaks ties at the bottom of the sort cascade. Q2 settled the rest of the math but left S's inputs open. The user has flagged S's inputs as part of economics, so this phase is naturally informed by Q20. |
| 4. Federation phase | 4 | **Q15** | Identity reconciliation across separately-running instances for handle-based and per-creation node types. Type 1 nodes (hashtags) federate for free per Q14; Types 2 and 3 need a protocol; cross-instance bootstrap and integrity raise further sub-questions. Deferred until federation becomes concrete. |
| 5. Governance v1.x | 5 | **Q19** | Bot-resistant non-arbitrary quorum for Network-scope governance. PR-05 shipped dual-quorum (fractional bar + absolute floor) as the v1 compromise; the absolute floor itself is a static parameter and the fractional bar's denominator is still bot-inflatable. A self-calibrating mechanism that does not depend on identity verification or economic gating is unsolved. Defer until the network's bot-density evidence justifies replacing the parameter pair. |

As questions resolve, their blocks disappear from below and their
rows disappear from this table. The table stays in place until all
questions are closed.

**Resolved:**

- Q7 — see [data-model.md §"author_id + author_type"](implementation/data-model.md#author_id--author_type--discriminator-not-foreign-key).
- Q8 — see [chats.md §10](instances/chats.md#10-moderation) and [governance.md §8](primitive/governance.md#8-instances).
- Q3 — see [graph-model.md §3](primitive/graph-model.md#3-edge-categories) "What creates an actor edge — stances-not-events".
- Q2 — see [feed-ranking.md §3-§4](primitive/feed-ranking.md#3-per-edge-composition-along-a-path) (per-edge composition, parallel tracks, taint rule, sum collapser) and [graph-model.md §6](primitive/graph-model.md#6-dimension-semantics) (dim1/dim2 unification, filtering vs. graph math). S's intrinsic derivation deferred — tracked as Q16.
- Q11 — see [feed-ranking.md §3.6–§3.7](primitive/feed-ranking.md#36-bot-resistance-via-the-0-0-severance-edge) (`(0, 0)` severance edge, cascading severance, redemption) and [feed-ranking.md §5](primitive/feed-ranking.md#5-algorithm) (zero-jail banishment of `h(t) = 0`). Self-discovery and return-pathway UX surfaces are tracked as forward sub-questions Q12 and Q13.
- Q12 — see [feed-ranking.md §3.8.1](primitive/feed-ranking.md#381-severance-discovery--the-inbound-side) (severance discovery via inbound self-query, trust-weighted reading) and [feed-ranking.md §3.8.2](primitive/feed-ranking.md#382-bot-cluster-identification--auto-detection-from-path-patterns) (auto-detection of bot-bridge nodes via delta-funnel path patterns, with path-length-aware action guidance). Cause identification is the auto-detect's job, complemented by the community posts in §3.8.3.
- Q13 — see [feed-ranking.md §3.8.4](primitive/feed-ranking.md#384-severance-redemption--the-outbound-side) (severer-side redemption surface, delta-funnel check on the redeeming node's outbound) and [feed-ranking.md §3.8.5](primitive/feed-ranking.md#385-self-redemption-posts) (self-redemption posts via the same `bot-defense` hashtag mechanism, surfaced in the severer's "review severed accounts" view).
- Q14 — see [data-model.md "Node identity strategies"](implementation/data-model.md#node-identity-strategies) (three-strategy framework: content-addressed UUIDv5 for canonical-string nodes like Hashtag, random UUID + UNIQUE handle for User/Collective, random UUID alone for per-creation nodes). Hashtag IDs are now content-addressed so independent creations of the same canonical name converge on one node. Cross-instance federation reconciliation for Types 2 and 3 is deferred as Q15.
- Q6 — see [invitations.md "Default values and customization"](primitive/invitations.md#default-values-and-customization). Defaults are `(+0.5, +0.5)` on both edges; both inviter and invitee choose their own outgoing edge during the invitation flow. The doc walks through the asymmetric-friend example (`(+1, -1)` on the invitee side as a deliberate "love them, not their content" stance that lets a later second edge dominate the feed).
- Q4 — see [feed-ranking.md §7](primitive/feed-ranking.md#7-time-and-recency). Time decay anchors on the **reactor edge's top-layer age** (the last actor edge in the path), applied as a scalar `f(Δt)` multiplier alongside `d(R)` to all four metrics (`h, i, j, k`). Default exponential with **30-day half-life**, frontend-tunable. Intermediate edges don't decay — silence on a relationship edge is not stance revocation. Post-node age has no separate decay — the authorship edge is itself a reactor edge and ages with the post, so old-with-no-engagement decays naturally and old-with-fresh-engagement resurfaces via fresh reactor-edge layers. Worked cold-start example in §7.3 shows the math.
- Q1 — see [graph-model.md §8](primitive/graph-model.md#8-append-only-history-edges). Layer count, layer timestamps, and the sequence of past edge values are **not ranking inputs**. They are metadata for audit, history, and UI surfaces (e.g., a "this edge has been revised N times" indicator, or a stale-edge prompt). Ranking sees only the top layer of each edge — the user's current expressed stance. Rationale: introducing layer-count amplification would let the system infer intent from interaction frequency, in tension with both **stances-not-events** ([graph-model.md §3](primitive/graph-model.md#3-edge-categories)) and the user-controlled-ranking principle. Edge cases like "two friends with identical edges but very different real-world contact frequency" are explicitly not auto-resolved by the system; users update stances reactively (similar to pruning a stale subscription list) rather than the system inferring from behavior.
- Q5 — see [feed-ranking.md §8](primitive/feed-ranking.md#8-the-already-seen-filter). The seen-list is a per-viewer set of content UUIDs treated as **another input to the feed-ranking computation**, alongside `R`, `d(R)`, `f(Δt)`, and the §5.2 friend-author-boost. Pre-rank exclusion (perf win — already-seen content never enters the math). New activity on a seen post does **not** resurface it; the new comment/reaction is independently rankable as its own node. Storage location is the viewing user's choice — backend-side `user_view_log` table in Postgres is the central frontend's default ([data-model.md](implementation/data-model.md)), but self-hosted clients/miners can keep the same data locally and pass it to the calculator (the math is the same regardless of where the JSON came from). Default frontend rule for "seen": every content item that passes through the viewport during a render. Frontend batches and flushes on natural checkpoints (batch-fill, scroll pause, app close); cache-clear before flush is an accepted small loss-window. Default 1-year compaction bounds storage at ~7 MB per active-user-year; the trade-off (a resurging old post will reappear if its view-log entry has been compacted) is documented and treated as acceptable feed character. No privacy-concealment story — viewing history is no more sensitive than reaction history per the network's transparency posture; "history" becomes a UI feature using the same data.
- Q10 — reframed as a side note rather than an open design question. See [layers.md "Side note on long-term storage"](primitive/layers.md#side-note-on-long-term-storage). Typical actor behavior bounds layer accumulation tightly — people update an edge a handful of times over its lifetime, not hundreds, and node properties change even less frequently. The corner cases that *would* accumulate substantial history (e.g., a decades-old company restructuring through CollectiveMember edges) are precisely the ones where preserving history has value. If a real instance ever runs into storage pressure, compaction-friendly approaches that respect the no-silent-deletion principle exist — but it's an implementation-time decision contingent on real data, not a design-time one to settle preemptively.
- Q9 — see [moderation.md](instances/moderation.md) and [network.md](primitive/network.md). Authorization for redaction runs through community-driven Network governance: any User authors a Proposal classifying content as `illegal`; threshold-cross requires at least one moderator's positive vote (the gate), ≥2/3 of cast votes in favor, and a low community quorum; threshold-cross triggers the [layers.md §5](primitive/layers.md#5-deletion-policy) redaction cascade. External pressure (court orders, etc.) doesn't bypass the mechanism — it prompts a moderator to start the same Proposal, which the community completes. Pathological corner cases (all moderators compromised) fall under the federation/forking exit per Q15.
- Q17 — see [feed-ranking.md §3.1](primitive/feed-ranking.md#31-which-edges-contribute-factors). No `Content → Author` back-edge exists or is added; content actor edges terminate at the content node and contribute only to ranking that content. The "I liked Alice's last three posts, so show me more Alice" intuition is supported by an explicit follow gesture, not inferred from post-affinity — that inference would be exactly the behavior-to-edge translation [graph-model.md §3](primitive/graph-model.md#3-edge-categories) (stances-not-events) rules out. Back-edge variants (with-cap, with-weight-discount, gated-on-reciprocation, propagate-to-author-only) each failed against either bot-bridge amplification or the actor-only-factor symmetry of §3.1, or both. A frontend may surface a follow-prompt after observed repeated engagement, but this is a UX nudge, not a graph mechanism, and is not added prophylactically — revisit only if feed-quality data shows the gap matters.
- Q18 — see [feed-ranking.md §3](primitive/feed-ranking.md#3-per-edge-composition-along-a-path) (simple-paths invariant — every path is vertex-simple, enforced via a per-path visited set; bidirectional topologies like mutual user edges, junction approval pairs, and `:BEARER` pairs would otherwise admit cyclic paths where the same intermediate's mediating role multiplies into the product without conveying new information) and [feed-ranking.md §4.1](primitive/feed-ranking.md#41-path-contribution-and-distance-decay) (single-transit-cap rejected — for 100 paths `U → Aᵢ → B → t` the sum factors as `d(3) · s(B → t) · Σᵢ s(U → Aᵢ) · s(Aᵢ → B)`, a clean product of "network-aggregate endorsement of `B`" times "`B`'s stance on `t`," which is trust propagation working correctly; bot-bridge amplification is already handled by severance + delta-funnel auto-detection in §3.6–§3.8, and `d(R)` already calibrates direct-vs-indirect, making 100 R=3 paths beating one R=2 path the intentional default). One-line entry added to [invariants.md "Ranking"](primitive/invariants.md#ranking) for discoverability.

---

## Q16 — Derivation of `S(t)`, the intrinsic node scalar

**Where it shows up:** [feed-ranking.md §2](primitive/feed-ranking.md#2-parameters) (S in the variable table) and [feed-ranking.md §5](primitive/feed-ranking.md#5-algorithm) (S as the final fallback after the h-cascade tie-breakers)
**Status:** open (forward sub-question of Q2)

### Context

The feed-ranking algorithm uses `S(t)` as a per-node intrinsic
scalar: the deepest fallback in the sort cascade. When `h`,
`h+i`, `h+i+j`, and `h+i+j+k` all tie, `S` decides the order.
The Q2 resolution settled the rest of the math but explicitly
left `S`'s derivation open.

### The question

What inputs feed `S(t)`?

The "intrinsic" framing is loose — nothing in CoGra is
universally intrinsic; everything is derivable from graph state
relative to a viewing user. Candidate inputs include the node's own
authorship-edge age, the node's neighborhood density, the node
type itself, or composite measures over the local subgraph. The
choice affects how ties resolve in sparse graphs, on cold-start
viewing users, and for users whose default values produce many exact
ties (e.g. integer `+1/0/-1` interaction styles).

### Constraints (from established principles)

- **No AI ranking.** `S` must be derivable from graph state, not
  from a learned model.
- **Append-only.** `S` is a derived value; it can be recomputed
  from the source data. It does not layer.
- **Per-viewer.** `S` is per-viewer, not globally intrinsic.
- **Rare in practice.** The cascade only triggers on strict
  equality, so `S` is the deepest fallback. Whatever derivation
  is chosen does not need to be cheap to compute on every node;
  it is computed only for the small set of candidates that
  reach this cascade depth.

### Options considered

None worked out yet.

### Related

Q2 (resolved — sets up the cascade that `S` terminates).

---

## Q15 — Cross-instance federation: identity reconciliation for handle-based and per-creation nodes

**Where it shows up:** [data-model.md "Node identity strategies"](implementation/data-model.md#node-identity-strategies) (Type 2 and Type 3 federation notes)
**Status:** open (deferred — federation phase)

### Context

The Q14 resolution settled three identity strategies in the data
model, with very different federation properties:

- **Type 1 — canonical-string identity, content-addressed
  UUIDv5** (Hashtag). Federates by construction when forks
  intend to share the namespace: the same canonical name
  produces the same UUID across any instance or fork. Forks
  that intend to diverge implicitly create incompatible
  hashtag IDs — the namespace UUID is committed forever the
  moment the genesis migration runs, so a fork keeping it
  inherits the shared namespace, and a fork rotating it
  breaks compatibility for every existing tag.
- **Type 2 — handle-based identity, random UUID + UNIQUE handle
  per instance** (User, Collective). Within an instance, the
  UNIQUE constraint prevents collision. Across separated
  instances, instance A's `@alice` and instance B's `@alice`
  have different UUIDs and could be the same person, two
  different people, or one impersonating another.
- **Type 3 — per-creation identity, random UUID alone** (Post,
  Comment, ChatMessage, Chat, Item, junction nodes). Within an
  instance, every creation is a distinct node. Across instances,
  cross-references (e.g. a post in instance A linked from
  content in instance B) require translation between local
  identities.

Type 1 is solved. Types 2 and 3 are open for any future
federation between Cogra instances.

### The question

When two separately-running instances begin to exchange data —
through a federation protocol, partial sync, or content embedded
in one another — how do their identity spaces reconcile?

Specifically:

- **Type 2 reconciliation (handles).** Instance A's `@alice` and
  instance B's `@alice`: same person or two? Manual claim by the
  owner with a cryptographic key? Inferred from external
  signals? Aliased explicitly via a graph mechanism? Always
  treated as different unless explicitly merged?
- **Type 3 reconciliation (per-creation).** A post in instance A
  referenced from instance B: does it get a "shadow" UUID in
  B's namespace? Is the original UUID preserved with an
  instance-prefix? How does cross-instance authorship
  attribution work?
- **Federation protocol surface.** How do instances discover
  each other, agree on synchronization scope, and handle
  disagreements (e.g. instance A says "Bob is a bot, severed,"
  instance B disagrees)?
- **`:Network` singleton ID distribution.** Within an instance
  the singleton's `id` is a one-query lookup, but every client
  composing a Network-scope Proposal needs that UUID up front.
  Across instances, each `:Network` has its own UUID; a
  federation protocol has to decide whether singleton IDs are
  discoverable, signed, or pinned to instance metadata. See
  [network.md §2](primitive/network.md#2-creation) and
  [graph-data-model.md](implementation/graph-data-model.md).
- **First-user serialization across instances.** Within one
  instance, the bootstrap migration is the only path that
  writes the genesis User, so concurrent registration cannot
  race ([network.md §2](primitive/network.md#2-creation),
  [auth.md](implementation/auth.md)). Two separately-running
  instances independently mint their own genesis users; if
  they later federate, the federation protocol has to decide
  what "the genesis user" means when both instances have one.
- **Hashtag UUIDv5 backend integrity.** Hashtag IDs are
  derived from a namespace UUID and the canonical name. The
  derivation runs in the backend, with no per-row check that
  `id == UUIDv5(namespace, name)`
  ([data-model.md](implementation/data-model.md)). Within one
  honest instance, backend discipline is sufficient. Federated
  exchange of hashtag references requires deciding whether
  instance B accepts instance A's hashtag IDs on trust, recomputes
  them, or expects an attestation (binary hash, signed build, or
  similar) that A computed the UUID the agreed way.

### Constraints (from established principles)

- **No central authority.** Per CLAUDE.md, anyone can fork and
  self-host. Federation cannot depend on a central registry.
- **Append-only.** Per [layers.md](primitive/layers.md),
  reconciliation cannot retroactively rewrite local state. New
  layers / new edges may be appended; old ones stay.
- **Transparency.** Reconciliation choices (alias, claim, merge)
  leave a visible trace on-graph.
- **Severance is local to the severing community.** Per
  [feed-ranking.md §3.7](primitive/feed-ranking.md#37-cascading-severance-and-redemption), the math is
  per-viewer. Federation should not import or export severance
  state automatically.

### Options considered

None worked out yet. Surfaced as possibilities only:

- **Cryptographic claim / proof.** Users hold a key pair;
  claiming an identity across instances requires signing with
  the private key. Solves the "is this the same person?"
  question but raises key-management questions and introduces a
  cryptographic dependency.
- **Aliasing edges.** A new edge type that maps "instance A's
  `@alice` → instance B's `@alice`" as an explicit graph-level
  claim. Requires consensus on what counts as authoritative
  aliasing.
- **Always-distinct.** Instances treat each other's identities
  as separate. Federation only allows reading, not merging.
  Loses the cross-instance same-person semantics but is the
  simplest model.
- **Hybrid (per-strategy).** Different reconciliation rules for
  Type 2 (cryptographic claim) and Type 3 (instance-prefix
  cross-references).

### Related

Q14 (resolved — sets up the per-type strategies that this
question completes for the cross-instance case),
[feed-ranking.md §3.7](primitive/feed-ranking.md#37-cascading-severance-and-redemption) (cluster
severance — local to the severing community per principle, but
federation could change this).

---

## Q19 — Bot-resistant non-arbitrary quorum for Network-scope governance

**Where it shows up:**
[governance.md §3 "Petition-style tally and dual quorum"](primitive/governance.md#petition-style-tally-and-dual-quorum-network-scope-only)
(the "Known limitation" paragraph),
[network.md §3](primitive/network.md#3-graph-side-properties)
(the dual-quorum parameter pairs).
**Status:** open

### Context

Network-scope governance uses a petition-style tally
(positive votes only) under a dual-quorum gate:
`positive_count ≥ min(P × |active members|, K)`. Both `P` and
`K` are amendable `:Network` properties. The mechanism
eliminates the passive-"no" veto (a real improvement over
bidirectional tallies that bot-cast `no`-votes can lock
indefinitely), but the underlying sybil problem is
unresolved.

### The question

What is the right denominator-and-floor mechanism for
Network-scope governance under unbounded membership and no
identity verification?

The two-bar v1 leaves two known holes:

- **Denominator inflation.** A bot account that exists as an
  active member (any outgoing actor edge inside
  `active_threshold_days`) inflates the fractional bar's
  denominator without ever needing to vote. At scale the
  fractional bar becomes unreachable independently of how
  many real positive votes accumulate.
- **Static floor.** The absolute bar `K` is a fixed number
  that needs periodic re-tuning as the network grows. It
  bounds damage from inflation but does not eliminate it,
  and its correct calibration over long horizons is
  unsettled.

### Constraints (from established principles)

- **No identity verification.** Per CLAUDE.md ethos,
  anyone can fork and self-host; identity-gated voting is
  off the table.
- **No AI in governance signal.** Per CLAUDE.md, the
  graph's signal and the governance arithmetic must be
  derivable from graph state, not from learned models.
- **All numeric parameters are amendable.** Per
  [governance.md §2.4](primitive/governance.md#24-threshold-policy),
  any value used in the tally is itself a `:Network`
  property amendable via the same primitive.
- **Append-only.** Per [layers.md](primitive/layers.md),
  no mechanism may delete graph structure to "expel" bot
  accounts.

### Options considered (none chosen)

- **Vote-active denominator.** Denominator = Users with ≥1
  positive Shape A vote in a recent window. Rejected
  because a bot pays one yes-vote to enter the window and
  then sits silent for the window duration, inflating the
  denominator without contributing.
- **Web-of-trust denominator.** Denominator = Users with
  ≥M inbound actor edges from accounts older than D days.
  Uses graph structure as the sybil filter. Promising but
  introduces a recursive bootstrap question and a "voting
  class" of trusted-enough accounts that newcomers must
  earn into.
- **Stake / token gating.** Defer to the economics layer;
  out of scope for the governance primitive.
- **Vote burn / quadratic cost.** Per-vote energy or
  cooldown cost that grows with vote frequency. Introduces
  a friction mechanism that punishes engaged real users
  alongside bots.
- **Sliding-window proportional.** Denominator = average
  unique-voter count across recent proposals. Auto-
  calibrates to engagement but is structurally similar to
  the vote-active variant and shares its gaming.

### Related

[governance.md §3](primitive/governance.md#petition-style-tally-and-dual-quorum-network-scope-only)
(the petition mechanism and dual-quorum v1),
[feed-ranking.md §3.6](primitive/feed-ranking.md#36-bot-resistance-via-the-0-0-severance-edge)
(severance as the existing graph-level bot defense; not
directly applicable to governance tally but informs the
"graph as sybil filter" candidates).

---

## Q20 — Economics primitive: distribution, ledger home, vocabulary anchor

**Where it shows up:**
[README.md "Fair economics"](../README.md) and the equivalent
section of [CONTRIBUTING.md](../CONTRIBUTING.md) (ad-revenue
claim with no primitive landing),
[CLAUDE.md](../CLAUDE.md) (dual-database never-rule with no slot
for transactions or payouts).
**Status:** open (post-audit dedicated workstream)

### Context

The meta documents promise a graph-driven economics: ad revenue
distributes across the network according to the graph and its
weights; bot clusters earn nothing. No primitive or
implementation doc yet says how. This block bundles the
sub-questions that the upcoming economics workstream will need
to answer together; tackling them in isolation risks a design
that satisfies one and breaks another.

### The questions

#### Q20.1 — Ad-revenue distribution mechanism

Who computes the distribution and when (per-impression,
per-tally, per-epoch); how revenue ingress is modeled (advertiser
deposit, escrow, immediate pass-through); which graph quantity
maps to payout (`h(t)` aggregated, path-products, authorship,
some new derived quantity). The audit identifies this as the
single largest pre-economics gap.

#### Q20.2 — Economics ledger database home

The dual-database rule in CLAUDE.md splits topology
(Memgraph) from display content (Postgres). Transactions, ledger
entries, and payout records fit neither cleanly. Options:
co-locate in Postgres as a third schema, introduce a third store
purpose-built for ledger semantics, or absorb into the graph as
edges with monetary semantics. Each option has cascading
consequences for the dual-database invariant.

#### Q20.3 — "Pull marketing" vocabulary anchor

"Pull marketing, not push marketing" appears only in meta docs
([README.md](../README.md), [CONTRIBUTING.md](../CONTRIBUTING.md))
with no anchor in a primitive or implementation doc. Either the
phrase is canonical for the economics primitive and needs a
landing pad once the mechanism is designed, or it is meta-only
framing that should be tightened. Resolved together with Q20.1
once the distribution shape is decided.

### Constraints (from established principles)

- **No AI in economics.** Per [CLAUDE.md](../CLAUDE.md), the
  graph and its weights drive distribution; learned models are
  out.
- **No central authority.** Anyone can fork and self-host; the
  economics mechanism must work without a privileged operator.
- **Append-only.** Per [layers.md](primitive/layers.md), no
  economics flow rewrites graph history. New layers, new edges,
  and new node types may be added; old ones stay.
- **Dual-database split.** Per [CLAUDE.md](../CLAUDE.md),
  topology lives in Memgraph and display content in Postgres.
  Q20.2 is the explicit question of whether the ledger extends
  this split or breaks it.

### Options considered

None worked out yet — this is the workstream that will produce
them.

### Related

[Q16](#q16--derivation-of-st-the-intrinsic-node-scalar) (`S(t)`
inputs are flagged by the user as "part of economics" — Q20 may
supply them),
[Q19](#q19--bot-resistant-non-arbitrary-quorum-for-network-scope-governance)
(the rejected "stake / token gating" option was deferred to the
economics layer; once Q20 settles, that option can be
re-evaluated).

---

## Q21 — Collective role catalog: how role strings are introduced, scoped, and bound to powers

**Where it shows up:**
[collectives.md §3](instances/collectives.md#collectivemember)
(the open-ended `role` string on CollectiveMember),
[collectives.md §8 "Per-decision-type instances"](instances/collectives.md#per-decision-type-instances)
(eligibility predicates like `role = CEO` referenced inside
`Collective.governance_rules.*` entries),
[collectives.md §8 "Example configurations"](instances/collectives.md#example-configurations)
(corporate / household / co-op tables that name `CEO`,
`founder`, `band_lead`, `social_media_intern`, etc. as if from a
catalog that does not exist on the graph).
**Status:** open

### Context

A Collective's social contract relies on role strings to scope
powers — only a `CEO` can fire a worker, only a `founder` votes on
ownership changes, only a `band_lead` writes promotional posts as
the band. The docs say role strings live in two places:

1. **As an assignment** — the `role` property on a
   CollectiveMember junction (§3).
2. **As a power** — referenced inside the `eligibility` clause of
   a `Collective.governance_rules.*` property (§8).

What the docs do **not** say is **where the catalog of valid role
strings lives, how new roles are introduced, and how a member's
role assignment is validated against that catalog (if at all)**.
The example tables read as if there is some authoritative
`Collective.roles = ['CEO', 'founder', ...]` list, but no such
property is declared anywhere on the graph.

Open sub-questions:

- **Catalog representation.** Is the set of valid role strings an
  explicit property on the Collective (a layered list of strings,
  amendable via Proposal)? Or implicit — derived from the union
  of strings that appear in any `governance_rules.*` entry plus
  the strings assigned to any CollectiveMember? Implicit avoids a
  second source of truth but lets typos silently create new roles;
  explicit forbids typos but adds a separate amendment surface.
- **Role-creation procedure.** If explicit, what is the Proposal
  shape for "add a new role to the catalog"? Is adding the role
  string atomic with adding at least one governance_rules entry
  that references it, or are the two edits separate?
- **Role-removal procedure.** Removing a role from the catalog
  while members still hold it — does the member's `role`
  layered-property hold the now-defunct string until amended, or
  is removal blocked while any holder exists?
- **Granularity range.** Some collectives need fine-grained
  hierarchies (a corporation with a dozen distinct roles and
  carefully tiered powers); others need very loose vocabularies
  (a household where the only role is "member" and consensus
  rules everything). The mechanism has to accommodate both
  extremes without forcing either to do bureaucratic work it
  doesn't need.
- **Cross-Collective role semantics.** `CEO` in one Collective has
  no relationship to `CEO` in another — same string, different
  authority. Should this be explicit at the schema layer (each
  Collective owns its catalog) or just emergent from the fact
  that governance_rules live on the Collective node? The current
  framing is emergent; that may be fine, but the docs should say
  so.

### Constraints (from established principles)

- **No central authority.** Per [CLAUDE.md](../CLAUDE.md), each
  Collective owns its own social contract; there is no
  Network-level role registry.
- **Append-only.** Per [layers.md](primitive/layers.md), role
  catalog changes must be layered, not destructive — even role
  removals leave the prior catalog intact in lower layers.
- **Governance of governance.** Per
  [collectives.md §8 "Where governance rules live"](instances/collectives.md#where-governance-rules-live),
  amendments to governance properties run through the same
  Proposal mechanism they govern. The role catalog, if explicit,
  inherits this.
- **No primitive defaults.** Per
  [collectives.md §8 "No primitive defaults"](instances/collectives.md#no-primitive-defaults),
  Collectives must explicitly define their rules at creation —
  any role-catalog mechanism must not impose a default vocabulary.

### Options considered

None worked out yet. Initial sketches in PR discussion proposed
both implicit-catalog and explicit-catalog framings; neither
exercised against the granularity-range constraint above.

### Related

[Q15](#q15--cross-instance-federation-identity-reconciliation-for-handle-based-and-per-creation-nodes)
(federation will need a position on whether two Collectives at
different instances can meaningfully share a role vocabulary or
not — likely "no" by default, but Q21 closes that question for
single-instance use first).
