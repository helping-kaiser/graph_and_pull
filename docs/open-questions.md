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
| 1. Sort fallback | 1 | **Q16** | Derivation of `S(t)`, the intrinsic per-node scalar that breaks ties at the bottom of the sort cascade. Q2 settled the rest of the math but left S's inputs open. Pure ranking math; no external dependency. |
| 2. Federation phase | 2 | **Q15** | Identity reconciliation across separately-running instances for handle-based and per-creation node types. Type 1 nodes (hashtags) federate for free per Q14; Types 2 and 3 need a protocol. Deferred until federation becomes concrete. |

As questions resolve, their blocks disappear from below and their
rows disappear from this table. The table stays in place until all
questions are closed.

**Resolved:**

- Q7 — see [data-model.md](implementation/data-model.md) §"author_id + author_type".
- Q8 — see [chats.md §6](instances/chats.md) and [governance.md §7](primitive/governance.md).
- Q3 — see [graph-model.md §3](primitive/graph-model.md) "What creates an actor edge — stances-not-events".
- Q2 — see [feed-ranking.md §3-§4](primitive/feed-ranking.md) (per-edge composition, parallel tracks, taint rule, sum collapser) and [graph-model.md §6](primitive/graph-model.md) (dim1/dim2 unification, filtering vs. graph math). S's intrinsic derivation deferred — tracked as Q16.
- Q11 — see [feed-ranking.md §3.5–§3.6](primitive/feed-ranking.md) (`(0, 0)` severance edge, cascading severance, redemption) and [feed-ranking.md §5](primitive/feed-ranking.md) (zero-jail banishment of `h(t) = 0`). Self-discovery and return-pathway UX surfaces are tracked as forward sub-questions Q12 and Q13.
- Q12 — see [feed-ranking.md §3.7.1](primitive/feed-ranking.md) (severance discovery via inbound self-query, trust-weighted reading) and [feed-ranking.md §3.7.2](primitive/feed-ranking.md) (auto-detection of bot-bridge nodes via hourglass path patterns, with path-length-aware action guidance). Cause identification is the auto-detect's job, complemented by the community posts in §3.7.3.
- Q13 — see [feed-ranking.md §3.7.4](primitive/feed-ranking.md) (severer-side redemption surface, hourglass check on the redeeming node's outbound) and [feed-ranking.md §3.7.5](primitive/feed-ranking.md) (self-redemption posts via the same `bot-defense` hashtag mechanism, surfaced in the severer's "review severed accounts" view).
- Q14 — see [data-model.md "Node identity strategies"](implementation/data-model.md) (three-strategy framework: content-addressed UUIDv5 for canonical-string nodes like Hashtag, random UUID + UNIQUE handle for User/Collective, random UUID alone for per-creation nodes). Hashtag IDs are now content-addressed so independent creations of the same canonical name converge on one node. Cross-instance federation reconciliation for Types 2 and 3 is deferred as Q15.
- Q6 — see [invitations.md "Default values and customization"](primitive/invitations.md). Defaults are `(+0.5, +0.5)` on both edges; both inviter and invitee choose their own outgoing edge during the invitation flow. The doc walks through the asymmetric-friend example (`(+1, -1)` on the invitee side as a deliberate "love them, not their content" stance that lets a later second edge dominate the feed).
- Q4 — see [feed-ranking.md §7](primitive/feed-ranking.md). Time decay anchors on the **reactor edge's top-layer age** (the last actor edge in the path), applied as a scalar `f(Δt)` multiplier alongside `d(R)` to all four metrics (`h, i, j, k`). Default exponential with **30-day half-life**, frontend-tunable. Intermediate edges don't decay — silence on a relationship edge is not stance revocation. Post-node age has no separate decay — the authorship edge is itself a reactor edge and ages with the post, so old-with-no-engagement decays naturally and old-with-fresh-engagement resurfaces via fresh reactor-edge layers. Worked cold-start example in §7.3 shows the math.
- Q1 — see [graph-model.md §8](primitive/graph-model.md). Layer count, layer timestamps, and the sequence of past edge values are **not ranking inputs**. They are metadata for audit, history, and UI surfaces (e.g., a "this edge has been revised N times" indicator, or a stale-edge prompt). Ranking sees only the top layer of each edge — the user's current expressed stance. Rationale: introducing layer-count amplification would let the system infer intent from interaction frequency, in tension with both **stances-not-events** ([graph-model.md §3](primitive/graph-model.md)) and the user-controlled-ranking principle. Edge cases like "two friends with identical edges but very different real-world contact frequency" are explicitly not auto-resolved by the system; users update stances reactively (similar to pruning a stale subscription list) rather than the system inferring from behavior.
- Q5 — see [feed-ranking.md §8](primitive/feed-ranking.md). The seen-list is a per-viewer set of content UUIDs treated as **another input to the feed-ranking computation**, alongside `R`, `d(R)`, `f(Δt)`, and the §5.2 friend-author-boost. Pre-rank exclusion (perf win — already-seen content never enters the math). New activity on a seen post does **not** resurface it; the new comment/reaction is independently rankable as its own node. Storage location is the viewer's choice — backend-side `user_view_log` table in Postgres is the central frontend's default ([data-model.md](implementation/data-model.md)), but self-hosted clients/miners can keep the same data locally and pass it to the calculator (the math is the same regardless of where the JSON came from). Default frontend rule for "seen": every content item that passes through the viewport during a render. Frontend batches and flushes on natural checkpoints (batch-fill, scroll pause, app close); cache-clear before flush is an accepted small loss-window. Default 1-year compaction bounds storage at ~7 MB per active-user-year; the trade-off (a resurging old post will reappear if its view-log entry has been compacted) is documented and treated as acceptable feed character. No privacy-concealment story — viewing history is no more sensitive than reaction history per the network's transparency posture; "history" becomes a UI feature using the same data.
- Q10 — reframed as a side note rather than an open design question. See [layers.md "Side note on long-term storage"](primitive/layers.md). Typical actor behavior bounds layer accumulation tightly — people update an edge a handful of times over its lifetime, not hundreds, and node properties change even less frequently. The corner cases that *would* accumulate substantial history (e.g., a decades-old company restructuring through CollectiveMember edges) are precisely the ones where preserving history has value. If a real instance ever runs into storage pressure, compaction-friendly approaches that respect the no-silent-deletion principle exist — but it's an implementation-time decision contingent on real data, not a design-time one to settle preemptively.
- Q9 — see [moderation.md](instances/moderation.md) and [network.md](primitive/network.md). Authorization for redaction runs through community-driven Network governance: any User authors a Proposal classifying content as `illegal`; threshold-cross requires at least one moderator's positive vote (the gate), ≥2/3 of cast votes in favor, and a low community quorum; threshold-cross triggers the [layers.md §5](primitive/layers.md) redaction cascade. External pressure (court orders, etc.) doesn't bypass the mechanism — it prompts a moderator to start the same Proposal, which the community completes. Pathological corner cases (all moderators compromised) fall under the federation/forking exit per Q15.

