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
    - `dim2` is **signed connection-weight** (interest / relevance /
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
the interest chain continues via `c_path`; `(+0.7, 0)` zeros
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
tells us nothing about my interest in B — interest doesn't compose
the way sentiment does. Signed multiplication of `dim2` along a
path would produce sign flips that don't correspond to any real
pattern (two avoidances would compose to a positive "connection,"
which is meaningless).

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
  anywhere in the path flips the interest signal to negative,
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

### 3.5 Bot resistance via the `(0, 0)` severance edge

The math gives users a community-driven defense against bot clusters
that doesn't require any algorithmic gatekeeping. The mechanism rests
on the unique adversarial-robustness property of the `(0, 0)` edge —
the **severance edge** — and on the immovability of `h(t) = 0` under
full community severance.

The same mechanism applies to any cluster the broader community
wants to disengage from. The math operates on path-set properties,
not on cluster type — see §3.6.

#### Why bots can dial any non-zero score

A cluster with unbounded internal nodes and edges can amplify a
single live entry path into an arbitrary aggregate `h(t)`. The
number of paths through a cluster of branching factor `b` grows as
`b^(R−1)`; path contributions decay as `d(R) = 0.1^(R−1)`. Once
`b ≥ 10`, the path-sum series diverges; even with a hard `R ≤ 5–6`
traversal cap, achievable amplification of a single entry edge runs
to ~100× and beyond.

So, as long as **any** path from `U` into the cluster has
`dim1 ≠ 0` and a non-zero `dim2` magnitude end-to-end, bots can
dial `h(t)` to any value they choose — strongly positive, slightly
positive, slightly negative, deeply negative, anywhere.

This rules out any "near-zero jail" defined as an interval
`[−ε, 0]`: bots tune their amplification to land at `−ε − δ` and
re-enter the visible feed directly below the positive section.

#### Why exact `h(t) = 0` is special

The only score bots cannot tune **away from** is exact `h(t) = 0`,
and only when **every** path from `U` into the cluster has at least
one edge enforcing `dim1 = 0` *and* `dim2 = 0` simultaneously. The
kill rule (§3.2) zeros each path's `s_path` and `|c_path|`
independently and irrevocably; with both dims killed on every path,
the aggregate is forced to exactly `0` and no internal edge
construction recovers it.

The asymmetry between the dims is what makes both kills necessary:
- `dim1 ≠ 0` gives bots full freedom on `s_path` — both **sign
  inversion** (signed-graph balance: chain in another negative
  factor and the product flips) and **magnitude scaling**. They can
  land `s_path` at any real value.
- `dim2 ≠ 0` gives bots **magnitude freedom** on `|c_path|`. The
  taint sign is one-way and not bot-recoverable, but `|c_path|`
  magnitude can be scaled to any value via internal edges.

Either dim non-zero on an entry edge gives bots a lever — `dim1` a
stronger one (sign + magnitude), `dim2` a narrower one (magnitude
only) — and either is enough to move `h(t)` off exact zero.
`(0, 0)` is the unique edge shape that closes both channels in a
single declaration.

#### The severance edge is not the everyday signal

`(0, 0)` is reserved for **deliberate severance** — a declaration
that the target is outside the user's graph of relevance entirely.
`(+, +)`, `(−, +)`, `(+, −)`, `(−, −)` and floats in between remain
the normal vocabulary for affinity, distance, dislike, and
avoidance, all of which leave path products live and feed back into
`h(t)` via the ordinary math. A user who simply dislikes a target
uses `(−, −)`, not `(0, 0)`.

Severance is a stronger statement and has stronger consequences: it
kills both dim chains on every path through the edge, removes the
severing user as a transit node toward the severed account, and
contributes to placing the severed account into zero-jail (§5) when
the entry path-set is fully community-marked.

#### Three layered defenses

1. **Inbound edges don't affect feeds**
   ([graph-model.md §7](graph-model.md)). A cluster cannot insert
   itself into `U`'s feed by creating outgoing edges *toward* `U`.
   Influence requires `U` (or a transitive contact) to have an
   outgoing edge *into* the cluster.

2. **Non-engagement keeps clusters isolated.** Per the
   action-creates-edges rule
   ([graph-model.md §3](graph-model.md)), no actor edge is created
   without an explicit gesture. A user who simply ignores a cluster
   creates no path into it from their neighborhood.

3. **`(0, 0)` severance forces zero-jail.** When the community marks
   every entry edge into the cluster with `(0, 0)`, every path from
   `U` to any target inside the cluster has at least one severance
   edge along it. The kill rule forces `s_path = 0` and `c_path = 0`
   on every such path; the aggregate `h(t) = 0` exactly. The sort
   rule (§5) banishes targets at exact `h(t) = 0` to the bottom of
   the feed, invisible by default.

The severance is **community-driven**, not gatekeeping. The math
gives users a tool; communities use it. The fundamental constraint
is that clusters can always create infinitely more edges than real
users can — but they cannot bypass inbound directionality, cannot
manufacture outgoing edges from real users into themselves, and
cannot recover paths through severance edges.

When a cluster has a live entry point holding it open — a real
user with a non-`(0, 0)` outgoing edge into the cluster — §3.6
covers how the defense cascades to that user, how the math applies
uniformly to clusters of any composition, and how a self-redeeming
node returns to the graph.

### 3.6 Cascading severance and redemption

#### The transit-node problem

A cluster is held open to `U`'s feed by any real user with a
non-`(0, 0)` outgoing edge into the cluster. Severance against the
cluster is only complete when **every** such entry path is dead.

A single real user — whether knowingly malicious, unknowingly
captured by a sophisticated impersonation, or simply slow to update
their edges — can therefore keep the cluster reachable from a
viewer's neighborhood. The math doesn't distinguish "real user
innocently still connected" from "real user actively bridging the
cluster"; both are live transit nodes.

#### Cascading severance

The defense extends naturally to transit nodes. Traversal
transparency lets a viewer audit *how* a piece of content arrived
in their feed (see [graph-model.md](graph-model.md) for the
transparency principle). When the viewer sees cluster content
reaching them through a specific real user, they can sever the
transit node itself with `(0, 0)`. Every path that reached the
cluster through that transit node is now killed at the transit hop
in the kill rule (§3.2).

As more viewers cascade severance outward through the graph, the
cluster's reach contracts. Full severance is achieved when, for
every viewer, every path into the cluster passes through at least
one severance edge — at which point `h(t) = 0` exactly for every
target inside (§3.5), and §5's zero-jail banishes those targets
from view.

#### The math has no cluster-type category

The cascade applies uniformly to any cluster the broader community
wants to disengage from — bot networks, coordinated harassment
groups, ideological cliques, content the broader graph judges as
low-signal. The math operates on path-set properties; it does not
inspect cluster composition. The community decides cluster-by-
cluster via their severance edges, and the math executes
identically.

A cluster of real humans who behave as a closed group can therefore
itself be placed in zero-jail. **Internal interactions within the
cluster continue unchanged** — each member's outgoing edges to
other members are unaffected, so their feeds of each other still
function. The cluster only loses external reach. There is no
special category for "humans" vs. "bots"; there is only *reachable
from your graph* vs. *not*.

This is consistent with the architecture's broader stance: anyone
can fork or self-host (CLAUDE.md), and clusters that have been
severed from one part of the graph can continue to operate among
themselves or on a separate instance entirely. Severance is local
to the severing community; it is not a global ban.

#### Redemption

Append-only layers (see [layers.md](layers.md)) make severance
**reversible by the severed node's own action**. A user who has
been severed by their community — because they were holding a
cluster open, whether by ignorance, captured invitation, or earlier
intent — can:

1. **Update their own outgoing edges to the cluster to `(0, 0)`.**
   This appends a new severance layer on top of the existing layer
   stack. The old positive layer remains in history (transparency);
   the top-of-stack value is now severance, and the user is no
   longer a live transit node.
2. **Wait for community signal to update.** Other users observing
   the updated edge state can choose to add a new positive layer to
   their own outgoing edges toward the redeeming user, restoring
   positive path flow. This is the same append-only mechanism that
   placed them in zero-jail; it runs in the other direction.

The redemption is fully transparent: the layer stack records the
sequence of stances over time. The severed user's history is
preserved (they had positive edges to the cluster, then severed
them); the community's response is preserved (they severed the
user, then restored). Nothing is hidden, and the community's
trust decision is made against the visible record.

