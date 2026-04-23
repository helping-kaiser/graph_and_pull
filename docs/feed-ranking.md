# Ranking via Weighted Graph Connections

A general framework for ranking and ordering **target nodes** in a **signed,
weighted graph**, relative to a chosen **root node**.

The social-feed setup (User -> other Users -> Posts) that originally motivated
this is shown as an *example* at the end. The rule itself is layer-agnostic --
it applies to any signed graph where a root node wants to rank some set of
target nodes reachable through intermediate connections.

---

## 1. Setup (general)

A **signed graph** with:
- a **root node** `U` -- the perspective we rank from,
- one or more layers of **intermediate nodes**,
- a set of **target nodes** -- what we're ranking,
- every edge carrying a sign `+` or `-` (and optionally a weight).

The algorithm's job: given this graph, produce an ordered list of the target
nodes as seen from `U`.

---

## 2. Parameters

| Symbol | Name | Meaning |
|--------|------|---------|
| `R` | Real number of graph hops | Path length (number of edges) from `U` to the target. Targets with the same `R` form a comparison group. In the example, `R = 2` for both posts. |
| `S` | Scalar value of a node | An intrinsic scalar assigned to each node. Used in the **sort** phase to pre-order nodes within an `R` group. |

---

## 3. Per-target metrics

Four quantities are computed per target node from the signed edges in its
neighborhood back to `U`. Two are **relative** (weighted by `U`'s
relationships); two are **absolute** (independent of `U`).

| Symbol | Name | Alias | Interpretation |
|--------|------|-------|----------------|
| `h` | relative opinion | *personal relevance* (a.k.a. affinity / relevancy) | Net opinion toward the target from `U`'s connected nodes, each weighted by `U`'s relationship to it. A `+`-linked node liking the target contributes `+`; a `-`-linked node liking it contributes `-`; etc. |
| `i` | relative connection | *importance* | Strength / number of connections between `U` and the nodes that reacted to the target. |
| `j` | absolute opinion | *controversy* | Net opinion on the target, independent of `U` -- raw sentiment across all reacting nodes. |
| `k` | absolute connection | *popularity* | Raw number of interactions with the target, independent of `U`. |

---

## 4. Algorithm

The ranking runs in two phases: a coarse **sort** into buckets, then a finer
**order** within buckets using cumulative tie-breakers.

### Step 1 -- Sort (bucketing, descending priority)

```
R  ->  S  ->  k  ->  j  ->  i  ->  h
```

Targets are bucketed first by reach `R`, then by scalar `S`, then popularity
`k`, controversy `j`, importance `i`, and finally personal relevance `h`.

### Step 2 -- Order (final sequence with cumulative tie-breakers)

Within each `R` group, the final order uses cumulative sums starting from `h`:

```
R  ->  h
        -> if equal:  h + i
        -> if equal:  h + i + j
        -> if equal:  h + i + j + k
```

1. Order primarily by `h` (personal relevance).
2. If tied -- compare by `h + i`.
3. If still tied -- compare by `h + i + j`.
4. If still tied -- compare by `h + i + j + k`.

> **Why `S` doesn't reappear in the order step:** during the sort step, nodes
> are already placed in scalar order. The cumulative tie-breakers `h`,
> `h+i`, `h+i+j`, `h+i+j+k` resolve any remaining ties inside that scalar
> order, so re-applying `S` at the end would not change the result.

---

## 5. Example -- Social feed (User -> User -> Post)

This is the specific instance the algorithm was developed against, and is only
one possible shape the graph can take.

### Node roles
- **Root**: the viewing user `U`.
- **Intermediate layer** (other users):
  - `f_A`, `f_B` -- friends, signed `+` from `U`.
  - `f'_C`, `f'_D` -- disliked, signed `-` from `U`.
- **Targets**: `Post 1`, `Post 2`.
- **Reach**: `R = 2` for both posts (two hops: `U -> user -> post`).

### Edge signs in this example

