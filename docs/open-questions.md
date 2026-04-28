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
| 1. Adversarial robustness | 1 | **Q11** | Default sort produces correct math, but bots can engineer score positions. Needs exhaustive treatment before the feed is production-ready. |
| 2. Onboarding | 2 | **Q6** | Now that the ranking math is defined (Q2), invitation-edge defaults can be designed against concrete ranking behavior. |
| 3. Decay & layer signal | 3 | **Q4** | Decay composes with `R/h/i/j/k` — needs the metrics defined (now done) before decay can be designed. |
| | 4 | **Q1** | Layer count finds its place once primitives and decay are settled (modifier? separate parameter? folded into `i`?). |
| 4. Build on foundations | 5 | **Q5** | Informed by Q4 (decay may absorb part of "seen"). Q3 already ruled out the implicit-view-edge options. |
| 5. Scale concerns | 6 | **Q10** | Gated by Q1 — compaction has to preserve (or explicitly degrade) the layer-count signal. Only pressing at scale. |
| 6. Policy, externally gated | 7 | **Q9** | Independent of technical work and independent of what blocks technical work. Needs legal + decentralization-roadmap input; don't let it gate anything else. |

As questions resolve, their blocks disappear from below and their
rows disappear from this table. The table stays in place until all
questions are closed.

**Resolved:**

- Q7 — see [data-model.md](implementation/data-model.md) §"author_id + author_type".
- Q8 — see [chats.md §6](instances/chats.md) and [governance.md §7](primitive/governance.md).
- Q3 — see [graph-model.md §3](primitive/graph-model.md) "What creates an actor edge — stances, not events".
- Q2 — see [feed-ranking.md §3-§4](primitive/feed-ranking.md) (per-edge composition, parallel tracks, taint rule, sum collapser) and [graph-model.md §6](primitive/graph-model.md) (dim1/dim2 unification, filtering vs. graph math). S's intrinsic derivation deferred — flagged as a forward sub-question.

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

## Q6 — Initial dimension values on invitation edges

**Where it shows up:** [invitations.md](primitive/invitations.md)
**Status:** open

### Context

When an existing actor invites a new actor, two edges are created —
one in each direction (see [invitations.md](primitive/invitations.md) for
the two-edge pattern). The new actor needs at least one outgoing edge
the moment they join, or their feed has nothing to compute from.

The values on the **new actor's edge toward the inviter** are the
design call. The new actor can update the edge over time like any
other, but the initial values shape the first week of their experience.

### The question

What should the default dimension values be on the new actor's
outgoing edge toward the inviter?

### Options considered

- **High positive** (e.g. sentiment +0.8, closeness +0.7) — you
  presumably like the person who invited you. But this biases the new
  user's feed heavily toward one person's graph neighborhood for their
  first days.
- **Moderate positive** (e.g. +0.3, +0.3) — softer start. The new
  user's feed will be thin until they build more edges.
- **Neutral** (0.0, 0.0) — no bias, but the new user has almost no
  foothold. Feed may be nearly empty.

### Related

None directly.

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

## Q11 — Adversarial robustness of the default sort

**Where it shows up:** [feed-ranking.md §3.5](primitive/feed-ranking.md) (Bot resistance), [feed-ranking.md §5](primitive/feed-ranking.md) (Algorithm)
**Status:** open

### Context

The Q2 resolution lays down honest math: `h(t)` reflects the
graph's actual signal toward a target. The score-based sort (§5)
puts strong-positive close-by signal at the top of the feed and
strong-negative close-by at the bottom. The community defenses
in §3.5 (inbound-edge directionality, non-engagement, `(0, -1)`
defender pattern) give real users powerful tools.

But sophisticated bot clusters can engineer their target's score
position by mixing path types. The math itself is honest — the
question is whether the *sort rule* and *defender patterns*
together leave exploitable surfaces. Several attack vectors have
surfaced and none is fully resolved.

### Attack vectors identified so far

**Attack 1 — `(0, 0)` muting of a `(0, -1)` defender.**
Defender places `(0, -1)` on a path entering the bot cluster. Bots
respond with `(0, 0)` edges *internal* to the cluster.
- Defender's `dim1 = 0` already zeros `s_path` (kill rule, §3.2).
- Bots' `dim2 = 0` zeros `|c_path|` magnitude — overrides
  defender's `|−1|` contribution.
- Total path score collapses to `0 + 0 = 0`. Defender's
  suppression is neutralized. Target lands at score 0, just below
  positives in the default sort.

