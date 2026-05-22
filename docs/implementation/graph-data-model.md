# Graph Data Model — Memgraph

This document covers the **Memgraph schema** — the graph-topology layer.

Memgraph carries the **bare minimum** needed for traversal, ranking,
governance, and authorship derivation. Everything else — display content,
counts, per-viewer operational state — lives in Postgres. The leaner the
graph stays, the better it scales. See
[architecture.md §1](architecture.md#1-graph-db-owns-topology-postgres-owns-content) for the full split.

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
   [data-model.md "Node identity strategies"](data-model.md#node-identity-strategies) for the
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
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. `'sensitive'` is set by a passing classification Proposal; `'illegal'` is auto-flipped by the system when any field on the node receives a redaction marker — see [moderation.md](../instances/moderation.md). |

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
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. `'sensitive'` is set by a passing classification Proposal; `'illegal'` is auto-flipped by the system when any field on the node receives a redaction marker — see [moderation.md](../instances/moderation.md). |

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
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. `'sensitive'` is set by a passing classification Proposal; `'illegal'` is auto-flipped by the system when any field on the node receives a redaction marker — see [moderation.md](../instances/moderation.md). |

```cypher
CREATE CONSTRAINT ON (p:Post) ASSERT p.id IS UNIQUE;
CREATE INDEX ON :Post(id);
```

#### `:Comment`

Same shape as `:Post`: `id` and `moderation_status` (`'normal'` /
`'sensitive'` / `'illegal'`, layered, default `'normal'`,
auto-flipped to `'illegal'` on field redaction — see
[moderation.md](../instances/moderation.md)).

```cypher
CREATE CONSTRAINT ON (c:Comment) ASSERT c.id IS UNIQUE;
CREATE INDEX ON :Comment(id);
```

#### `:Chat`

| Property            | Type   | Notes |
|---|---|---|
| `id`                     | String  | UUID v4. |
| `name`                   | String  | Optional; layered. The graph carries it for routing/display hints. |
| `join_policy`            | String  | `'open'` / `'invite-only'` / `'request-entry'`. Layered. Read by the system when an actor's claim toward a `:ChatMember` arrives, to decide what approval is required. Multi-sig is not a fourth value — it is the configuration shape produced when `entry_approval_required_count > 1` under either `'invite-only'` or `'request-entry'`. See [chats.md §11](../instances/chats.md#11-joining-and-leaving-a-chat). |
| `moderation_status`      | String  | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. `'sensitive'` is set by a passing classification Proposal; `'illegal'` is auto-flipped by the system when any field on the node receives a redaction marker — see [moderation.md](../instances/moderation.md). |
| `epoch`                  | Integer | Current chat-key epoch. Default `1`. Advanced by `+1` on every membership transition that takes effect — `:CLAIM` and `:APPROVAL` both present with positive top layers (join), or active `:APPROVAL` flipped to `dim1 < 0` (leave / disavowal cascade) — and on every passing mid-epoch rotation Proposal. Concurrent transitions serialize per Chat. See [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism). |
| `rotate_key_quorum`      | Float   | Quorum for mid-epoch rotation Proposals targeting `epoch`. Default `0.50`. Layered, amendable via Proposal. See [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism). |
| `rotate_key_threshold`   | Float   | Pass-threshold for mid-epoch rotation Proposals. Default `0.667` (2/3). Layered, amendable via Proposal. See [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism). |

The `content_privacy` setting (plaintext vs E2EE) lives in Postgres,
not on the graph — message bodies are always Postgres-side per
[chats.md §8-9](../instances/chats.md#8-chatmessages-as-first-class-content), so the graph never reads it.
See [data-model.md](data-model.md).

```cypher
CREATE CONSTRAINT ON (c:Chat) ASSERT c.id IS UNIQUE;
CREATE INDEX ON :Chat(id);
```

#### `:ChatMessage`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. `'sensitive'` is set by a passing classification Proposal; `'illegal'` is auto-flipped by the system when any field on the node receives a redaction marker — see [moderation.md](../instances/moderation.md). The protocol does not gate classification on disclosure of the chat key; "moderate only after reading" is a normative requirement on moderators, not a protocol invariant — see [moderation.md §5](../instances/moderation.md#5-scope). |

```cypher
CREATE CONSTRAINT ON (m:ChatMessage) ASSERT m.id IS UNIQUE;
CREATE INDEX ON :ChatMessage(id);
```

The `epoch` index a ciphertext was encrypted under lives in
Postgres alongside the body row, not on the graph — message bodies
are always Postgres-side per [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism),
so the graph never reads it. See [data-model.md](data-model.md).

#### `:Item`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. `'sensitive'` is set by a passing classification Proposal; `'illegal'` is auto-flipped by the system when any field on the node receives a redaction marker — see [moderation.md](../instances/moderation.md). |

```cypher
CREATE CONSTRAINT ON (i:Item) ASSERT i.id IS UNIQUE;
CREATE INDEX ON :Item(id);
```

#### `:Hashtag`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUIDv5, content-addressed from `name`. See [data-model.md "Node identity strategies"](data-model.md#node-identity-strategies). |
| `name`              | String | Canonical form: lowercase, no `#`. |
| `moderation_status` | String | `'normal'` / `'sensitive'` / `'illegal'`. Layered. Default `'normal'`. `'sensitive'` is set by a passing classification Proposal; `'illegal'` is auto-flipped by the system when any field on the node receives a redaction marker — see [moderation.md](../instances/moderation.md). |

```cypher
CREATE CONSTRAINT ON (h:Hashtag) ASSERT h.id IS UNIQUE;
CREATE CONSTRAINT ON (h:Hashtag) ASSERT h.name IS UNIQUE;
CREATE INDEX ON :Hashtag(id);
```

#### `:Proposal`

| Property          | Type    | Notes |
|---|---|---|
| `id`              | String  | UUID v4. |
| `target_property` | String  | Name of the property on the target node, or one of the reserved values: the moderation directive `'full'` (every user-input field on the node, plus all attachments) — see [moderation.md §5](../instances/moderation.md#5-scope); or the whole-node sentinel `'node'` — see [nodes.md "Whole-node targeting"](../primitive/nodes.md#whole-node-targeting-the-node-sentinel). |
| `proposed_value`  | Variant | The proposed new value (type matches `target_property`). Common patterns: (a) **Moderation classification.** `'sensitive'` Proposals set `target_property = 'moderation_status'` and `proposed_value ∈ {'sensitive', 'normal'}`; `'illegal'` Proposals set `target_property` to a specific user-input field name (or `'full'`) and `proposed_value = 'illegal'`. See [moderation.md §1](../instances/moderation.md#1-the-two-classification-paths). The cascade interprets `'illegal'` to write redaction markers and archive originals. (b) **Chat-internal disavowal.** `target_property = 'node'`, `proposed_value ∈ {'disavowed', 'normal'}`. The cascade writes a `dim1 < 0` (or `dim1 > 0` on reversal) layer on the relevant `:APPROVAL` edge per [chats.md §10](../instances/chats.md#10-moderation). (c) **Property amendments.** `proposed_value` is the new value of whatever graph property `target_property` names — a role string, a numeric threshold, etc. |

The target node itself is reached via a `:TARGETS` structural edge
(`Proposal → Target`), not a foreign-key property — see
[edges.md §2](../primitive/edges.md#2-structural-edges).

`:Proposal` intentionally has no `moderation_status` property:
Proposals carry no user-authored content fields, so they fall
outside the universal moderation property per
[nodes.md §"Universal: moderation_status"](../primitive/nodes.md#universal-moderation_status).

```cypher
CREATE CONSTRAINT ON (p:Proposal) ASSERT p.id IS UNIQUE;
CREATE INDEX ON :Proposal(id);
```

See [governance.md §2.1](../primitive/governance.md#21-subject) for the role of
Proposal nodes.

### Junction nodes

All three junction types bind to their bearing actor via a
`:BEARER` structural edge — `Junction → User|Collective` — set
by the API at junction creation, never re-pointed. The Shape A
self-claim that activates the junction must come from the actor
this edge points at; mismatched claims are rejected. See
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows)
and edge-labels table below.

#### `:ChatMember`

| Property        | Type   | Notes |
|---|---|---|
| `id`            | String | UUID v4. |
| `role`          | String | `'admin'` / `'chat_mod'` / `'member'`. Layered. Distinct from the Network-scope `User.network_role = 'moderator'`. |
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

No additional properties — transfer state lives entirely in the
surrounding edges (claim, approval, and supersession layers per
[items.md](../instances/items.md)).

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
| `singleton_marker`                | String  | Always `'singleton'`. Combined with the existence + uniqueness constraints below, prevents a second `:Network` node from ever being inserted. Set at bootstrap; never changes. |
| `mod_role_change_quorum_fraction`     | Float   | Fractional bar for `User.network_role` Proposals: `positive_count ≥ P × \|active members\|`. Default `0.50`. Mod-gate applies (≥1 existing mod positive vote). |
| `mod_role_change_quorum_count`        | Integer | Absolute bar for `User.network_role` Proposals: `positive_count ≥ K`. Default `5000`. The operative bar at tally time is `min(P × \|active\|, K)`. See [governance.md §3 "Petition-style tally and dual quorum"](../primitive/governance.md#petition-style-tally-and-dual-quorum-network-scope-only). |
| `moderation_sensitive_quorum_fraction` | Float   | Fractional bar for `'sensitive'` classification Proposals. Default `0.25`. Mod-gate applies. |
| `moderation_sensitive_quorum_count`   | Integer | Absolute bar for `'sensitive'`. Default `5000`. |
| `moderation_illegal_quorum_fraction`  | Float   | Fractional bar for `'illegal'` classification Proposals. Default `0.50`. Mod-gate applies. |
| `moderation_illegal_quorum_count`     | Integer | Absolute bar for `'illegal'`. Default `10000`. |
| `guidelines_version`              | Integer | Monotonic version of the [platform guidelines](../instances/platform-guidelines.md). Bumped by 1 on each amendment Proposal. Default `1` at bootstrap. |
| `guidelines_hash`                 | String  | SHA-256 hex digest of the canonical guidelines document at the current version. 64 lowercase hex chars. Set at bootstrap to the digest of the version-1 doc; updated together with `guidelines_version` on each amendment. |
| `guidelines_change_quorum_fraction`   | Float   | Fractional bar for guidelines amendment Proposals. Default `0.50`. Mod-gate applies. |
| `guidelines_change_quorum_count`      | Integer | Absolute bar for guidelines amendments. Default `10000`. |
| `property_change_quorum_fraction`     | Float   | Fractional bar for amending baseline-bucket `:Network` properties (`moderation_sensitive_*`, `active_threshold_days`, the baseline pair itself). Default `0.25`. Mod-gate applies. See [network.md §11](../primitive/network.md#11-amending-network-parameters). |
| `property_change_quorum_count`        | Integer | Absolute bar for the same. Default `5000`. |
| `critical_property_change_quorum_fraction` | Float | Fractional bar for amending critical-bucket `:Network` properties (`mod_role_change_*`, `moderation_illegal_*`, `guidelines_change_*`, the critical pair itself). Default `0.50`. Mod-gate applies. See [network.md §11](../primitive/network.md#11-amending-network-parameters). |
| `critical_property_change_quorum_count` | Integer | Absolute bar for the same. Default `10000`. |
| `active_threshold_days`           | Integer | A User counts as an "active member" if they have at least one outgoing actor edge with timestamp within the last N days. Default `30`. |

```cypher
CREATE CONSTRAINT ON (n:Network) ASSERT n.id IS UNIQUE;
CREATE CONSTRAINT ON (n:Network) ASSERT EXISTS (n.singleton_marker);
CREATE CONSTRAINT ON (n:Network) ASSERT n.singleton_marker IS UNIQUE;
CREATE INDEX ON :Network(id);
```

There is exactly **one** `:Network` node per CoGra instance.
Singleton enforcement combines two mechanisms:

- **Graph-side constraint.** The `singleton_marker` property
  carries a fixed value (`'singleton'`); the existence +
  uniqueness constraints together refuse any second insert. A
  second `:Network` either omits the property (fails the
  existence constraint) or carries the only legal value (fails
  the uniqueness constraint).
- **Application discipline.** The bootstrap migration
  ([network.md §2](../primitive/network.md#2-creation)) is the
  only writer; ordinary code paths never attempt a second
  `:Network`.

Belt-and-suspenders: discipline keeps the wrong code from
running; the constraint catches it if discipline fails. The
instance configuration knows the singleton's `id`.

---

## Edge labels

Memgraph relationships carry exactly one label. The catalog and the rules
for picking the right one live in
[edges.md §3](../primitive/edges.md#3-edge-labels-at-the-graph-layer). Per-label assignment:

| Label          | Endpoints                                                                | Source     |
|---|---|---|
| `:ACTOR`       | User \| Collective → any node                                            | Actor sets |
| `:CLAIM`       | Junction → Parent (e.g. `ChatMember → Chat`)                             | System     |
| `:APPROVAL`    | Parent → Junction (e.g. `Chat → ChatMember`)                             | System     |
| `:BEARER`      | Junction → User \| Collective (e.g. `ChatMember → User`)                 | System     |
| `:CONTAINMENT` | Comment → Post / Comment / Chat / ChatMessage / Item; ChatMessage → Chat | System     |
| `:TAGGING`     | Post → Hashtag, Comment → Hashtag, Item → Hashtag                        | System     |
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

See [graph-model.md §4](../primitive/graph-model.md#4-edge-structure) for the edge
structure and [graph-model.md §6](../primitive/graph-model.md#6-dimension-semantics) for the
unified two-axis dimension grammar.

---

## What is intentionally NOT in Memgraph

- **Display content** — bios, profile text, post bodies, comment
  bodies, message bodies, chat descriptions, image and video URLs.
  Lives in Postgres or media servers, linked by UUID. See
  [data-model.md](data-model.md).
- **Materialized aggregations** — counts, sums, or averages over
  edges. Derivable from graph traversal at query time. See
  [architecture.md §3](architecture.md#3-all-ranking-comes-from-the-graph).
- **Per-viewer operational state** — `user_view_log` (seen-list)
  and similar per-viewer filter data. Lives in Postgres, or wherever
  the viewing user chooses to store it. See
  [data-model.md](data-model.md).