Discovery of one's own zero-jail state and the specific
gestures that invite re-edges from the community are covered
in §3.7.

### 3.7 Post-severance surfaces

§3.5 and §3.6 specify the math: severance kills paths via the
kill rule, cascading severance and append-only redemption
operate on edge values. This section specifies the
**surfaces** built on top of that math — how a severed node
discovers their state and identifies the cause edge, and how
a severer learns when someone they severed has updated their
outgoing edges and might warrant re-evaluation.

Three properties hold throughout:

- **All surfaces are client- or miner-computed from existing
  graph state.** No new edge types, no new node types beyond
  what the graph already supports, no backend logic beyond
  the subgraph it already serves. The surfaces are
  derivations over the layer-stack data the client already
  has access to (or can fetch on demand for self-queries).
- **Discovery is loud; redemption is deliberate.** A severed
  node should learn quickly so they can act — the discovery
  surface is continuous and visible. A severer reviewing
  whether to restore a severed account requires deliberate
  per-severer review with the full layer-stack history
  visible, not automatic mass restoration. One real user
  re-attached to a live bot bridge is a network failure, so
  the bar for restoration is the severer's own gesture, not
  any system automatism.
- **Frontend latitude on presentation.** This section spells
  out what data is available and what the client can derive
  from it. Visual styling, notification frequency,
  aggregation thresholds, and badge design are frontend
  concerns and intentionally not specified here. Where path
  cutoffs or scoring formulas appear below, treat them as
  frontend-tunable defaults like `d(R)` (§4.1) — guidance
  surfaced as tooltips, not enforced rules.

