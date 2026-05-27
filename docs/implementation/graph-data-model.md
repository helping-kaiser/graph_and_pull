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

UUIDs are the shared key between Memgraph and Postgres (see
[architecture.md §2](architecture.md#2-uuids-as-the-shared-key)).
Memgraph nodes store the UUID as a `String` property named `id`. Most
node types use random UUIDs (v4); Hashtags use a content-addressed UUIDv5
so independent creations of the same hashtag converge on one node — see
[data-model.md "Node identity strategies"](data-model.md#node-identity-strategies).

---

## Node labels

Memgraph allows ad-hoc properties, but the protocol leans on declarative
constraints (uniqueness, existence) wherever the rule admits one. The shapes
below describe what each label carries and the constraints/indexes the
application relies on; rules the storage layer can't directly express
(e.g. forbidding a property by absence) are stated as ethos invariants and
enforced in code tests.

### Shared shape: per-field moderation-status properties

Every content-bearing node carries one graph property per user-filled
field, holding that field's current moderation status. The shape is
identical wherever it appears: `String`, layered, default `'normal'`,
values `'normal'` / `'sensitive'` / redaction marker. `'sensitive'` is
set by a passing classification Proposal on the per-field property;
the redaction marker is written by the `'illegal'` cascade.

**Naming.** For fields with no graph-side data sibling (`bio`,
`avatar`, `content`, `description`, …), the moderation-status property
is named after the field — `User.bio`, `Post.content`, etc. — and the
property's value IS the status. For fields that already have a
graph-side data property for uniqueness or content-addressing
(`User.username`, `Collective.name`, `Chat.name`, `Hashtag.name`),
a companion `<field>_status` property carries the status; the data
property keeps its name and its current shape, and the cascade writes
to both on illegal classification.

**Node-level cache.** Every content-bearing node also carries
a `moderation_status` property — `String`, **not layered**,
default `'normal'`, values `'normal'` / `'sensitive'` /
`'illegal'` — caching the max severity across that node's
per-field statuses. The cascade writes both the targeted per-field
property and the cache atomically per
[nodes.md "Node-level cache"](../primitive/nodes.md#node-level-cache-moderation_status).
The cache is what feed-ranking and filter queries read.

See [nodes.md "Universal: per-field moderation status"](../primitive/nodes.md#universal-per-field-moderation-status)
and [moderation.md](../instances/moderation.md). Each node-label
table below lists the per-field properties and the cache with a
short "per intro" note rather than repeating the shape.

### Actor nodes

#### `:User`

| Property       | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. Always set by the API. |
| `username`          | String | Handle for mentions/lookups. Layered per [layers.md](../primitive/layers.md). Data; per-field status carried separately by `username_status`. |
| `network_role`      | String | `'member'` or `'moderator'`. Layered. Backs platform-wide governance — see [network.md](../primitive/network.md). |
| `username_status`   | String | Per intro (status for `username`). |
| `display_name`      | String | Per intro. Content lives in Postgres. |
| `bio`               | String | Per intro. Content lives in Postgres. |
| `avatar`            | String | Per intro. Asset lives in object storage; the per-field property exists for moderation targeting. |
| `website_url`       | String | Per intro. Content lives in Postgres. |
| `moderation_status` | String | Node-level cache (per intro). |

```cypher
CREATE CONSTRAINT ON (u:User) ASSERT u.id IS UNIQUE;
CREATE CONSTRAINT ON (u:User) ASSERT u.username IS UNIQUE;
CREATE INDEX ON :User(id);
```

The `username` UNIQUE constraint applies to the property's
current value — the top layer. When an account-deletion or
illegal-content cascade redacts a User's `username`, the new top
layer takes the form `redacted-user-{user_id_uuid}`, mirroring
the Postgres tombstone (see
[account-deletion.md "Username post-redaction"](../instances/account-deletion.md#username-post-redaction)).
The embedded UUID suffix makes the sentinel unique per User by
construction, so the UNIQUE constraint holds across any number
of redactions without requiring layer-aware constraint logic.

#### `:Collective`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUID v4. |
| `name`              | String | Handle, analogous to `User.username`. Layered. Data; per-field status carried separately by `name_status`. |
| `name_status`       | String | Per intro (status for `name`). |
| `display_name`      | String | Per intro. Content lives in Postgres. |
| `description`       | String | Per intro. Content lives in Postgres. |
| `avatar`            | String | Per intro. Asset lives in object storage. |
| `website_url`       | String | Per intro. Content lives in Postgres. |
| `moderation_status` | String | Node-level cache (per intro). |

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
| `content`           | String | Per intro. Body lives in Postgres. |
| `attachments`       | String | Per intro. Covers every attached media on the post as a single status (per-attachment targeting is a future refinement — see [moderation.md §5](../instances/moderation.md#5-scope)); assets live in object storage. |
| `moderation_status` | String | Node-level cache (per intro). |

```cypher
CREATE CONSTRAINT ON (p:Post) ASSERT p.id IS UNIQUE;
CREATE INDEX ON :Post(id);
```

#### `:Comment`

Same shape as `:Post`: `id`, `content` (per intro), `attachments`
(per intro), `moderation_status` (per intro).

```cypher
CREATE CONSTRAINT ON (c:Comment) ASSERT c.id IS UNIQUE;
CREATE INDEX ON :Comment(id);
```

#### `:Chat`

| Property            | Type   | Notes |
|---|---|---|
| `id`                     | String  | UUID v4. |
| `name`                   | String  | Optional; layered. The graph carries it for routing/display hints. Data; per-field status carried separately by `name_status`. |
| `name_status`            | String  | Per intro (status for `name`). |
| `description`            | String  | Per intro. Content lives in Postgres. |
| `image`                  | String  | Per intro. Asset lives in object storage. |
| `join_policy`            | String  | `'open'` / `'invite-only'` / `'request-entry'`. Layered. Read by the system when an actor's claim toward a `:ChatMember` arrives, to decide what approval is required. Multi-sig is not a fourth value — it is the configuration shape produced when `entry_approval_required_count > 1` under either `'invite-only'` or `'request-entry'`. See [chats.md §11](../instances/chats.md#11-joining-and-leaving-a-chat). |
| `invite_proposer_roles`  | String[] | `ChatMember.role` values whose bearers may propose a new ChatMember under `'invite-only'`. Default `['chat_mod','admin']`. Inapplicable to `'open'` and `'request-entry'`. Layered. See [chats.md §3.1, §11](../instances/chats.md#31-chat). |
| `entry_approval_required_count` | Integer | Number of qualifying Shape B approver votes the new ChatMember's junction must collect before activation. `0` under `'open'`; default `1` otherwise; higher values produce the multi-sig configuration shape. Layered. See [chats.md §11](../instances/chats.md#11-joining-and-leaving-a-chat). |
| `entry_approval_eligible_roles` | String[] | `ChatMember.role` values whose bearers' Shape B votes count toward `entry_approval_required_count`. Default `['chat_mod','admin']`. Inapplicable to `'open'`. Layered. See [chats.md §3.1, §11](../instances/chats.md#31-chat). |
| `epoch`                  | Integer | Current chat-key epoch. Default `1`. Advanced by `+1` on every membership transition that takes effect — `:CLAIM` and `:APPROVAL` both present with positive top layers (join), or active `:APPROVAL` flipped to `dim1 < 0` (leave / disavowal cascade) — and on every passing mid-epoch rotation Proposal. Concurrent transitions serialize per Chat. See [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism). |
| `rotate_key_quorum`      | Float   | Quorum for mid-epoch rotation Proposals targeting `epoch`. Default `0.50`. Layered, amendable via Proposal. See [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism). |
| `rotate_key_threshold`   | Float   | Pass-threshold for mid-epoch rotation Proposals. Default `0.667` (2/3). Layered, amendable via Proposal. See [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism). |
| `weight_admin`           | Integer | Default voting weight for `ChatMember.role = 'admin'`. Default `5`. Layered. Overridden per-bearer by a non-null `ChatMember.voting_weight`. See [chats.md §10 "How roles fit in"](../instances/chats.md#how-roles-fit-in). |
| `weight_chat_mod`        | Integer | Default voting weight for `ChatMember.role = 'chat_mod'`. Default `3`. Layered. |
| `weight_member`          | Integer | Default voting weight for `ChatMember.role = 'member'`. Default `1`. Layered. |
| `disavowal_l1_quorum`    | Float   | Quorum for Level 1 (ChatMessage) disavowal Proposals. Default `0.20`. Layered. See [chats.md §10](../instances/chats.md#10-moderation). |
| `disavowal_l1_threshold` | Float   | Pass-threshold for Level 1 disavowal. Default `0.50`. Layered. |
| `disavowal_l2_quorum`    | Float   | Quorum for Level 2 (ChatMember) disavowal Proposals. Default `0.40`. Layered. |
| `disavowal_l2_threshold` | Float   | Pass-threshold for Level 2 disavowal. Default `0.667` (2/3). Layered. |
| `role_change_quorum`     | Float   | Quorum for Proposals targeting `ChatMember.role`. Default `0.30`. Layered. The subject member is excluded from eligibility — see [chats.md §10 "Property and role changes via Proposals"](../instances/chats.md#property-and-role-changes-via-proposals). |
| `role_change_threshold`  | Float   | Pass-threshold for role changes. Default `0.50`. Layered. |
| `name_change_quorum`     | Float   | Quorum for Proposals targeting `Chat.name`. Default `0.10`. Layered. |
| `name_change_threshold`  | Float   | Pass-threshold for name changes. Default `0.50`. Layered. |
| `join_policy_change_quorum` | Float | Quorum for Proposals targeting `join_policy`, `invite_proposer_roles`, `entry_approval_required_count`, or `entry_approval_eligible_roles`. Default `0.30`. Layered. |
| `join_policy_change_threshold` | Float | Pass-threshold for the join-policy family. Default `0.667` (2/3). Layered. |
| `governance_amendment_quorum`  | Float | Quorum for Proposals targeting any of the governance fraction or weight properties listed above (the "governance of governance" case). Default `0.30`. Layered. |
| `governance_amendment_threshold` | Float | Pass-threshold for governance amendments. Default `0.667` (2/3). Layered. |
| `moderation_status` | String | Node-level cache (per intro) for the Chat's per-field statuses (`name_status`, `description`, `image`). |

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
| `content`           | String | Per intro. Body lives in Postgres (plaintext or ciphertext per [chats.md §9](../instances/chats.md#9-encryption-as-the-privacy-mechanism)). The protocol does not gate classification on disclosure of the chat key; "moderate only after reading" is a normative requirement on moderators, not a protocol invariant — see [moderation.md §5](../instances/moderation.md#5-scope). |
| `attachments`       | String | Per intro. Assets live in object storage. |
| `moderation_status` | String | Node-level cache (per intro). |

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
| `name`              | String | Per intro. Content lives in Postgres. (Items do not need graph-side uniqueness on `name`; the moderation-status property uses the field name directly.) |
| `description`       | String | Per intro. Content lives in Postgres. |
| `attachments`       | String | Per intro. Assets live in object storage. |
| `moderation_status` | String | Node-level cache (per intro). |

```cypher
CREATE CONSTRAINT ON (i:Item) ASSERT i.id IS UNIQUE;
CREATE INDEX ON :Item(id);
```

#### `:Hashtag`

| Property            | Type   | Notes |
|---|---|---|
| `id`                | String | UUIDv5, content-addressed from `name`. See [data-model.md "Node identity strategies"](data-model.md#node-identity-strategies). |
| `name`              | String | Canonical form: lowercase, no `#`. Immutable except via the `'illegal'` redaction cascade — see [hashtag.md §5](../instances/hashtag.md#5-lifecycle). Data; per-field status carried separately by `name_status`. |
| `name_status`       | String | Per intro (status for `name`; the only user-input field on a Hashtag). |
| `moderation_status` | String | Node-level cache (per intro). |

```cypher
CREATE CONSTRAINT ON (h:Hashtag) ASSERT h.id IS UNIQUE;
CREATE CONSTRAINT ON (h:Hashtag) ASSERT h.name IS UNIQUE;
CREATE INDEX ON :Hashtag(id);
```

#### `:Proposal`

| Property          | Type    | Notes |
|---|---|---|
| `id`              | String  | UUID v4. |
| `target_property` | String  | Name of the property on the target node, or the reserved whole-node sentinel `'node'` — see [nodes.md "Whole-node targeting"](../primitive/nodes.md#whole-node-targeting-the-node-sentinel). The `'node'` sentinel covers both the moderation cascade (every user-input field plus all attachments — see [moderation.md §5](../instances/moderation.md#5-scope)) and chat-internal disavowal — see [chats.md §10](../instances/chats.md#10-moderation). |
| `proposed_value`  | Variant | The proposed new value (type matches `target_property`). Common patterns: (a) **Moderation classification.** `target_property` names the per-field moderation-status property on the target node (e.g., `'bio'`, `'content'`, `'username_status'`) or the `'node'` sentinel for whole-node coverage; `proposed_value ∈ {'sensitive', 'illegal', 'normal'}`. On pass, the cascade writes the new value as a layer (for `'sensitive'` / `'normal'`) or replaces the top layer with a redaction marker (for `'illegal'`, plus archive + Postgres tombstone). See [moderation.md §1](../instances/moderation.md#1-the-two-classification-paths). (b) **Chat-internal disavowal.** `target_property = 'node'`, `proposed_value ∈ {'disavowed', 'normal'}`. The cascade writes a `dim1 < 0` (or `dim1 > 0` on reversal) layer on the relevant `:APPROVAL` edge per [chats.md §10](../instances/chats.md#10-moderation). (c) **Property amendments.** `proposed_value` is the new value of whatever graph property `target_property` names — a role string, a numeric threshold, etc. |

The target node itself is reached via a `:TARGETS` structural edge
(`Proposal → Target`), not a foreign-key property — see
[edges.md §2](../primitive/edges.md#2-structural-edges).

`:Proposal` intentionally carries no per-field moderation
properties and no `moderation_status` cache: Proposals have no
user-authored content fields, so they fall outside the
moderation scope per
[nodes.md §"Universal: per-field moderation status"](../primitive/nodes.md#universal-per-field-moderation-status).

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
| `voting_weight` | Float  | Nullable per-bearer override of the role-derived weight (per-chat defaults `weight_admin` / `weight_chat_mod` / `weight_member` on the Chat node, §`:Chat` above). When non-null, the tally reads this value directly and the role-derived default is ignored; when null (default), the role-derived rule applies. Layered. See [governance.md §2.3](../primitive/governance.md#23-weight-function). |

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

#### Junction state lives in topology, not in a property

None of the three junction tables declares a `status` property — by design.
Junction state (pending / active / revoked) is derived from the two-edge
approval pair's top-layer `dim1` values per
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows). A
stored flag would be a second source of truth that could drift.

Memgraph can't directly forbid a property by absence, so enforcement is
ethos + test: the schema above is the canonical declaration of what
junction labels carry, an integration test asserts no junction ever
materializes with a `status` property, and service-layer write paths never
write one.

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
| `property_change_quorum_fraction`     | Float   | Fractional bar for amending baseline-bucket `:Network` properties (`moderation_sensitive_*`, `active_threshold_days`, `time_decay_half_life_days`, the baseline pair itself). Default `0.25`. Mod-gate applies. See [network.md §11](../primitive/network.md#11-amending-network-parameters). |
| `property_change_quorum_count`        | Integer | Absolute bar for the same. Default `5000`. |
| `critical_property_change_quorum_fraction` | Float | Fractional bar for amending critical-bucket `:Network` properties (`mod_role_change_*`, `moderation_illegal_*`, `guidelines_change_*`, the critical pair itself). Default `0.50`. Mod-gate applies. See [network.md §11](../primitive/network.md#11-amending-network-parameters). |
| `critical_property_change_quorum_count` | Integer | Absolute bar for the same. Default `10000`. |
| `active_threshold_days`           | Integer | A User counts as an "active member" if they have at least one outgoing actor edge with timestamp within the last N days. Default `30`. |
| `time_decay_half_life_days`       | Integer | Half-life of the reactor-edge time-decay factor `f(Δt)` used by feed-ranking. Default `30`. Baseline-bucket amendable. See [feed-ranking.md §7.3](../primitive/feed-ranking.md#73-shape--exponential-30-day-half-life-frontend-tunable). |

```cypher
CREATE CONSTRAINT ON (n:Network) ASSERT n.id IS UNIQUE;
CREATE CONSTRAINT ON (n:Network) ASSERT EXISTS (n.singleton_marker);
CREATE CONSTRAINT ON (n:Network) ASSERT n.singleton_marker IS UNIQUE;
CREATE INDEX ON :Network(id);
```

There is exactly **one** `:Network` node per CoGra instance.
Singleton enforcement combines two mechanisms:

- **Graph-side constraint.** `singleton_marker` carries a fixed value
  (`'singleton'`); the existence + uniqueness constraints together refuse
  any second insert. A second `:Network` either omits the property (fails
  existence) or carries the only legal value (fails uniqueness).
- **Application discipline.** The bootstrap migration
  ([network.md §2](../primitive/network.md#2-creation)) is the only writer;
  ordinary code paths never attempt a second `:Network`.

The instance configuration knows the singleton's `id`.

---

## Edge labels

Memgraph relationships carry exactly one label. The catalog and the rules
for picking the right one live in
[edges.md §3](../primitive/edges.md#3-edge-labels-at-the-graph-layer). Per-label assignment:

| Label          | Endpoints                                                                | Source     |
|---|---|---|
| `:ACTOR`       | User \| Collective → any node                                            | Actor sets |
| `:AUTHOR`      | User \| Collective → Post \| Comment \| Chat \| ChatMessage \| Item \| Proposal | Actor sets |
| `:CLAIM`       | Junction → Parent (e.g. `ChatMember → Chat`)                             | System     |
| `:APPROVAL`    | Parent → Junction (e.g. `Chat → ChatMember`)                             | System     |
| `:BEARER`      | Junction → User \| Collective (e.g. `ChatMember → User`)                 | System     |
| `:CONTAINMENT` | Comment → Post / Comment / Chat / ChatMessage / Item; ChatMessage → Chat | System     |
| `:TAGGING`     | Post → Hashtag, Comment → Hashtag, Item → Hashtag                        | System     |
| `:TARGETS`     | Proposal → Target Node                                                   | System     |
| `:REFERENCES`  | ChatMessage → any node; Post → any node (except Hashtag); Comment → any node (except Hashtag) | System     |
| `:STRUCTURAL`  | Any structural edge not in a sub-category above                          | System     |

### Single-edge-label enforcement

A `(source, target)` pair carries **at most one edge label** —
actor or structural. Layers within that single label are how the
pair accumulates history; a second label between the same
endpoints is forbidden. See
[edges.md §2](../primitive/edges.md#2-structural-edges) for the
invariant, the rationale, and the cases this rule rules out
(notably the `Post → Hashtag` `:TAGGING` / `:REFERENCES` carve-out
and the parent-Collective `:APPROVAL` / `:ACTOR` collision).

Two enforcement layers:

1. **Service-layer transaction (primary).** Before any edge
   insert, the service layer reads existing edges between the
   same endpoints and rejects the write if any existing edge
   carries a different label. Returns a meaningful error to the
   caller. Same-label layerings (the normal append-only path)
   pass through.
2. **Memgraph trigger (backstop).** Detects writes that bypass
   the service layer. Indicative shape — the exact abort
   primitive depends on the Memgraph version and the
   query-modules library available:

   ```cypher
   CREATE TRIGGER unique_edge_label_per_pair
   ON --> CREATE
   BEFORE COMMIT
   EXECUTE
     UNWIND createdEdges AS new_e
     MATCH (startNode(new_e))-[other]->(endNode(new_e))
     WHERE id(other) <> id(new_e)
       AND type(other) <> type(new_e)
     // Abort the transaction. The exact call depends on the
     // procedure library — e.g. a custom mgps.assert(false, msg)
     // or a write that fails (RAISE-equivalent) — but the
     // matching condition above is the invariant's contract.
     CALL custom.abort_transaction(
       'multiple labels between same (source, target) pair forbidden')
     YIELD * RETURN *;
   ```

   Same-label layers do not match the `type(other) <> type(new_e)`
   filter and pass through unchanged.

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

### Tensor uniformity enforcement

The [edge-tensor-uniformity invariant](../primitive/invariants.md#topology-and-visibility)
— every edge carries `(dim1, dim2, timestamp, layer)` regardless of
label — is enforced at the storage layer via per-label EXISTS
constraints. Shown explicitly for `:ACTOR`; an identical block of
four constraints applies to each remaining label in the table
above (`:AUTHOR`, `:CLAIM`, `:APPROVAL`, `:BEARER`, `:CONTAINMENT`,
`:TAGGING`, `:TARGETS`, `:REFERENCES`, `:STRUCTURAL`):

```cypher
CREATE CONSTRAINT ON ()-[r:ACTOR]-() ASSERT EXISTS (r.dim1);
CREATE CONSTRAINT ON ()-[r:ACTOR]-() ASSERT EXISTS (r.dim2);
CREATE CONSTRAINT ON ()-[r:ACTOR]-() ASSERT EXISTS (r.timestamp);
CREATE CONSTRAINT ON ()-[r:ACTOR]-() ASSERT EXISTS (r.layer);
```

Range checks on `dim1` and `dim2` (`[-1.0, +1.0]`) are not
expressible as a single existence constraint; the service layer
clamps on write and a test suite asserts the invariant
end-to-end. Memgraph's type-constraint family (where available
in the deployed version) takes care of `dim1` / `dim2` being
floats and `timestamp` / `layer` being the expected types.

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

