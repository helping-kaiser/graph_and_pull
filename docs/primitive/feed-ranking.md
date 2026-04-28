# Ranking via Weighted Graph Connections

A general framework for ranking and ordering **target nodes** in a graph
where edges carry **2-dimensional tensors**, relative to a chosen
**root node**.

The social-feed setup (User → other Users → Posts) that originally
motivated this is shown as examples at the end. The rule itself is
layer-agnostic — it applies to any graph where edges encode
`(valence, connection-weight)` per edge and a root wants to rank some
set of target nodes reachable through intermediate connections.

---

## 1. Setup

A graph with:
- a **root node** `U` — the perspective we rank from,
- one or more layers of **intermediate nodes**,
- a set of **target nodes** — what we're ranking,
- two edge categories (per [graph-model.md §3](graph-model.md)):
  - **Actor edges**: created by actors. Carry a 2D tensor
    `(dim1, dim2)`, each in `[-1.0, +1.0]`.
    - `dim1` is **signed valence** (sentiment / approval / affirmation).
    - `dim2` is **signed connection-weight** (closeness / relevance /
      importance).
  - **Structural edges**: system-created topology. Do not contribute
    factors to the ranking math; only count toward path length and
    (where state-bearing) gate traversability — see §3.1.

The algorithm's job: given this graph, produce an ordered list of the
target nodes as seen from `U`.

---

## 2. Parameters