#### 3.7.1 Severance discovery — the inbound side

Inbound edges do not affect the viewer's feed
([graph-model.md §7](graph-model.md)), so the feed-pull
traversal does not include them. Discovering one's own
severance state therefore requires an **explicit
self-query** — the client (or a delegated miner) requests
inbound state on demand, separately from the feed pull. The
data is on the graph and traversable; it is just not
pre-loaded for free.

For node `U` running this self-query, two derived surfaces
are available:

**Severance pattern.** Count inbound edges with top-of-stack
value `(0, 0)`. For each, identify the severer `S` and the
severance layer's timestamp. The directional edge structure
provides a natural per-edge weighting:

- A severance from `S` where `U` has an outbound edge to `S`
  with non-`(0, 0)` top-of-stack — `U` considers `S` part of
  their network. **Strong per-edge signal.**
- A severance from `S` where `U` has no outbound edge to `S`,
  or has `(0, 0)` top-of-stack toward `S` — `S` is outside
  `U`'s outbound network. **Weaker per-edge, but volume
  matters.** A celebrity or hub will have thousands of
  inbound edges from non-trusted-network users under normal
  conditions; a sudden burst of severance from this category
  is itself a meaningful signal even though no individual
  severer is in `U`'s network.

Frontends present these two categories with different
prominence — neither is dismissed. Trusted-network severance
is the per-edge alarm; stranger-severance volume is the
population-level alarm.

**Outbound-edge audit list.** List `U`'s outbound edges with
metadata derivable from the layer stacks: when each was
created, top-of-stack values, layer count. This is the audit
material — `U` reviews their outbound list with the severance
pattern as context.

**The cause-pointing gap.** No automatic on-graph signal
points from "you are severed" to "this specific outbound edge
is the cause." Severance walks backward from a cluster to
transit nodes; it does not propagate forward to sever the
cluster endpoints (the trusted-network severers update their
edge to `U`, not to the cluster behind `U`, because they had
no edge there to update — see §3.6). The cause information
lives at the severers' content traversals, not in the inbound
severance data the discovery surface sees.

The cause-pointing aid lives in §3.7.2 (auto-detection via
path patterns) and §3.7.3 (community bot-defense posts as
supplementary evidence).

#### 3.7.2 Bot-cluster identification — auto-detection from path patterns

The cause-pointing gap closes via direct analysis of the
viewer's subgraph for path patterns characteristic of bot
bridges. This is graph math on existing state — no AI
classification, no central allow/blocklist, no per-account
verdict beyond what the path structure says. The client (or a
delegated miner) computes the analysis from the same
subgraph it pulls for ranking; the path-set the analysis
reads is the same path-set used to compute `h(t)` (§4).

**The hourglass signal.** For viewer `U` and any node `B` in
`U`'s outbound subgraph, examine the paths from `U` to
content and accounts behind `B`. Two patterns characterize:

- **Fan pattern.** Content `t` behind `B` is reachable from
  `U` via diverse paths through multiple intermediates — many
  distinct chains, no common bottleneck. `B` is one of
  several routes to that part of the graph. Normal
  connectedness.
- **Hourglass pattern.** Content `t` behind `B` is reachable
  from `U` *only through `B`* (or with `B` on the
  overwhelming majority of paths). `B` is the sole bridge
  into that subgraph from `U`'s perspective.

A pure hourglass is the bot-bridge signature. The cluster
behind that node has no other entry into `U`'s graph — exactly
the topology of a bot cluster a real user has bridged into.
Bots cannot manufacture outgoing edges from real users
([graph-model.md §7](graph-model.md)), so the cluster's only
entry points are the legitimate user-created edges. If only
one such edge exists from `U`'s reachable subgraph (or
cascading severance has reduced the cluster's open bridges to
one), the path pattern is unambiguous.

**Differentiating from legit hubs.** Influencers, popular
accounts, and big bridging nodes also generate hourglass-shaped
paths — many users reach a lot of content through them. The
differentiator is whether the content behind the suspect bridge
has alternative paths into the broader graph. Real content
circulates through multiple channels; bot content typically
does not.

