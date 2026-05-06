# Graph Data Model — Memgraph

This document covers the **Memgraph schema** — the graph-topology layer.

Memgraph carries the **bare minimum** needed for traversal, ranking,
governance, and authorship derivation. Everything else — display content,
counts, per-viewer operational state — lives in Postgres. The leaner the
graph stays, the better it scales. See
[architecture.md §1](architecture.md) for the full split.

For the conceptual model (node categories, edge dimensions, append-only
rule, junction approval pattern), see:

- [graph-model.md](../primitive/graph-model.md) — foundation
- [nodes.md](../primitive/nodes.md) — full node catalog with rationale
- [edges.md](../primitive/edges.md) — full edge catalog with the
  relationship-label scheme
- [layers.md](../primitive/layers.md) — append-only across edges,
  node properties, and Postgres-side display content

For the Postgres side, see [data-model.md](data-model.md).

---

## ID strategy

UUIDs are the shared key between Memgraph and Postgres. Both databases
store the same ID for the same entity; neither stores the other's
fields.

1. UUIDs are generated in the **API layer** (Rust), not by the database.
2. The same UUID is written to both databases in the same request.
3. Memgraph nodes store the UUID as a `String` property named `id`.
4. Most node types use random UUIDs (v4). Hashtags use a content-
   addressed UUID (UUIDv5 of the canonical name with a fixed
   project-scoped namespace) so independent creations of the same
   hashtag converge on one node — see
   [data-model.md "Node identity strategies"](data-model.md) for the
   three identity strategies and their federation properties.

---

## Node labels

Memgraph is schemaless — properties don't need to be declared up front.
The shapes below describe what each label carries and the
constraints/indexes the application relies on.

### Actor nodes

#### `:User`

| Property       | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. Always set by the API. |
| `username`          | String | Handle for mentions/lookups. Layered per [layers.md](../primitive/layers.md). |
| `network_role`      | String | `'member'` or `'moderator'`. Layered. Backs platform-wide governance — see [network.md](../primitive/network.md). |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. Set via Network-level governance — see [moderation.md](../primitive/moderation.md). |

```cypher
CREATE CONSTRAINT ON (u:User) ASSERT u.id IS UNIQUE;
CREATE CONSTRAINT ON (u:User) ASSERT u.username IS UNIQUE;
CREATE INDEX ON :User(id);
```

#### `:Collective`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `name`              | String | Handle, analogous to `User.username`. Layered. |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. See [moderation.md](../primitive/moderation.md). |

```cypher
CREATE CONSTRAINT ON (c:Collective) ASSERT c.id IS UNIQUE;
CREATE CONSTRAINT ON (c:Collective) ASSERT c.name IS UNIQUE;
CREATE INDEX ON :Collective(id);
```

### Content nodes

#### `:Post`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `author_id`         | String | Cached derivation; rebuilt from earliest incoming layer-1 edge. See [authorship.md](../primitive/authorship.md). |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. See [moderation.md](../primitive/moderation.md). |

```cypher
CREATE CONSTRAINT ON (p:Post) ASSERT p.id IS UNIQUE;
CREATE INDEX ON :Post(id);
```

#### `:Comment`

Same shape as `:Post`: `id`, cached `author_id`, and
`moderation_status` (`'normal'` / `'sensitive'` / `'illegal'`,
layered, default `'normal'` — see
[moderation.md](../primitive/moderation.md)).

```cypher
CREATE CONSTRAINT ON (c:Comment) ASSERT c.id IS UNIQUE;
CREATE INDEX ON :Comment(id);
```

#### `:Chat`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `name`              | String | Optional; layered. The graph carries it for routing/display hints. |
| `join_policy`       | String | `'open'` / `'invite-only'` / `'request-entry'` / `'multi-sig'`. Layered. Read by the system when an actor's claim toward a `:ChatMember` arrives, to decide what approval is required. See [chats.md §2](../instances/chats.md). |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. See [moderation.md](../primitive/moderation.md). |