**Attack 2 — Sign-flip of a `(-1, -1)` defender via `(-1, 0)`.**
A less-coordinated defender uses `(-1, -1)` (negative on both
dims) instead of the more robust `(0, -1)`. Bots can flip this:
- Path: `... → defender(-1, -1) → bot(-1, 0) → target(...)`.
- `s_path`: `... × (-1) × (-1) × ... = +1` (sign flip via
  signed-graph balance — two negatives multiply to positive).
- `c_path`: defender's `|−1| = 1` magnitude × bot's `|0| = 0`
  magnitude = 0. `c_path = 0`.
- Total: `+1 + 0 = +1`. The defender's tainted path is **flipped
  to positive**.
- Bots can then amplify via infinite internal nodes/edges,
  creating a flood of positive paths to the target.

The `(-1, -1)` shape is fundamentally exploitable: its `dim1 = -1`
is recoverable via signed-graph balance (bots add another
negative dim1), and its `dim2 = -1` magnitude can be zeroed by
`dim2 = 0` downstream. Only `(0, -1)` is robust — zero in `dim1`
is permanent (kill rule), and the negative `dim2` taint sign is
one-way.

**Attack 3 — Arbitrary score control via path mixing.**
With infinite internal nodes/edges, bots can compose paths in any
proportion they choose:
- `(0, 0)`-muted paths to **neutralize** defender contributions
  (zero out paths through them).
- `(-1, 0)` sign-flips on `(-1, -1)` defender paths to **convert**
  them into positives (Attack 2).
- Bot-positive paths (amplified through cluster) to **inject**
  arbitrary positive contribution.
- Selectively unguarded paths to introduce a controlled amount
  of negative or positive contribution.

The net effect: a target's aggregate score is the sum of these
contributions, and **bots can dial the aggregate to any value**
— very positive, slightly positive, near zero, slightly negative,
or very negative — by tuning the mix of paths their cluster
exposes. They are not constrained to "small negative" or any
specific position; they can choose their feed slot up to the
limits of what the surrounding real-user signal allows.

The fundamental constraint is that bots cannot manufacture
*outgoing edges from real users into the cluster* (per the
inbound-edges-don't-affect-feed rule). Once a real user has an
outgoing edge into the cluster, however, bots can compound that
edge into virtually any aggregate they want.

### The question

How should the default sort and/or the math handle adversarial
score engineering so that bot-controlled content lands at the
bottom of the feed *regardless* of what mix of path types the
bot cluster uses, given that bots can create unbounded internal
edges/nodes?

Constraints (non-negotiable):
- Bot networks can always create infinitely more edges and nodes
  than real users can.
- Defense must come from community signal + math properties, not
  central gatekeeping (no allow/blocklists, no AI classification).
- The default sort must keep "closeness is the most important
  factor" (positives ranked by closeness × strength).
- Negatives stay visible (transparency principle,
  [layers.md](primitive/layers.md)). They are not banished.
- Real users actively marking bot clusters with `(0, -1)` should
  be **decisive** against those clusters — community moderation
  must work even against an infinite-edge adversary.

### Options considered (none worked out — surfaced for context only)

These are notes from prior discussion. Each has failure modes
against one or more of the attacks above. The full design needs
to be built up from scratch, considering all attack vectors at
once rather than mitigating one at a time.

- **Zero-class at bottom.** Sort `|score| < ε` to the bottom of
  feed. Defends Attack 1 but trivially bypassed by Attack 3
  (bots tune to small non-zero values).
- **Threshold-based bot flagging.** Real users mark suspected
  bots; flagged nodes are sorted distinctly. Risks centralization
  if the flagging mechanism isn't append-only and community-driven.
- **R-distribution heuristics.** Use the distribution of paths
  across `R` levels as a signal — bot clusters tend to have
  many far-R internal paths and few close-R real-user paths.
  Speculative; needs simulation.
- **Trust-propagation analogues.** Treat `(0, -1)` edges as
  trust suppression that propagates further than the direct
  path. Tension with "signal flows along actual paths."
- **Constraints at the math level.** Possibly limit how `dim1`
  and `dim2` compose to deny bots the sign-flip / muting tools
  in the first place. Risks breaking other parts of the math.

### Where this needs to go

- A sort rule (and possibly math refinements) that defend all
  three attack vectors *together*.
- Emphasis that `(0, -1)` is the *only* robust defender shape
  (already noted in [feed-ranking.md §3.5](primitive/feed-ranking.md);
  should be educational material for users).
- Acknowledgment that some attack vectors may be fundamental
  limits of multi-path graph ranking against infinite-budget
  adversaries — the design is choosing where to draw the
  practical line, not eliminating the threat entirely.

### Related

Q2 (the underlying ranking math). Bot-attack vectors will likely
re-emerge as Q4 (time decay) and Q1 (layer count) are designed —
adversarial robustness should be considered when adding any new
ranking signal.
