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
| 1. Independent quick wins | 1 | **Q8** | Self-contained to chats; independent of ranking math. |
| 2. Principle-driven | 2 | **Q3** | Principle-driven (transparency). Gates Q5 — the likely answer ("only explicit actions create edges") rules out Q5's implicit-view-edge options. |
| 3. Ranking foundation | 3 | **Q2** | The keystone. Every downstream ranking question needs the primitive operation defined (what a float edge value *means* to the ranker). |
| | 4 | **Q6** | Depends on Q2: "good" default values on invitation edges only mean something once the ranking math gives them meaning. |
| | 5 | **Q4** | Now that ranking primitives exist, decay can compose with `R/h/i/j/k`. |
| | 6 | **Q1** | Now that primitives *and* decay are settled, layer count finds its place (modifier? separate parameter? folded into `i`?). |
| 4. Build on foundations | 7 | **Q5** | Gated by Q3 (what signals are allowed) and informed by Q4 (decay may absorb part of "seen"). |
| 5. Scale concerns | 8 | **Q10** | Gated by Q1 — compaction has to preserve (or explicitly degrade) the layer-count signal. Only pressing at scale. |
| 6. Policy, externally gated | 9 | **Q9** | Independent of technical work and independent of what blocks technical work. Needs legal + decentralization-roadmap input; don't let it gate anything else. |

As questions resolve, their blocks disappear from below and their
rows disappear from this table. The table stays in place until all
questions are closed.

**Resolved:** Q7 — see [data-model.md](data-model.md) §"author_id + author_type".

---

## Q1 — Layer count as a ranking signal

**Where it shows up:** [graph-model.md §8](graph-model.md) (append-only history)
**Status:** open

### Context

Every edge is a stack of append-only layers. Each interaction adds a
new layer; old layers are never removed (see
[layers.md](layers.md)). The number of layers on an edge is
therefore itself a signal: an edge with 50 layers represents a deep,
frequently-revisited relationship; an edge with 1 layer is a passing
interaction.

The [feed ranking algorithm](feed-ranking.md) currently has no
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

Q2 (cross-type dimension comparability) — if layer count modifies a
dimension, it has to modify it consistently across edge types.

---

## Q2 — Cross-type dimension comparability and float-to-sign mapping

**Where it shows up:** [graph-model.md §9](graph-model.md) (relationship to feed ranking)
**Status:** open

### Context

Two tangled sub-questions that are best resolved together:

**(a) Cross-type combination.** The ranking algorithm traverses paths
that cross edge types with different dimension meanings — for example
`User -> User -> Comment -> Post`. The `User -> User` dimensions are
*sentiment* and *closeness*; the `Comment -> Post` structural edge is
*(0, 0)* by default; the `User -> Post` dimensions are *sentiment* and
*relevance*. The math is uniform (all `f64` in `[-1, +1]`) but the
semantics differ.

**(b) Float-to-sign mapping.** [feed-ranking.md](feed-ranking.md)
is framed over a **signed** graph (each edge is `+` or `-`). CoGra's
actual edges carry **continuous** values in `[-1, +1]`. How continuous
values map into the ranker's signed math is not specified.

### The question

**(a)** When the ranker walks across edge types, how are dimensions
with different meanings combined into a single scalar per hop?

**(b)** How does a `[-1, +1]` tensor edge feed into the ranker's
per-hop sign/weight?

### Options considered

For (b), plausible shapes:

- **Sign + weight:** `sign = sign(dim1)`, `weight = |dim1| * |dim2|`
  (or similar). Keeps ranker math unchanged; dim2 becomes a magnitude.
- **Product:** single scalar `dim1 * dim2`, pass the sign and
  magnitude of the product to the ranker. Collapses information early.
- **Per-dimension contribution:** compute `h/i/j/k` twice, once per
  dimension, then combine. Preserves information at ranking time but
  doubles the compute.
- **Primary dimension only:** ranker uses `dim1` (sentiment in most
  edge types); `dim2` is a secondary signal used elsewhere (filtering,
  suggestions, decay weighting).

None ruled out. (a) depends on what (b) settles.

### Related

Q1 (layer count), Q4 (time decay — decay needs to know what it's
decaying).

---

## Q3 — Minimum interaction required to create an edge

**Where it shows up:** [graph-model.md §1](graph-model.md) (transparency principle)
**Status:** open

### Context