The `content_privacy` setting (plaintext vs E2EE) lives in Postgres,
not on the graph — message bodies are always Postgres-side per
[chats.md §4-5](../instances/chats.md), so the graph never reads it.
See [data-model.md](data-model.md).

```cypher
CREATE CONSTRAINT ON (c:Chat) ASSERT c.id IS UNIQUE;
CREATE INDEX ON :Chat(id);
```

#### `:ChatMessage`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `author_id`         | String | Cached. |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. **Plaintext chats only** — community moderation can't classify content it can't read. See [moderation.md](../primitive/moderation.md). |

```cypher
CREATE CONSTRAINT ON (m:ChatMessage) ASSERT m.id IS UNIQUE;
CREATE INDEX ON :ChatMessage(id);
```

#### `:Item`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. See [moderation.md](../primitive/moderation.md). |

```cypher
CREATE CONSTRAINT ON (i:Item) ASSERT i.id IS UNIQUE;
CREATE INDEX ON :Item(id);
```

#### `:Hashtag`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUIDv5, content-addressed from `name`. See [data-model.md "Node identity strategies"](data-model.md). |
| `name`              | String | Canonical form: lowercase, no `#`. |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. See [moderation.md](../primitive/moderation.md). |

```cypher
CREATE CONSTRAINT ON (h:Hashtag) ASSERT h.id IS UNIQUE;
CREATE CONSTRAINT ON (h:Hashtag) ASSERT h.name IS UNIQUE;
CREATE INDEX ON :Hashtag(id);
```

#### `:Proposal`

| Property          | Type    | Notes |
|---|---|---|
| `id`              | String  | UUID v4. |
| `target_property` | String  | Name of the property on the target node. |
| `proposed_value`  | Variant | The proposed new value (type matches the target property). |

The target node itself is reached via a `:TARGETS` structural edge
(`Proposal → Target`), not a foreign-key property — see
[edges.md §2](../primitive/edges.md).

```cypher
CREATE CONSTRAINT ON (p:Proposal) ASSERT p.id IS UNIQUE;
CREATE INDEX ON :Proposal(id);
```

See [governance.md §2.1](../primitive/governance.md) for the role of
Proposal nodes.

### Junction nodes

#### `:ChatMember`

| Property        | Type   | Notes |
|---|---|---|
| `id`            | String | UUID v4. |
| `role`          | String | `'admin'` / `'mod'` / `'member'`. Layered. |
| `voting_weight` | Float  | Optional; used when the chat sets per-member weight directly rather than deriving it from `role` at tally time. Layered when present. |

```cypher
CREATE CONSTRAINT ON (m:ChatMember) ASSERT m.id IS UNIQUE;
CREATE INDEX ON :ChatMember(id);
```

#### `:CollectiveMember`

| Property        | Type   | Notes |
|---|---|---|
| `id`            | String | UUID v4. |
| `role`          | String | Open-ended per the social contract: `'founder'`, `'shareholder'`, `'worker'`, `'band member'`, `'subsidiary'`, `'partner'`, `'member'`, etc. Layered. |
| `ownership_pct` | Float  | Optional; when the role implies a stake. Layered. |
| `voting_weight` | Float  | Optional override. Layered. |

```cypher
CREATE CONSTRAINT ON (m:CollectiveMember) ASSERT m.id IS UNIQUE;
CREATE INDEX ON :CollectiveMember(id);
```

#### `:ItemOwnership`

| Property | Type   | Notes |
|---|---|---|
| `id`     | String | UUID v4. |

Additional properties pending — committed alongside the
[items.md](../instances/items.md) design pass.

```cypher
CREATE CONSTRAINT ON (o:ItemOwnership) ASSERT o.id IS UNIQUE;
CREATE INDEX ON :ItemOwnership(id);
```

### System nodes

#### `:Network`

A **singleton per instance** carrying Network-level configuration —
moderation thresholds, role-change quorums, eligibility-definition
parameters. Properties are layered per
[layers.md](../primitive/layers.md); each is settable via a Proposal
targeting that property name. See
[network.md](../primitive/network.md).