---

## Q16 — Derivation of `S(t)`, the intrinsic node scalar

**Where it shows up:** [feed-ranking.md §2](primitive/feed-ranking.md) (S in the variable table) and [feed-ranking.md §5](primitive/feed-ranking.md) (S as the final fallback after the h-cascade tie-breakers)
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
relative to a viewer. Candidate inputs include the node's own
authorship-edge age, the node's neighborhood density, the node
type itself, or composite measures over the local subgraph. The
choice affects how ties resolve in sparse graphs, on cold-start
viewers, and for users whose default values produce many exact
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

**Where it shows up:** [data-model.md "Node identity strategies"](implementation/data-model.md) (Type 2 and Type 3 federation notes)
**Status:** open (deferred — federation phase)

### Context

The Q14 resolution settled three identity strategies in the data
model, with very different federation properties:

- **Type 1 — canonical-string identity, content-addressed
  UUIDv5** (Hashtag). Federates by construction. Same canonical
  name produces the same UUID across any instance or fork. No
  reconciliation needed.
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

### Constraints (from established principles)

- **No central authority.** Per CLAUDE.md, anyone can fork and
  self-host. Federation cannot depend on a central registry.
- **Append-only.** Per [layers.md](primitive/layers.md),
  reconciliation cannot retroactively rewrite local state. New
  layers / new edges may be appended; old ones stay.
- **Transparency.** Reconciliation choices (alias, claim, merge)
  leave a visible trace on-graph.
- **Severance is local to the severing community.** Per
  [feed-ranking.md §3.6](primitive/feed-ranking.md), the math is
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
[feed-ranking.md §3.6](primitive/feed-ranking.md) (cluster
severance — local to the severing community per principle, but
federation could change this).
