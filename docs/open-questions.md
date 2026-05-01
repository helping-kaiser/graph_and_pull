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
| 1. Decay & layer signal | 1 | **Q4** | Decay composes with `R/h/i/j/k` — needs the metrics defined (now done) before decay can be designed. |
| | 2 | **Q1** | Layer count finds its place once primitives and decay are settled (modifier? separate parameter? folded into `i`?). |
| 2. Build on foundations | 3 | **Q5** | Informed by Q4 (decay may absorb part of "seen"). Q3 already ruled out the implicit-view-edge options. |
| 3. Scale concerns | 4 | **Q10** | Gated by Q1 — compaction has to preserve (or explicitly degrade) the layer-count signal. Only pressing at scale. |
| 4. Policy, externally gated | 5 | **Q9** | Independent of technical work and independent of what blocks technical work. Needs legal + decentralization-roadmap input; don't let it gate anything else. |
| 5. Federation, post-spike | 6 | **Q15** | Identity reconciliation across separately-running instances for handle-based and per-creation node types. Type 1 nodes (hashtags) federate for free per Q14; Types 2 and 3 need a protocol. Deferred until federation becomes concrete. |

As questions resolve, their blocks disappear from below and their
rows disappear from this table. The table stays in place until all
questions are closed.

**Resolved:**

- Q7 — see [data-model.md](implementation/data-model.md) §"author_id + author_type".
- Q8 — see [chats.md §6](instances/chats.md) and [governance.md §7](primitive/governance.md).
- Q3 — see [graph-model.md §3](primitive/graph-model.md) "What creates an actor edge — stances, not events".
- Q2 — see [feed-ranking.md §3-§4](primitive/feed-ranking.md) (per-edge composition, parallel tracks, taint rule, sum collapser) and [graph-model.md §6](primitive/graph-model.md) (dim1/dim2 unification, filtering vs. graph math). S's intrinsic derivation deferred — flagged as a forward sub-question.
- Q11 — see [feed-ranking.md §3.5–§3.6](primitive/feed-ranking.md) (`(0, 0)` severance edge, cascading severance, redemption) and [feed-ranking.md §5](primitive/feed-ranking.md) (zero-jail banishment of `h(t) = 0`). Self-discovery and return-pathway UX surfaces are tracked as forward sub-questions Q12 and Q13.
- Q12 — see [feed-ranking.md §3.7.1](primitive/feed-ranking.md) (severance discovery via inbound self-query, trust-weighted reading) and [feed-ranking.md §3.7.2](primitive/feed-ranking.md) (auto-detection of bot-bridge nodes via hourglass path patterns, with path-length-aware action guidance). Cause identification is the auto-detect's job, complemented by the community posts in §3.7.3.
- Q13 — see [feed-ranking.md §3.7.4](primitive/feed-ranking.md) (severer-side redemption surface, hourglass check on the redeeming node's outbound) and [feed-ranking.md §3.7.5](primitive/feed-ranking.md) (self-redemption posts via the same `bot-defense` hashtag mechanism, surfaced in the severer's "review severed accounts" view).
- Q14 — see [data-model.md "Node identity strategies"](implementation/data-model.md) (three-strategy framework: content-addressed UUIDv5 for canonical-string nodes like Hashtag, random UUID + UNIQUE handle for User/Collective, random UUID alone for per-creation nodes). Hashtag IDs are now content-addressed so independent creations of the same canonical name converge on one node. Cross-instance federation reconciliation for Types 2 and 3 is deferred as Q15.
- Q6 — see [invitations.md "Default values and customization"](primitive/invitations.md). Defaults are `(+0.5, +0.5)` on both edges; both inviter and invitee choose their own outgoing edge during the invitation flow. The doc walks through the asymmetric-friend example (`(+1, -1)` on the invitee side as a deliberate "love them, not their content" stance that lets a later second edge dominate the feed).

---

## Q1 — Layer count as a ranking signal

**Where it shows up:** [graph-model.md §8](primitive/graph-model.md) (append-only history)
**Status:** open

### Context

Every edge is a stack of append-only layers. Each interaction adds a
new layer; old layers are never removed (see
[layers.md](primitive/layers.md)). The number of layers on an edge is
therefore itself a signal: an edge with 50 layers represents a deep,
frequently-revisited relationship; an edge with 1 layer is a passing
interaction.

The [feed ranking algorithm](primitive/feed-ranking.md) currently has no
input for layer count. Its metrics `h`, `i`, `j`, `k` operate on edge
values and presence, not on how many times an edge has been touched.

### The question

How should layer count factor into ranking? Is it:

- A modifier on the top-layer dimension values (e.g. a multiplier that
  amplifies a strong, long-standing relationship)?
- A separate ranking parameter alongside `h/i/j/k`?
- Folded into an existing parameter (e.g. part of `i` "importance")?
- Used only for time decay and recency weighting, not structural ranking?

### Options considered

None yet.

### Related

The dim1/dim2 grammar is now uniform project-wide
([graph-model.md §6](primitive/graph-model.md)), so a layer-count
modifier on a dimension would compose consistently across edge types.

---

## Q4 — Time and recency: decay shape

**Where it shows up:** [feed-ranking.md §7](primitive/feed-ranking.md)
**Status:** open

### Context

Time decay must exist in some form but is not designed. Known
constraints:

- Old content can become newly relevant (a friend comments on a post I
  liked years ago — I should see the comment, and the post becomes
  slightly more relevant again).
- New content can be irrelevant (a brand-new post from someone 5 hops
  away that no one I know has interacted with).
- **Recency is not importance.** Time is a factor but not a dominant
  one.

### The question

What shape should the decay function take, and how does it compose
with the ranking parameters `R`, `h`, `i`, `j`, `k`?

### Options considered

Shapes plausible but not evaluated:

- **Exponential decay** on edge age — standard in most feed systems;
  simple but makes old content vanish quickly.
- **Linear decay with a floor** — old content never goes below some
  minimum weight.
- **Step function** — "active" vs "archived" buckets.
- **No decay on edge weight; decay only on the *target node*'s
  recency score** — separates "my relationship" from "this post is
  old."

Composition with `R/h/i/j/k` is open: decay could multiply `h`, act
as a separate dimension in the sort/order chain, or modify `R` (old
content pushed into a higher bucket).

### Related

Q1 (layer count), Q5 (already-seen).

---

## Q5 — The "already seen" problem

**Where it shows up:** [feed-ranking.md §8](primitive/feed-ranking.md)
**Status:** open

### Context

Users should not be re-shown content they've already seen, **unless**
something meaningful happened (e.g. a friend commented on it since
they last saw it). Any solution must:

1. Not flood the graph with low-signal edges.
2. Not be a black box.
3. Allow users to revisit content manually.
4. Surface content again when something meaningful changes (new
   interactions from people the user cares about).

### The question

How should "already seen" tracking work?

### Options considered

- **~~A. View edges on the graph~~** — ruled out by Q3
  (graph-model.md §3 "stances, not events"): a viewed-but-unreacted
  node does not create an actor edge.
- **B. Separate "seen" store outside the graph** (e.g. Redis set,
  bitset, bloom filter).
  - Pro: doesn't pollute the graph; compact data structures possible.
  - Con: breaks the "everything is in the graph" property. Adds a
    third data store.
- **C. Client-side "seen" tracking.**
  - Pro: aligns with the decentralized/compute-close-to-viewer vision
    (see [feed-ranking.md §9](primitive/feed-ranking.md)). The client already
    has the subgraph and knows what it rendered.
  - Con: doesn't sync across devices without additional infra.
- **~~D. View edges with aggressive compaction~~** — ruled out by Q3
  for the same reason as A.

### Related

Q4 (decay).

---

## Q9 — Who authorizes a redaction, and through what process

**Where it shows up:** [layers.md §5](primitive/layers.md) (Out of scope)
**Status:** open (policy)

### Context

The graph is append-only, but [layers.md §5](primitive/layers.md) carves out a
narrow exception: the contents of a specific node-property layer (or a
Postgres display-content row) can be **redacted in place** when the
content itself is illegal. The layer stays; its value is replaced
with a `[redacted — ...]` marker. No silent deletion, ever.

Layers.md defines the **mechanism**. The **policy** around who
authorizes a redaction is explicitly out of scope there and lives
here.

### The question

Who can trigger a redaction, and what process must happen first?

- What threshold of evidence is required (e.g. court order, platform
  moderator judgment, community vote)?
- Who actually applies the redaction — a central operator, a
  multi-sig of trusted validators, anyone running the software?
- What appeal rights does the affected actor have?
- How does this interact with the decentralization goal — if anyone
  can run an instance, whose redactions propagate to whose instance?

### Options considered

None concrete. This is a policy design that will be influenced by
legal jurisdiction, the decentralization roadmap, and the economic
model.

### Related

Q10 (retention). Chat moderation (resolved Q8 — see
[chats.md §6](instances/chats.md)) is a similar "who decides" shape with
much lower stakes.

---

## Q10 — Layer retention and pruning for storage cost

**Where it shows up:** [layers.md §5](primitive/layers.md) (Out of scope)
**Status:** open (implementation optimization)

### Context

Append-only means every interaction adds a layer, forever. At some
point, storing infinite history has a cost — both at the graph layer
(edge layer stacks) and at the Postgres layer (version rows on
display content).

The principle in [layers.md](primitive/layers.md) is non-negotiable: no silent
deletion. But there's a spectrum between "keep every layer verbatim
forever" and "compact old layers into summaries" that preserves the
principle.

### The question

Should we compact old layers? If yes, how — and what's preserved vs.
summarized?

- Is there a retention horizon after which old layers are compacted
  into a rollup (e.g. "100 layers between T=0 and T=5"
  → one summary layer)?
- Does compaction apply to edges, node properties, Postgres content,
  or all three? Different data has different decay curves.
- Does "layer count" (see Q1) stay exact or become an estimate after
  compaction?

### Options considered

None worked out. Pure optimization concern — the principle is
decided; this is how cheaply it can be implemented without breaking
the principle.

### Related

Q1 (layer count as signal — compaction changes what layer count
means), Q9 (redaction authority — retention decisions affect what
can still be redacted).

---

## Q15 — Cross-instance federation: identity reconciliation for handle-based and per-creation nodes

**Where it shows up:** [data-model.md "Node identity strategies"](implementation/data-model.md) (Type 2 and Type 3 federation notes)
**Status:** open (deferred — post-spike)

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