| Property                          | Type    | Notes |
|---|---|---|
| `id`                              | String  | UUID v4. Always set by the API at instance bootstrap. |
| `mod_role_change_quorum`          | Float   | Minimum fraction of active members that must cast a vote on a `User.network_role` Proposal. Default `0.30`. |
| `mod_role_change_threshold`       | Float   | Fraction of cast votes required in favor for a `User.network_role` Proposal to pass. Default `0.50`. Mod-gate applies (≥1 existing mod positive vote). |
| `moderation_sensitive_quorum`     | Float   | Quorum for `'sensitive'` classification Proposals. Default `0.01` (1%). |
| `moderation_sensitive_threshold`  | Float   | Pass-threshold for `'sensitive'`. Default `0.50`. Mod-gate applies. |
| `moderation_illegal_quorum`       | Float   | Quorum for `'illegal'` classification Proposals. Default `0.02` (2%). |
| `moderation_illegal_threshold`    | Float   | Pass-threshold for `'illegal'`. Default `0.667` (2/3). Mod-gate applies. |
| `active_threshold_days`           | Integer | A User counts as an "active member" if they have at least one outgoing actor edge with timestamp within the last N days. Default `30`. |

```cypher
CREATE CONSTRAINT ON (n:Network) ASSERT n.id IS UNIQUE;
CREATE INDEX ON :Network(id);
```

There is exactly **one** `:Network` node per CoGra instance.
Singleton enforcement is application-level (the bootstrap path
creates it; ordinary code paths never insert a second). The
instance configuration knows the singleton's `id`.

---

## Edge labels

Memgraph relationships carry exactly one label. The catalog and the rules
for picking the right one live in
[edges.md §3](../primitive/edges.md). Per-label assignment:

| Label          | Endpoints                                                                | Source     |
|---|---|---|
| `:ACTOR`       | User \| Collective → any node                                            | Actor sets |
| `:CLAIM`       | Junction → Parent (e.g. `ChatMember → Chat`)                             | System     |
| `:APPROVAL`    | Parent → Junction (e.g. `Chat → ChatMember`)                             | System     |
| `:CONTAINMENT` | Comment → Post / Comment / Chat / ChatMessage / Item; ChatMessage → Chat | System     |
| `:TAGGING`     | Post → Hashtag, Item → Hashtag                                           | System     |
| `:TARGETS`     | Proposal → Target Node                                                   | System     |
| `:REFERENCES`  | ChatMessage → any node                                                   | System     |
| `:STRUCTURAL`  | Any structural edge not in a sub-category above                          | System     |

## Edge properties

Every edge carries the same property shape, regardless of label:

| Property    | Type           | Notes |
|---|---|---|
| `dim1`      | Float          | Range `[-1.0, +1.0]`. Actor edges: signed valence (sentiment / approval / affirmation). Structural edges: typically `0`, except state-bearing pairs (junction approval claim/approval) where `dim1` carries affirmed (`> 0`) / neutral (`0`) / revoked (`< 0`) state. |
| `dim2`      | Float          | Range `[-1.0, +1.0]`. Actor edges: signed connection-weight (interest / relevance / importance). Structural edges: typically `0`. |
| `timestamp` | LocalDateTime  | When this layer was created. |
| `layer`     | Integer        | Layer number (≥ 1). |

See [graph-model.md §4](../primitive/graph-model.md) for the edge
structure and [graph-model.md §6](../primitive/graph-model.md) for the
unified two-axis dimension grammar.

---

## What is intentionally NOT in Memgraph

- **Display content** — bios, profile text, post bodies, comment
  bodies, message bodies, chat descriptions, image and video URLs.
  Lives in Postgres or media servers, linked by UUID. See
  [data-model.md](data-model.md).
- **Materialized aggregations** — counts, sums, or averages over
  edges. Derivable from graph traversal at query time. See
  [architecture.md §3](architecture.md).
- **Per-viewer operational state** — `user_view_log` (seen-list)
  and similar per-viewer filter data. Lives in Postgres, or wherever
  the viewer chooses to store it. See
  [data-model.md](data-model.md).