**`U` -> other users** (U's feelings):

| Edge | Sign |
|------|:----:|
| `U -> f_A`  | `+` |
| `U -> f_B`  | `+` |
| `U -> f'_C` | `-` |
| `U -> f'_D` | `-` |

**Other users -> posts** -- each intermediate user is connected to **both**
posts:

| From    | -> Post 1 | -> Post 2 |
|---------|:---------:|:---------:|
| `f_A`   | `+` | `+` |
| `f_B`   | `+` | `-` |
| `f'_C`  | `+` | `+` |
| `f'_D`  | `+` | `-` |

### Diagram

```
                               U
                     +  /    +\    -/     \ -
                       /      \    /       \
                     f_A     f_B  f'_C     f'_D
                      |\      |\   /|      /|
                      | \     | \ / |     / |
                      |  \    |  X  |    /  |          (each f-node has
                      |   \   | / \ |   /   |           one edge to EACH
                      v    v  vv   vv  v    v           post -- 4 edges
                   +--------------+ +--------------+    arrive at each post)
                   |   Post 1     | |   Post 2     |
                   | fA:+  fB:+   | | fA:+  fB:-   |
                   | f'C:+ f'D:+  | | f'C:+ f'D:-  |
                   +--------------+ +--------------+
```

If the ASCII crossings are hard to parse, the edge tables above are the
canonical source of truth.

### Metric matrices for this example

Each matrix has one row per intermediate user. The contribution rule per
metric is:

| Metric | Per-user contribution |
|--------|-----------------------|
| `h` | `sign(U -> user)  *  sign(user -> post)` |
| `i` | `sign(U -> user)` (only for users who reacted) |
| `j` | `sign(user -> post)` |
| `k` | `1` for each user who reacted |

The sum of the contribution column is the value of that metric for the post.

#### Post 1 -- all f-nodes reacted `+`

**Matrix `h` (relative opinion)**

| User   | `sign(U -> user)` | `sign(user -> post)` | contribution |
|--------|:-----------------:|:--------------------:|:------------:|
| f_A    | `+`               | `+`                  | `+`          |
| f_B    | `+`               | `+`                  | `+`          |
| f'_C   | `-`               | `+`                  | `-`          |
| f'_D   | `-`               | `+`                  | `-`          |
| **Sum**|                   |                      | **`0`**      |

**Matrix `i` (relative connection)**

| User   | `sign(U -> user)` | contribution |
|--------|:-----------------:|:------------:|
| f_A    | `+`               | `+`          |
| f_B    | `+`               | `+`          |
| f'_C   | `-`               | `-`          |
| f'_D   | `-`               | `-`          |
| **Sum**|                   | **`0`**      |

**Matrix `j` (absolute opinion)**

| User   | `sign(user -> post)` | contribution |
|--------|:--------------------:|:------------:|
| f_A    | `+`                  | `+`          |
| f_B    | `+`                  | `+`          |
| f'_C   | `+`                  | `+`          |
| f'_D   | `+`                  | `+`          |
| **Sum**|                      | **`+4`**     |

**Matrix `k` (absolute connection)**

| User   | reacted? | contribution |
|--------|:--------:|:------------:|
| f_A    | yes      | `1`          |
| f_B    | yes      | `1`          |
| f'_C   | yes      | `1`          |
| f'_D   | yes      | `1`          |
| **Sum**|          | **`4`**      |

#### Post 2 -- signs: f_A `+`, f_B `-`, f'_C `+`, f'_D `-`

**Matrix `h` (relative opinion)**

| User   | `sign(U -> user)` | `sign(user -> post)` | contribution |
|--------|:-----------------:|:--------------------:|:------------:|
| f_A    | `+`               | `+`                  | `+`          |
| f_B    | `+`               | `-`                  | `-`          |
| f'_C   | `-`               | `+`                  | `-`          |
| f'_D   | `-`               | `-`                  | `+`          |
| **Sum**|                   |                      | **`0`**      |

**Matrix `i` (relative connection)**

| User   | `sign(U -> user)` | contribution |
|--------|:-----------------:|:------------:|
| f_A    | `+`               | `+`          |
| f_B    | `+`               | `+`          |
| f'_C   | `-`               | `-`          |
| f'_D   | `-`               | `-`          |
| **Sum**|                   | **`0`**      |

**Matrix `j` (absolute opinion)**

| User   | `sign(user -> post)` | contribution |
|--------|:--------------------:|:------------:|
| f_A    | `+`                  | `+`          |
| f_B    | `-`                  | `-`          |
| f'_C   | `+`                  | `+`          |
| f'_D   | `-`                  | `-`          |
| **Sum**|                      | **`0`**      |

**Matrix `k` (absolute connection)**

| User   | reacted? | contribution |
|--------|:--------:|:------------:|
| f_A    | yes      | `1`          |
| f_B    | yes      | `1`          |
| f'_C   | yes      | `1`          |
| f'_D   | yes      | `1`          |
| **Sum**|          | **`4`**      |

#### Resulting metric vector

| Post   | `R` | `h` | `i` | `j` | `k` |
|--------|:---:|:---:|:---:|:---:|:---:|
| Post 1 | `2` | `0` | `0` | `+4`| `4` |
| Post 2 | `2` | `0` | `0` | `0` | `4` |

Both posts tie on `h` (`0`), on `h + i` (`0`), and diverge on `h + i + j`
(`+4` vs. `0`), so Post 1 ranks above Post 2 in the final order.

---

## 6. Summary

- General rule: any signed graph, any root, any number of intermediate layers,
  any target layer.
- For each target node, compute `R`, `S`, `h`, `i`, `j`, `k`.
- **Sort** by `R -> S -> k -> j -> i -> h`.
- **Order** inside each `R` group by `h`, then `h + i`, then `h + i + j`,
  then `h + i + j + k`.
- `S` is not reused in the ordering phase -- scalar order is already set in
  the sort phase and the tie-breaker chain completes the resolution.
- The `User -> User -> Post` scenario is a specific example of this rule, not
  the rule itself.

---

## 7. Time and recency (OPEN DESIGN QUESTION)

Time decay must exist in some form but is not yet fully designed. Known
constraints:

- Old content can become newly relevant (a friend comments on a post I liked
  years ago — I should see the comment, and the post becomes slightly more
  relevant again).
- New content can be irrelevant (a brand new post from someone 5 hops away
  that no one I know has interacted with).
- **Recency is not importance.** Time is a factor but not a dominant one.

What shape the decay function should take (exponential, linear, step
function) and how it interacts with `R`, `h`, `i`, `j`, `k` is still an
open design choice.

---

## 8. The "already seen" problem (OPEN DESIGN QUESTION)

Users should not be re-shown content they've already seen unless something
meaningful happened (e.g. a friend commented on it). This creates a problem:

**Option A: "View" edges (0, 0 sentiment/relevance edges for any node visited)**
- Pro: Clean graph-native solution. "I've seen this" is just another edge.
- Con: Explodes the edge count. Instead of sorting through 3 posts a friend
  liked, you sort through 10,000 posts they've viewed. Computation cost
  becomes untenable.

**Option B: Separate "seen" store outside the graph**
- Pro: Doesn't pollute the graph. Can use a compact data structure (bloom
  filter, bitset, Redis set).
- Con: Breaks the "everything is in the graph" purity. Adds a third data
  store.

**Option C: Client-side "seen" tracking**
- Pro: Aligns with the decentralized feed calculation vision (the client
  already has a subgraph). The client knows what it's shown the user.
- Con: Doesn't sync across devices without additional infrastructure.

**Option D: View edges with aggressive compaction**
- Pro: Graph-native. Only recent view edges are kept as individual layers;
  older ones are compacted into a summary.
- Con: Compaction logic adds complexity. Defining "recent" is another design
  decision.

This needs a dedicated design session. The solution must:
1. Not flood the graph with low-signal edges.
2. Not be a black box.
3. Allow users to revisit content manually.
4. Surface content again when something meaningful changes (new interactions
   from people the user cares about).