For each suspect bridge `B`, the analysis samples some
downstream content and checks: is this content reachable from
`U` via *any* path that does not go through `B`? Even one
alternative path within the traversal window indicates `B` is
one route among several (legit hub). No alternative paths
indicates `B` is the sole route (suspected bot bridge).

The check is bounded computation — a 1–2 hop traversal beyond
`B`'s downstream targets, looking for any inbound edge that
does not trace back through `B`. False positives are still
possible (a brand-new viral account with one early bridge would
look bot-shaped briefly), but the heuristic is sharp enough for
first-cut detection.

**Detection sharpens with severance.** In a fresh, fully
connected graph a bot cluster may have multiple live entries
and the hourglass pattern is weak. As soon as any user severs
one of the entries, the cluster's reach contracts and the
hourglass forms more clearly for everyone else. The first
detection often comes from a manually-identified bot
(triggering a §3.7.3 post); auto-detection then takes over for
the rest of the network as the cluster's bridges narrow. The
two mechanisms reinforce each other.

**Bot-defense page.** The frontend assembles a bot-defense
page from this analysis: a list of suspect bridge nodes
detected in the viewer's subgraph, each with a frontend-computed
**score** representing the likelihood of being a bot bridge.
Inputs to the score include (at minimum) hourglass-purity of
the path pattern and the result of the alternative-paths check.
Frontends may add additional inputs; the doc does not specify a
formula. The page also surfaces the viewer's path to each
suspect — the actual chain of intermediates — so users who
want to verify the score's basis can drill in.

**Path-length-aware action guidance.** The action recommendation
for each suspect depends on hop count. Frontends present these
as tooltips, not enforcement:

- **1 hop** (direct edge `viewer → suspect`): clean fix by
  updating the edge to `(0, 0)`. No collateral.
- **2 hops** (`viewer → C → suspect`): updating `viewer → C`
  to `(0, 0)` kills the path but with collateral — the viewer
  loses everything else flowing through `C`, not just the bot
  content. Frontend can surface "this also disconnects you
  from N other accounts you reach via `C`." Alternative:
  signal `C` to act (out-of-band, or via the post mechanism in
  §3.7.3).
- **3+ hops**: graph-level severance is high-collateral and
  rarely worth the cost — the viewer is far from the bridge,
  and closer-to-bridge users are the natural fixers.
  Recommended approach: use the frontend filter (per §5.1) to
  block content from the suspect directly, or signal a closer
  user (the path itself names them — `D` in
  `viewer → C → D → suspect` is the cheapest fixer).

The cutoffs are frontend-tunable defaults. Some users may
prefer aggressive (2-hop maximum direct action); others
conservative (1-hop only). The doc does not enforce a number.

**No automatic action.** Detection populates the page; the user
always decides whether and how to act. The math does not
auto-banish on hourglass detection. Severance still requires
the user's `(0, 0)` gesture, exactly as specified in §3.5–§3.6.

#### 3.7.3 Community bot-defense posts — supplementary evidence

Auto-detection (§3.7.2) surfaces *structural* suspicion. A
**community bot-defense post** adds what structure cannot
capture: human-evaluated context. A real user who has
identified a suspected bot publishes a regular post on the
graph with a structural edge to a `bot-defense` Hashtag node.
The post body holds the evidence the math can't see —
screenshots of bot-like behavior, profile observations,
content samples, written explanation — and links to the
suspected node by ID.

This is not a new graph mechanism. Posts and structural edges
to Hashtag nodes already exist (per the data model). The
bot-defense post is a **usage convention** that frontends
recognize via the hashtag. Hashtag identity is content-
addressed (UUIDv5 of the canonical name; see
[data-model.md "Node identity strategies"](../implementation/data-model.md)),
so any user creating `bot-defense` independently lands on the
same Hashtag node automatically.

**Authorship is open.** Anyone can author a bot-defense post.
The post inherits the graph's existing trust mechanisms:

- **Bot-authored posts don't reach trusted feeds.** Per the
  inbound-edges-don't-affect-feeds rule
  ([graph-model.md §7](graph-model.md)), a bot's post reaches
  viewer `V` only if `V` (or a transitive contact) has an
  outgoing edge into the bot's neighborhood. False accusations
  by bot accounts about innocent targets mostly stay in the
  bots' own cluster.
- **False accusers are themselves severable.** A real user
  publishing bad-faith bot-defense posts faces the same
  community severance mechanism. The math doesn't carve out
  a protected category for accusers.