The graph is built from interactions. But "interaction" covers a wide
spectrum — from an explicit like on a post to merely scrolling past it
in a feed. Implicit signals (dwell time, scroll-past, hover) are how
existing social apps build their ranking inputs, and they feel like
surveillance. CoGra's transparency principle
([CLAUDE.md](../CLAUDE.md) principle #6) makes implicit tracking
uncomfortable.

### The question

Where is the line between *implicit signal* and *explicit action*?
Does viewing a post for 3 seconds create an edge? Does scrolling past
it without viewing? Does opening a chat create an edge toward it?

### Options considered

None explicitly mapped. The principle constraint is that implicit
signals are uncomfortable with transparency. A defensible default
might be: **only explicit actions** (like, comment, follow, mute,
open-and-read-a-message) create or update edges — but that leaves
signals on the table that legitimately help ranking (e.g. a user
repeatedly opens a chat without posting).

### Related

Q5 (already-seen tracking) — some options for "seen" tracking create
implicit view-edges, so this question gates them.

---

## Q4 — Time and recency: decay shape

**Where it shows up:** [feed-ranking.md §7](feed-ranking.md)
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

**Where it shows up:** [feed-ranking.md §8](feed-ranking.md)
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

- **A. View edges on the graph** — every viewed node gets a `(0, 0)`
  edge.
  - Pro: graph-native; "I've seen this" is just another edge.
  - Con: explodes the edge count. Instead of sorting through 3 posts a
    friend liked, you sort through 10,000 posts they've viewed.
    Computation cost becomes untenable.

- **B. Separate "seen" store outside the graph** (e.g. Redis set,
  bitset, bloom filter).
  - Pro: doesn't pollute the graph; compact data structures possible.
  - Con: breaks the "everything is in the graph" property. Adds a
    third data store.

- **C. Client-side "seen" tracking.**
  - Pro: aligns with the decentralized/compute-close-to-viewer vision
    (see [feed-ranking.md §9](feed-ranking.md)). The client already
    has the subgraph and knows what it rendered.
  - Con: doesn't sync across devices without additional infra.

- **D. View edges with aggressive compaction** — only recent views
  live as individual layers; older ones get compacted into a summary.
  - Pro: graph-native.
  - Con: compaction logic adds complexity; "recent" is another
    decision to make.

### Related

Q3 (minimum interaction — view-edge options (A) and (D) conflict with
"only explicit actions create edges"), Q4 (decay).

---

## Q6 — Initial dimension values on invitation edges

**Where it shows up:** [invitations.md](invitations.md)
**Status:** open

### Context

When an existing actor invites a new actor, two edges are created —
one in each direction (see [invitations.md](invitations.md) for
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

## Q8 — Admin/mod powers vs community voting in chats

**Where it shows up:** [chats.md §6](chats.md) (Roles still apply)
**Status:** open (implementation-level design)

### Context

Open public chats have no admin gating joining — anyone can become a
member. But chats can still have admins and mods (roles carried as
properties on ChatMember junction nodes — see
[nodes.md §3](nodes.md)). Moderation in CoGra works by **the chat
moving away from a message, not the other way around** — members vote
to disavow a message, and a new layer is added to the relevant
structural edge signaling that the chat no longer associates itself
with it.

An admin's disavowal may carry more weight than a regular member's,
but the exact composition between role weight and community voting is
unspecified.

### The question

How do admin/mod powers compose with community voting?

- Does an admin's vote count as N regular votes?
- Can an admin single-handedly disavow without community input (unilateral veto)?
- Is there a threshold of community votes that overrides an admin?
- How does this scale with multi-sig policies (several admins required)?

### Options considered

None explicitly laid out. The primitives exist (weighted votes via
role properties, multi-sig via N-of-M actor edges); the policy on top
of them is the design call.

### Related

Q9 (redaction authority — there's a similar "who decides" shape for
content takedown).

---

## Q9 — Who authorizes a redaction, and through what process

**Where it shows up:** [layers.md §5](layers.md) (Out of scope)
**Status:** open (policy)

### Context

The graph is append-only, but [layers.md §5](layers.md) carves out a
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

Q8 (admin/mod × community voting has a similar "who decides" shape,
though with lower stakes), Q10 (retention).

---

## Q10 — Layer retention and pruning for storage cost

**Where it shows up:** [layers.md §5](layers.md) (Out of scope)
**Status:** open (implementation optimization)

### Context

Append-only means every interaction adds a layer, forever. At some
point, storing infinite history has a cost — both at the graph layer
(edge layer stacks) and at the Postgres layer (version rows on
display content).

The principle in [layers.md](layers.md) is non-negotiable: no silent
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