| Symbol | Name | Meaning |
|--------|------|---------|
| `R` | Real number of graph hops | Path length (number of edges) from `U` to the target. Targets with the same `R` form a comparison group. Both actor and structural edges count toward `R`. |
| `S` | Scalar value of a node | An intrinsic scalar assigned to each node. Used in the **sort** phase to pre-order nodes within an `R` group. (S's exact derivation is left as a follow-up — see [open-questions.md](../open-questions.md).) |

---

## 3. Per-edge composition along a path

Per-target metrics (§4) are computed by composing edge tensors along
each path from `U` to a reactor (a node with an actor edge to the
target). The composition uses **parallel tracks**: `dim1` and `dim2`
flow independently through the path product and only collapse to a
scalar at sort time.

### 3.1 Which edges contribute factors

Only **actor edges** contribute factors to the path products.
Structural edges count toward `R` (path length) but do not contribute
factors — they are pure topology.

```
s_path uses only dim1 of actor edges in the path
|c_path| uses only |dim2| of actor edges in the path
R counts every edge in the path (actor + structural)
```

Why structural edges count toward `R` but not toward the products:
a path `U → friend (actor) → Comment (actor reaction) → Post`
where `Comment → Post` is a structural containment edge has `R = 3`.
Under the `d(R)` decay applied at sort time (§4–§5), the friend's
directly-reacted-to comment sits at `R = 2` (more proximate to U),
and the post it's attached to is one structural hop further away
at `R = 3` (slightly less proximate). This matches feed intuition:
a friend's strong comment is more directly relevant than the post
it sits on, even by a small margin.

Deep structural chains (e.g., replies of replies on a post)
accumulate `R` naturally and decay via `d(R)` without needing an
explicit depth cap. The dataminer's R-fetch limit (typically `R ≤
5` or `6` in practice) bounds traversal at the system level.

State-bearing structural edges (junction approval pairs, see
[graph-model.md §5](graph-model.md)) act as **gates on traversability**:
a path is traversable through such an edge only if its top-layer
`dim1` is positive (the relationship is currently affirmed). Their
values do not enter the ranking math; they only decide whether the
path exists at all.

### 3.2 Zero handling — kill rule

A factor of `0` in either dim of any actor edge along the path
zeros that dimension's path product. Zeros are **not** skipped
or treated as multiplicative identity — they are real factors
that, through ordinary multiplication, collapse the chain.

```
if dim1(eᵢ) = 0 for any actor edge eᵢ in path  →  s_path = 0
if dim2(eᵢ) = 0 for any actor edge eᵢ in path  →  c_path = 0
```

The two tracks are independent: a zero in one dim does not affect
the other dim's chain. An edge `(0, +0.7)` zeros `s_path` while
the closeness chain continues via `c_path`; `(+0.7, 0)` zeros
`c_path` while sentiment continues via `s_path`.

Defensible in feed terms: if I have no opinion on a hop, signal
of that type does not flow through me on this path. The hop still
counts as a real edge in the topology; it just contributes nothing
on the dim where I expressed nothing. Compared to a "skip zero"
rule (treating `0` as multiplicative identity `1`), the kill rule
prevents the artifact where a path with a single weak hop and one
zero hop scores stronger than a path with two real weak hops.

### 3.3 dim1 chain — signed multiplication

For a path with actor edges `e_1, e_2, ..., e_R'` (where `R'` is the
number of actor edges in the path; structural edges contribute no
factors per §3.1):

```
s_path = ∏ dim1(e_k)   over all actor edges in the path
       = 0             if any dim1(e_k) is zero (kill rule, §3.2)
```

Signed multiplication preserves **signed-graph balance**: the
"enemy of my enemy is my friend" pattern. Sentiment has trust
transitivity — a real social property, well-studied in signed
graph theory. A path with an even number of negative `dim1`
factors flips back to positive; an odd number stays negative. The
math captures this structural property at every path length.

### 3.4 dim2 chain — taint sign × magnitude product

`dim2` does not have a transitivity rule. "I avoid A; A avoids B"
tells us nothing about my relationship to B — closeness doesn't
compose the way sentiment does. Signed multiplication of `dim2`
along a path would produce sign flips that don't correspond to any
real social pattern (two avoidances would compose to a positive
"connection," which is meaningless).

Instead, dim2 composes via a **taint rule**:

```
|c_path|     = ∏ |dim2(e_k)|   over all actor edges in the path
sign(c_path) = -1   if ANY dim2(e_k) in the path is negative
             = +1   otherwise
c_path       = sign(c_path) × |c_path|
```

If any `dim2(e_k) = 0`, then `|c_path| = 0` per the kill rule
(§3.2) and the sign becomes irrelevant — `c_path = 0`.

Two important properties:

- **Magnitude decays naturally with path length.** The product of
  `|dim2| ≤ 1` factors shrinks with each hop, matching the decay
  behavior of `s_path`. The two tracks scale together — neither
  dominates the other purely by path length.
- **Any avoidance taints the path.** A single negative `dim2`
  anywhere in the path flips the closeness signal to negative,
  regardless of magnitude. Avoidance is non-transitive but
  *carrying*: any cut-off connection along a route reduces what
  flows through it.

A weakest-link rule (`min(dim2)`) was considered and rejected: it
keeps `c_path` magnitude pinned to the most-negative hop, which
doesn't decay with path length and would dominate `s_path` for
deeper paths.

A signed-multiplication rule (matching dim1) was also considered
and rejected: it produces "two-avoidances → positive connection"
artifacts that don't reflect social reality.

---

## 4. Per-target metrics

A target `t` is generally reachable from `U` via **multiple paths**
of varying lengths. The personalized metrics (`h`, `i`) aggregate
signal across all those paths, with each path's contribution
weighted by a distance decay `d(R_π)`. The absolute metrics (`j`,
`k`) are global properties of the target — they describe its
reception across the graph and are independent of U's position, so
no `d(R)` weighting applies.

### 4.1 Path contribution and distance decay

For a path `π` from `U` to `t` of length `R_π` (per §3.1), the
path produces a **2-tuple**:

```
path_tuple(π) = (s_path(π), c_path(π))
```

computed via the rules in §3.3 and §3.4.

Each path's contribution is scaled by a decay factor based on its
length:

```
d(R) = 0.1^(R-1)        (default)
```

So `d(1) = 1`, `d(2) = 0.1`, `d(3) = 0.01`, `d(4) = 0.001`, ...

Steep decay reflects "closeness is the most important factor in the
graph." Direct connections (R=1) carry full weight; each additional
hop reduces the path's contribution by 10×. Bots and viral-distant
content cannot dominate a user's feed by sheer multi-path count
alone — at any reasonable graph density, distant paths contribute
proportionally to how far they are.

The decay function is a frontend-tunable parameter. A user who wants
a broader-network feed can soften the decay (e.g., `0.5^(R-1)`); one
who wants only direct-friend signal can steepen it (e.g.,
`0.01^(R-1)`). The default is calibrated so that a single strong
R=2 path roughly matches ~15 strong R=3 paths' aggregate
contribution — balancing direct signal with friend-of-friend buzz.

### 4.2 The four metrics

The four metrics form a symmetric grid: **opinion** vs. **reach**,
each in **personal** and **absolute** flavors. Personal metrics
depend on U's position in the graph and use `d(R)` decay;
absolute metrics are global properties of the target, unweighted
by U's distance.

|         | **Personal** (uses `d(R)`)  | **Absolute** (no `d(R)`)    |
|---------|-----------------------------|-----------------------------|
| Opinion | `h` — personal opinion       | `j` — absolute opinion       |
| Reach   | `i` — personal reach         | `k` — absolute reach         |

Each metric is a **2-tuple** (one component per dim track):

| Symbol | Name | Sentiment component (`*_s`) | Closeness component (`*_c`) |
|---|---|---|---|
| `h` | personal opinion | `H_s = ∑_π d(R_π) · s_path(π)` over all paths to `t` | `H_c = ∑_π d(R_π) · c_path(π)` over all paths to `t` |
| `i` | personal reach | `I_s = ∑_π d(R_π) · s_path_R−1(π)` over first R−1 edges of each path | `I_c = ∑_π d(R_π) · c_path_R−1(π)` over first R−1 edges |
| `j` | absolute opinion | `J_s = ∑_B dim1(B → t)` over reactors `B` (signed) | `J_c = ∑_B dim2(B → t)` over reactors (signed) |
| `k` | absolute reach | `K_s = ∑_B \|dim1(B → t)\|` over reactors | `K_c = ∑_B \|dim2(B → t)\|` over reactors |

Reading:
- `h` — personal opinion: trust- and connection-weighted opinion
  toward the target, summed across all paths from U with closer
  paths weighted more.
- `i` — personal reach: how strongly U reaches the reactors,
  *regardless* of what they thought of the target.
- `j` — absolute opinion: target's net valence in the graph at
  large — what reactors collectively think of `t`. Same value
  for every viewer.
- `k` — absolute reach: target's total interaction reach — how
  much reaction volume `t` has accumulated, signs absorbed. Same
  for every viewer.

Each metric uses **both `dim1` and `dim2`** through the parallel
tracks. No metric drops a dimension; no dimension drops a metric.

A target with one R=2 path and 15 R=3 paths to the same content has
**meaningfully different** `h` and `i` from one with only the R=2
path — the multi-path sum captures the breadth of engagement across
U's network, weighted by how directly each path reaches U. `j` and
`k` are unaffected by this difference (they describe the target's
graph-wide reception, not U's reach).

### 4.3 Tuple collapse to scalar

Each metric's 2-tuple is collapsed to a scalar at sort time:

```
h(t) = H_s(t) + H_c(t)
i(t) = I_s(t) + I_c(t)
j(t) = J_s(t) + J_c(t)
k(t) = K_s(t) + K_c(t)
```

Default collapser is **sum** (equal weight to both tracks). A
frontend may override with a weighted combination:

```
score(metric) = α × M_s + β × M_c
```

— for example, `α = 2, β = 1` to favor sentiment-weighted ordering,
or `α = 1, β = 2` to favor closeness-weighted ordering.

Sum is the default because it correctly handles the case where both
tracks are negative: a path the graph is pushing down on both axes
should stay pushed down. A **product** collapser was rejected for
this reason — it would flip `(−)(−) → +` and surface paths the math
is trying to suppress.

---

## 5. Algorithm

The ranking is a single sort by **personal opinion `h`** descending,
with cumulative tie-breakers and `S` as the final fallback.

```
sort by:   h(t)
           if equal:  h(t) + i(t)
           if equal:  h(t) + i(t) + j(t)
           if equal:  h(t) + i(t) + j(t) + k(t)
           if equal:  S(t)
```

`d(R)` decay (§4.1) is already baked into the personal metrics
(`h`, `i`), so a single sort by `h` naturally puts close-strong
signals at the top and distant signals at the bottom — no
separate R-bucketing phase is needed.

Strict R-bucketing was considered and rejected: it forced any
direct connection (however weak) above any indirect connection
(however strong). The score-based sort is more nuanced — it lets
a target with many strong R=3 paths outrank a target with one
weak R=2 path, while preserving "closeness is most important"
through `d(R)`'s steep decay.

Targets with `h(t) > 0` appear at the top of the feed; `h(t) < 0`
at the bottom. Negatives are **visible**, not banished — a friend
strongly disliking something is meaningful information for the
viewer to be aware of, and the graph's transparency principle
favors showing them over hiding. They sort below positives because
the score itself is negative; that's it.

The cascade activates only on **strict equality** at each level.
With float math, exact ties on `h` are rare; the cascade kicks in
mostly for sparse graphs (where many targets have `h ≈ 0` exactly)
and for users who default to `+1/0/-1` integer values (where ties
are common). `S` (the intrinsic node scalar) is the deepest
fallback — see [open-questions.md](../open-questions.md) for its
derivation.

> **Adversarial robustness of the default sort is an open question.**
> The math is honest about path-product values, but bots can
> engineer their content's score by mixing path types (positive,
> zero, negative) to land at chosen positions in the feed. See
> [open-questions.md Q11](../open-questions.md) for the unresolved
> question and current thinking.

### 5.1 Filtering vs ranking

Hard "never show me content from user X" exclusion is a
**frontend concern**, applied as a post-ranking filter. The graph
math uses `dim2 < 0` as a continuous taint signal (§3.4) but does
not snap such paths to zero — paths are reduced smoothly via the
taint rule, proportional to the rest of the path's strength. This
separation lets the math stay smooth and continuous while still
letting users enforce hard exclusions in their UI.

For where ranking and filtering compute (client-side, miner nodes,
etc.), see §8.

---

## 6. Examples

These examples use small floats (and `±1` unit values for the
exhaustive R=2 table) to illustrate the math. All paths use only
actor edges; structural edges in real paths would be skipped in the
products per §3.1.

### 6.1 R=2, all 16 sign combinations

Path: `U → A → post`. Each edge `(dim1, dim2)` with values in
`{+1, -1}`. Score = `s_path + c_path` (default sum collapser).

| # | U→A | A→post | s_path | c_path | score | reading |
|---|---|---|:---:|:---:|:---:|---|
| 1 | (+,+) | (+,+) | +1 | +1 | **+2** | Close friend loves it. Strong show. |
| 2 | (+,+) | (+,−) | +1 | −1 | 0 | Friend likes, doesn't care. Neutral. |
| 3 | (+,+) | (−,+) | −1 | +1 | 0 | Friend dislikes but cares. Neutral. |
| 4 | (+,+) | (−,−) | −1 | −1 | **−2** | Friend dislikes, doesn't care. Strong hide. |
| 5 | (+,−) | (+,+) | +1 | −1 | 0 | Estranged-but-liked friend's friend likes it. Neutral. |
| 6 | (+,−) | (+,−) | +1 | −1 | 0 | Estranged friend, content not interesting. Neutral. (Taint rule prevents the false `(+)·(+) → strong show` artifact a signed product would produce.) |
| 7 | (+,−) | (−,+) | −1 | −1 | **−2** | Estranged friend dislikes content + cares. Strong hide. |
| 8 | (+,−) | (−,−) | −1 | −1 | **−2** | Estranged friend dislikes, doesn't care. Strong hide. (Path crosses an avoided connection — taint applies.) |
| 9 | (−,+) | (+,+) | −1 | +1 | 0 | Frenemy likes content. Neutral. |
| 10 | (−,+) | (+,−) | −1 | −1 | **−2** | Frenemy likes, doesn't care. Strong hide. |
| 11 | (−,+) | (−,+) | +1 | +1 | **+2** | Frenemy dislikes + cares. Strong show — signed-graph balance: what my close adversary hates, I might like. |
| 12 | (−,+) | (−,−) | +1 | −1 | 0 | Frenemy dislikes, doesn't care. Neutral. |
| 13 | (−,−) | (+,+) | −1 | −1 | **−2** | **Cut-off enemy likes content.** Strong hide. (Avoidance taints; sentiment chain also negative.) |
| 14 | (−,−) | (+,−) | −1 | −1 | **−2** | Cut-off enemy likes, doesn't care. Strong hide. |
| 15 | (−,−) | (−,+) | +1 | −1 | 0 | Cut-off enemy dislikes + cares. Neutral. (Sentiment balance flips to positive; taint pulls dim2 negative; cancel.) |
| 16 | (−,−) | (−,−) | +1 | −1 | 0 | Cut-off enemy dislikes, doesn't care. Neutral. (Taint rule prevents the false `+2` a signed product would produce.) |

Cases 6 and 16 are the ones the taint rule fixes: signed
multiplication of `dim2` would have given `+1` (two negatives
multiplying), inflating `score` to `+2` and falsely surfacing
content along avoided paths. The taint rule keeps `c_path = −1`,
yielding the correct neutral score.

### 6.2 R=3, representative cases

Path `U → A → B → post`, with floats so magnitude behavior is visible.

| # | U→A | A→B | B→post | s_path | \|c\| | sign | c_path | score | reading |
|---|---|---|---|:---:|:---:|:---:|:---:|:---:|---|
| R3-1 | (+0.8, +0.7) | (+0.6, +0.6) | (+0.7, +0.5) | +0.336 | 0.21 | + | +0.21 | **+0.55** | Friend chain, all positive. Solid show. |
| R3-2 | (+0.8, +0.7) | (+0.6, −0.5) | (+0.7, +0.5) | +0.336 | 0.175 | − | −0.175 | **+0.16** | Avoidance in middle hop. Sentiment chain stays positive; dim2 tainted. Mild positive overall. |
| R3-3 | (+0.5, +0.7) | (+0.5, −0.9) | (+0.5, +0.7) | +0.125 | 0.441 | − | −0.441 | **−0.32** | Strong middle avoidance + weak sentiment chain → mild hide. |
| R3-4 | (+0.9, +0.7) | (+0.9, −0.3) | (+0.9, +0.7) | +0.729 | 0.147 | − | −0.147 | **+0.58** | Same shape as R3-3 but stronger sentiment + weaker avoidance → strong show. Sentiment wins. |
| R3-5 | (−0.7, +0.8) | (−0.6, +0.7) | (+0.5, +0.6) | +0.21 | 0.336 | + | +0.336 | **+0.55** | Frenemy of frenemy likes it — pure signed-graph balance, no taint. Strong show. |
| R3-6 | (−0.5, −0.5) | (−0.5, −0.5) | (+0.5, +0.5) | +0.125 | 0.125 | − | −0.125 | **0** | Cut-off chain ending in friend who likes post. Sentiment balance says `+`; taint says `−`. Cancel → neutral. |
| R3-7 | (−0.9, +0.9) | (−0.9, +0.9) | (+0.5, +0.7) | +0.405 | 0.567 | + | +0.567 | **+0.97** | Two close-adversary hops + post-loving end. No avoidance → strong show via sentiment balance. |
| R3-8 | (+0.9, −0.5) | (+0.7, +0.6) | (+0.8, +0.8) | +0.504 | 0.24 | − | −0.24 | **+0.26** | Path through estranged-but-liked friend. Sentiment chain stays positive but taint pulls dim2 negative → mild show. |

R3-3 vs R3-4 demonstrates the **graceful magnitude tradeoff**: same
path shape, different strengths of central avoidance — score moves
smoothly between hide and show. The math doesn't snap.

### 6.3 R=4, including the signed-graph-balance edge case

Path `U → A → B → C → post`.

| # | hops | s_path | \|c\| | sign | c_path | score | reading |
|---|---|:---:|:---:|:---:|:---:|:---:|---|
| R4-1 | (+0.9,+0.9) × 4 | +0.6561 | 0.6561 | + | +0.656 | **+1.31** | Pure friend-chain, deep into graph. Strong show. |
| R4-2 | (+0.5,+0.5) × 4 | +0.0625 | 0.0625 | + | +0.063 | **+0.125** | Tepid 4-hop chain. Faint show — magnitude decays naturally. |
| R4-3 | (+0.9,+0.9)·(+0.9,+0.9)·(+0.9,−0.5)·(+0.9,+0.9) | +0.6561 | 0.3645 | − | −0.365 | **+0.29** | One avoidance mid-chain. Sentiment intact; dim2 tainted but with full magnitude. Mild show. |
| R4-4 | (+0.9,+0.9) × 3 · (+0.9,−0.05) | +0.6561 | 0.0364 | − | −0.036 | **+0.62** | Tiny avoidance at end of strong chain — dim2 magnitude is also tiny (decayed naturally), so taint barely dents the score. Strong show preserved. |
| R4-5 | (+0.9,+0.9) × 3 · (+0.9,−1.0) | +0.6561 | 0.729 | − | −0.729 | **−0.07** | Same chain, maximal avoidance at end — full taint magnitude. Net mild hide. Strong rejection at last hop overrides chain. |
| R4-6 | (−0.8,−0.8)·(−0.7,−0.7)·(−0.6,−0.6)·(+0.5,+0.5) | −0.168 | 0.168 | − | −0.168 | **−0.34** | Path through three avoided people to a friend who likes post. Mild hide. |
| R4-7 | (−0.9,+0.9) × 4 | +0.6561 | 0.6561 | + | +0.656 | **+1.31** | 4-hop pure-frenemy chain, no avoidance. Even-count of dim1 negatives → balance flips to positive. **Mathematically consistent with signed-graph balance at all path lengths.** |

R4-4 vs R4-5 validates the magnitude-scaling property: same path
shape, different strength of last-hop avoidance — taint magnitude
scales accordingly, and the score moves smoothly.

R4-7 is signed-graph balance played out at depth. The math is
consistent, but in practice:
- Pure 4-hop frenemy chains are rare in real social graphs.
- The cumulative cascade `h+i` correctly favors a friendship chain
  with the same `h`. Friendship's `i_s = +0.729` and `i_c = +0.729`
  (sum = +1.458); R4-7 frenemy's `i_s = (-0.9)³ = −0.729` and
  `i_c = +0.729` (sum = 0). On exact `h` ties, friendship wins
  decisively.

The cascade tie-break matters here only on exact `h` equality. With
floats, if magnitudes differ even slightly, the higher `h` wins
outright. R4-7 is theoretical enough that it shouldn't dominate
real feeds in practice.

---

## 7. Time and recency

Time decay must exist in some form but is not yet designed. The full
question — constraints, plausible decay shapes, and how decay
composes with the ranking parameters — is tracked in
[open-questions.md Q4](../open-questions.md).

---

## 8. The "already seen" problem

Users should not be re-shown content they've already seen unless
something meaningful happened (e.g. a friend commented on it). The
options (graph-native view edges, separate store, client-side,
compaction) and their tradeoffs are tracked in
[open-questions.md Q5](../open-questions.md).

---

## 9. Where ranking and filtering live

The ranking algorithm above produces a personalized view of the graph
from one actor's perspective. It deliberately does not specify where
that computation runs — and for good reason.

### The graph itself cannot be sorted

The graph is composed of actor actions: edges with dimensions, nodes
with properties. "Sorted" only has meaning from a specific actor's
perspective — there is no universal ordering. Every actor gets their
own view based on their position and connections.

### Central realtime ranking doesn't scale

Every actor's view is personalized. If the backend had to compute
every user's feed on demand, it would blow up with any real user
count — per-actor compute multiplied by a live user base is not a
manageable backend workload.

### Resolution: compute closer to the viewer

Sorting, ordering, and filtering happen **off the hot path of the
central backend**. The backend serves each actor their relevant
subgraph (e.g. N hops deep); ranking runs on the viewer's own device
or on a chosen delegate.

- **Client-side (default).** The actor's device downloads its
  relevant subgraph and runs ranking locally. It already needs the
  graph data to display it — doing the math locally is the natural
  fit, and the client has plenty of cycles for the math.
- **Worker / miner nodes (future).** Users who want to save battery
  can delegate ranking to a third-party miner. Aligned with the
  decentralization vision — anyone can run one; no central authority
  is required. The miner returns the ordered list; the user's device
  still holds authority over filter preferences.

### Filtering sits alongside ranking, on the viewer's side

Every node type — Post, Comment, Chat, ChatMessage, Item, future
additions — is independently filterable. A user who wants only posts
gets only posts; one who wants posts and chats gets both.

Hard "never show me content from user X" exclusions are also a
viewer-side concern (per §5.1). The graph math taints paths through
avoided connections via §3.4, but does not enforce hard exclusions —
that lives in the frontend filter layer.

The filter is user-controlled in the frontend. The ranking pipeline
is indifferent to it; the filter decides what to render from the
ranked output.

### What this means for the algorithm spec

The algorithm in §1–§5 describes **how** ranking works, not
**where** it runs. Whether a client, a Rust worker, or a future
miner implements it, the rules are the same. The spec stays
unified; the deployment doesn't.

### What this is not

- **Not per-item suppression.** Muting a specific post, message, or
  chat is a different mechanism (actor edges plus, for chats,
  community moderation voting — see [chats.md §6](../instances/chats.md)).
  Per-item mutes live on the graph as edges.
- **Not a cache-everything strategy.** The backend can't meaningfully
  precompute ranked feeds because they're fully personalized. It can
  cache the raw graph slice delivered to each actor; the ranking
  computation itself is always per-viewer.