- **Source-distribution check.** The score for community posts
  on the bot-defense page also accounts for where the post's
  reach concentrates. A post reaching the viewer with high
  `h(t)` only because a bot cluster is amplifying it from
  inside — even when there is also fan-pattern reach from
  trusted users alongside — has its score adjusted down. The
  signature is the same hourglass-plus-fan combination as in
  §3.7.2: a sudden burst of cluster-internal engagement on a
  post that otherwise has organic reach is a manipulation
  pattern, and the score down-weights it accordingly.

**Surfacing on the bot-defense page.** Community posts appear
alongside auto-detected suspects from §3.7.2. The page shows
both signal sources together — they answer the same question
from different angles:

- Auto-detection says "this node has hourglass-shaped reach
  into your subgraph."
- A community post says "this node is doing X, and here is
  the evidence."

The two reinforce each other. A node flagged by both is
high-confidence. A node flagged by only one is worth
investigating but less conclusive.

**The natural workflow.** A viewer notices an auto-detection
ping — "hourglass shape detected at node `B`." They check,
agree, and sever (`B` becomes `(0, 0)` from their outbound).
The frontend can then offer to **scaffold a bot-defense
post** about `B` — pre-filling the body with structural facts
(the path the viewer just severed, hourglass score, layer-
stack snapshot of `B`'s outbound at time of severance) and
leaving the viewer to add free-text observations. The post
then propagates to others' bot-defense pages, accelerating
community detection. None of this is required — the viewer
can sever silently — but the option lowers friction for
spreading the signal.

**Path-matching for community posts.** The same path-matching
and hop-count action guidance from §3.7.2 apply: for each
community post, the client computes whether the viewer has
paths to the accused account, and presents action options
based on hop count. Posts about accounts the viewer has no
path to are interesting context but not actionable for that
viewer.

**Generalizes beyond bots.** Per §3.6, the math operates on
path-set properties and applies uniformly to any cluster the
broader community wants to disengage from — coordinated
harassment groups, ideological cliques, content the broader
graph judges as low-signal. Community posts can target any
such cluster; the `bot-defense` tag is shorthand, not a
type-restriction. The auto-detection mechanism in §3.7.2
similarly does not check for "botness" specifically — it
checks for path patterns characteristic of *isolated clusters
reachable through narrow bridges*, which captures all of the
above.

#### 3.7.4 Severance redemption — the outbound side

Per §3.6, append-only layers make severance reversible: the
severed node updates their own outbound edges to the cluster
to `(0, 0)`, and community members can append a new positive
layer to their own outbound edge toward the redeeming node.
The math allows this; the surface for the severer makes the
redemption signal visible.

**What signals redemption.** The clean answer comes from
applying §3.7.2's hourglass detection to `T`'s outbound
edges. `T` is in the redeemed state when they have **no
remaining positive outbound edges to nodes exhibiting
hourglass-bridge patterns** — `T` no longer holds open any
isolated cluster reachable through a narrow bridge.

This is graph-derivable; the severer does not need to
remember why they severed `T`. Severance edges do not carry
reasons (graph state does not represent intent), and the
severer's app cannot reliably reconstruct intent from
inbound edges weeks or months later. The hourglass check
sidesteps the question by asking "does `T` *currently* bridge
into any suspect cluster?" — a property of the graph state
right now, not of past history.

**The check is binary, not gradient.** A genuine transit-node
case has one or two outbound edges to a suspect cluster,
typically formed without scrutiny (an invite accepted
casually, an early positive engagement that aged badly).
Cleaning those up is a small, finite act. A user with *many*
positive outbound edges to suspect bridges is much more
likely a member of the cluster themselves than a transit, and
the discovery and auto-detection surfaces (§3.7.1, §3.7.2)
should already classify them accordingly — they are the
cluster's body, not its bridge. The redemption check is thus
naturally binary: `T` has no remaining bridges (redeemed), or
`T` still has bridges (not redeemed). There is no "halfway
redeemed" state worth surfacing as such.

**Computing the check.** The severer's client makes an
explicit self-query for `T`'s outbound state — analogous to
the inbound self-query in §3.7.1, since `T` is severed and
not in the severer's normal feed pull. For each of `T`'s
positive-valued outbound edges to target `V`, the client runs
the §3.7.2 hourglass-and-alternative-paths analysis on `V`
from `T`'s subgraph perspective. If any `V` classifies as a
suspect bridge, `T` still has open bridges. If none do, `T`'s
bridges are clean.

**Surfacing.** The severer's client surfaces:

- A status: "`T` has [N] positive outbound edges to suspect
  bridges. `T` appears [redeemed | still bridging]."
- The list of bridges (if any remain) for inspection.
- Full layer-stack history of `T`'s outbound, available for
  audit. The severer can see when each edge was created, when
  (and if) it was severed, and the sequence of changes over
  time. The decision to restore is made against this complete
  record, not against a single point-in-time signal.

**Decision is individual.** The math signal is one input; the
severer's own judgment and the visible layer history are
others. The severer may add a new positive layer to `S → T`
(restoring `T` from `S`'s perspective), wait for more
evidence, or do nothing. **No automatic restoration.**

The friction is intentional. Per the §3.7 intro: a severed
user re-attaching to a live bot bridge is a network failure.
Per-severer individual review with full history visible is the
design's answer.

**Ongoing watch.** The check runs continuously over the
severer's watch list. If `T` re-attaches to a suspect bridge
after a previous clean state, the signal flips back, and the
severer's client surfaces the change.

#### 3.7.5 Self-redemption posts

The structural redemption signal in §3.7.4 (`T`'s outbound has
no remaining suspect-bridge edges) is necessary but easy to
miss — the severer has to be running the outbound-watch query
and looking. To make redemption more discoverable and to add
human-evaluated context, the severed user can author a
**self-redemption post**, symmetric to the community
bot-defense posts in §3.7.3.

The post is a regular post on the graph with a structural edge
to the same `bot-defense` Hashtag node (or a sibling redemption
hashtag if the data model later distinguishes; the surface
treats them equivalently). The body explains the fix in `T`'s
own
words — what edge they updated, what they observed, why they
believe they were a transit node, what they will do
differently.

**Visibility despite severance.** The severed user's own feed
is unaffected by severance (their feed runs on their outbound
edges, which still work). Their *posts*, however, are
zero-jailed for anyone who has severed them — the path through
their content does not reach those severers' main feeds. The
self-redemption post needs a different surface to reach the
severers.

The frontend's "review severed accounts" view (the surface
from §3.7.4) is exactly that. The severer's client, which
already runs the outbound-watch self-query, also fetches
recent posts from severed accounts and surfaces them in this
view. Self-redemption posts (recognized via the tag) are
highlighted separately within that view.

**Cross-checking against graph state.** The severer can
compare the post's claims to the §3.7.4 structural signal.
Post says "I severed my edge to V," graph confirms the
hourglass check passes → consistent. Post claims redemption
but graph still shows positive outbound to suspect bridges →
inconsistent (likely a false claim, or `T` misunderstands what
they need to fix). The severer trusts the math first, then
reads the post for context.

**Same trust mechanisms apply.** A bot can publish a
self-redemption post claiming innocence. The same defenses
cover it: the post is evaluated by each severer individually,
against the full layer-stack history visible on graph and
against the structural redemption signal. Post claims and
graph state must be consistent for the severer to accept.

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

Steep decay reflects "graph proximity is the most important factor."
Direct connections (R=1) carry full weight; each additional hop
reduces the path's contribution by 10×. Bots and viral-distant
content cannot dominate a user's feed by sheer multi-path count
alone — at any reasonable graph density, distant paths contribute
proportionally to how far they are. (Note: this is about path length
through the graph, not `dim2` interest. Direct ≠ "high interest";
a high-interest target many hops away still gets steep d(R) decay,
and a low-interest target right next to you carries full d(1) weight
on whatever signal its dims contribute.)

The decay function is a frontend-tunable parameter. A user who wants
a broader-network feed can soften the decay (e.g., `0.5^(R-1)`); one
who wants only direct-friend signal can steepen it (e.g.,
`0.01^(R-1)`). The default is calibrated so that a single strong
R=2 path roughly matches ~15 strong R=3 paths' aggregate
contribution — balancing direct signal with friend-of-friend buzz.

A separate **time-decay** factor `f(Δt)` is applied alongside `d(R)`
on the reactor edge of each path — see §7. Both are
frontend-tunable.

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

| Symbol | Name | Sentiment component (`*_s`) | Interest component (`*_c`) |
|---|---|---|---|
| `h` | personal opinion | `H_s = ∑_π d(R_π) · f(Δt_π) · s_path(π)` over all paths to `t` | `H_c = ∑_π d(R_π) · f(Δt_π) · c_path(π)` over all paths to `t` |
| `i` | personal reach | `I_s = ∑_π d(R_π) · f(Δt_π) · s_path_R−1(π)` over first R−1 edges of each path | `I_c = ∑_π d(R_π) · f(Δt_π) · c_path_R−1(π)` over first R−1 edges |
| `j` | absolute opinion | `J_s = ∑_B f(Δt_B→t) · dim1(B → t)` over reactors `B` (signed) | `J_c = ∑_B f(Δt_B→t) · dim2(B → t)` over reactors (signed) |
| `k` | absolute reach | `K_s = ∑_B f(Δt_B→t) · \|dim1(B → t)\|` over reactors | `K_c = ∑_B f(Δt_B→t) · \|dim2(B → t)\|` over reactors |

`f(Δt)` is the time-decay factor on the reactor edge (the last
actor edge in the path, or `B → t` directly for `j` and `k`); see
§7 for its definition and rationale. `Δt` is the elapsed time
since that edge's top layer was added.

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
or `α = 1, β = 2` to favor interest-weighted ordering.

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
weak R=2 path, while preserving "graph proximity matters most"
through `d(R)`'s steep decay.

Targets with `h(t) > 0` appear at the top of the feed; `h(t) < 0`
at the bottom. Negatives are **visible**, not banished — a friend
strongly disliking something is meaningful information for the
viewer to be aware of, and the graph's transparency principle
favors showing them over hiding. They sort below positives because
the score itself is negative; that's it.

**Exact `h(t) = 0` is the zero-jail.** Targets whose aggregate `h(t)`
is exactly zero are banished from the feed — sorted below the
negatives, into nothingness. This is the only sort position
**immovable under unbounded internal cluster amplification**: a
cluster with infinite internal nodes can tune its target's `h(t)`
to any non-zero value (positive or negative) but cannot move it off
exact zero once every path from `U` into the cluster has at least
one severance edge `(0, 0)` along it. Zero-jail is the math-level
realization of full community severance (see §3.5).

Why exact, not an `[−ε, 0]` interval: bots facing an interval-jail
simply tune their amplification to land at `−ε − δ` and re-enter
the visible feed below the positives. Only the single point
`h(t) = 0` is unreachable by bot tuning, and only when full
severance is in place. The interval cut is not defensible; the
point cut is.

A target that produces `h(t) = 0` from cancellation of positive
and negative path contributions (no severance involved) lands in
the same bucket. With float math, exact cancellation is rare in
practice; in sparse graphs or with `+1/0/-1` integer values where
it can happen, the outcome — "the graph's signal toward this
target sums to neutral" — is a reasonable thing to push out of the
default feed.

The cascade activates only on **strict equality** at each level.
With float math, exact ties on `h` are rare; the cascade kicks in
mostly for sparse graphs (where many targets have `h ≈ 0` exactly)
and for users who default to `+1/0/-1` integer values (where ties
are common). `S` (the intrinsic node scalar) is the deepest
fallback — see [open-questions.md](../open-questions.md) for its
derivation.

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

#### The severance edge `(0, 0)` — special case

The 16-case table enumerates the ordinary `±1` vocabulary.
`(+, +)`, `(−, +)`, `(+, −)`, `(−, −)` and floats in between
remain the normal way to express affinity, distance, dislike,
and avoidance. The **severance edge** `(0, 0)` is qualitatively
different — a deliberate declaration that the target is outside
the user's graph of relevance (see §3.5). One representative
case at R=2:

| # | U→A | A→post | s_path | c_path | score | reading |
|---|---|---|:---:|:---:|:---:|---|
| 17 | (0, 0) | (anything) | 0 | 0 | **0** | Severance at U's outgoing edge. Both dim chains killed at the entry hop; nothing downstream recovers either. The path contributes 0 to `h(t)` regardless of what `A → post` is. |

`(0, 0)` is not the everyday signal — it is reserved for the
deliberate cut described in §3.5, where its consequences (transit
removal, contribution to zero-jail under full community severance)
are spelled out.

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

The path math in §3–§5 has no time component on its own. Without
one, ranking exhibits a **cold-start failure**: a brand-new post
from a close friend can rank below an old viral post that no one
is currently engaging with, because accumulated multi-path signal
outweighs a single fresh path. Worked example below
(§7.3) shows the gap is concretely ~3.5× under the default `d(R)`.

Time decay closes this gap by attenuating contributions from
**stale reactor activity**, leaving fresh signal at full weight.

### 7.1 What decays — reactor-edge top-layer age

Decay anchors on the **top-layer timestamp of the reactor edge**:
the last actor edge in the path (`B → t`), the edge that
expresses a stance toward the target. Per
[layers.md](layers.md), every edge has a stack of timestamped
layers; the top layer is the most recent stance expression.

A new layer on the reactor edge — a friend re-liking,
commenting again, updating their reaction — resets the age clock
and restores full freshness. This is how old content resurfaces:
not through a special "resurface" mechanism, but because new
reactor-edge layers naturally re-enter the math at full weight
through the same formulas. The append-only layer system is the
mechanism.

**Intermediate edges don't decay.** For a path `U → A → B → t`,
the time-decay factor is applied only on the `B → t` hop. The
`U → A` and `A → B` edges are full-weight regardless of when
their top layer was added. This carries the **stances-not-events**
rule (§3, [graph-model.md §3](graph-model.md)) through to time:
silence on a relationship edge is not a partial revocation of the
stance — the stance still holds until the actor changes it.
Active-relationship signal lives in **layer count**
([open-questions.md Q1](../open-questions.md)) — frequency of
interaction, not recency of last interaction.

**Post-node age has no separate decay.** It falls out
automatically: the **authorship edge** is itself a normal actor
edge, and is the reactor edge for the path through the author
(per [authorship.md:1-10](authorship.md#L1-L10)). Its top layer
ages with the post. An old post with no engagement → only the
stale authorship path survives → naturally decayed by `f(Δt)`
on that hop. An old post with new engagement → fresh reactor
edges from new reactors carry the path at full weight. Node age
never enters the math directly.

### 7.2 Composition — scalar multiplier per path

Decay is a positive scalar in `(0, 1]` multiplied into each
path's contribution to the metric, alongside `d(R)`. The full
formulas are stated in §4.2; in summary, every term in the sum
that defines `H_s, H_c, I_s, I_c, J_s, J_c, K_s, K_c` carries an
`f(Δt)` factor on the reactor edge.

Because decay is a positive scalar, it does not interact with the
**kill rule** (§3.2) — a `0` in a dim chain still zeros that
dim's path product irreversibly; decay only scales the surviving
contribution. Dim values themselves are never mangled by time,
preserving their meaning as **stances** (§3.3 signed-graph
balance reasoning depends on the dim values being the actor's
expressed stance, not a time-mangled approximation of it).

### 7.3 Shape — exponential, 30-day half-life, frontend-tunable

Default decay function:

```
f(Δt) = 0.5^(Δt / 30 days)         (default)
```

So `f(0d) = 1.0`, `f(30d) = 0.5`, `f(90d) = 0.125`, `f(1y) ≈ 7×10⁻⁴`.

Worked cold-start example, with and without decay:

**Setup.**
- `U → A`: close friend, `(+0.9, +0.9)`, fresh.
- A just authored post P. Authorship edge `A → P`: `(+0.9, +0.9)`,
  fresh.
- `U → B`: also a close friend, `(+0.8, +0.8)`.
- B authored post Q 3 years ago. `B → Q`: `(+0.9, +0.9)`, top
  layer 3 years old.
- 100 R=3 paths reach Q via U's network. For each:
  `U → C` = `(+0.5, +0.5)`, `C → reactor` = `(+0.6, +0.6)`,
  `reactor → Q` = `(+0.7, +0.7)`.

**Without decay:**
- `h(P) = d(2) · (s_path + c_path) = 0.1 · (0.81 + 0.81) = 0.162`.
- `h(Q)` direct: `0.1 · (0.72 + 0.72) = 0.144`.
- `h(Q)` per R=3 path: `0.01 · (0.21 + 0.21) = 0.0042`. Times 100:
  `0.42`.
- `h(Q) = 0.144 + 0.42 = 0.564`. **Q wins ~3.5× over P.**

**With decay (default 30-day half-life):**
- P: authorship edge fresh → `f = 1.0` → `h(P) = 0.162`.
- Q direct: `f(1095d) ≈ 8×10⁻¹²` → contribution collapses to ~0.
- Q reactor paths: assume 10 of 100 reactor edges are recent
  (≤30d, average `f ≈ 0.7`); the remaining 90 are years old
  (`f ≈ 0`).
  - Recent: `10 · 0.0042 · 0.7 ≈ 0.029`.
  - Old: ≈ 0.
- `h(Q) ≈ 0.029`. **P wins ~5.5× over Q.** Cold start fixed.

**Currently-surging old post.** If 50 of the 100 reactor edges
are fresh instead of 10 (the post is currently being re-engaged
across the network), `h(Q) ≈ 0.147` — under P (0.162) but
visibly competitive. With even fresher reactor activity (≤7d,
`f ≈ 0.85`) and stronger reactor edges, Q can overtake P. **This
is correct behavior**: 50 people in U's network currently
engaging with content is genuinely a stronger signal than one
fresh post from one friend. The §4.1 calibration ("a single
strong R=2 path roughly matches ~15 strong R=3 paths") is
preserved on freshly-active content; only stale aggregate signal
is suppressed.

**Frontend tunability.** Same pattern as `d(R)` (§4.1). A user
who wants longer-tail visibility softens the half-life (e.g. 90
days). One who wants strict freshness shortens it (e.g. 7 days).
Setting `f(Δt) = 1` constant disables decay entirely — useful as
an opt-in "no-decay" sort for users who want pure-graph signal.

### 7.4 What this does not solve

Time decay attenuates content that is **old and quiet**. It does
**not** suppress content that is **old, currently active, and
already seen by U**. The "already seen" problem is tracked
separately as [open-questions.md Q5](../open-questions.md); it
requires its own mechanism (client-side seen-set, per-viewer
markers, or similar) and is not absorbed by reactor-edge decay.

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
