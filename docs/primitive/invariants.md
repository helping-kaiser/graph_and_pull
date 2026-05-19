# Invariants

A discoverable index of the load-bearing invariants of the CoGra
protocol. Each entry is one line — a short statement and a link to
the section that owns the rule. **The linked section is canonical**
(every rule below is tagged in its owning doc as `**Invariant:**`);
this file is a pointer, not a restatement, so it cannot drift from
the owning doc.

Grep-able: `grep -ri "Invariant:" docs/` finds every call-out the
entries below link to.

The themes are a curator's grouping, not a part of the protocol.
Same invariant can be load-bearing under more than one theme;
listed under the most useful one.

---

## Topology and visibility

- [Edges are directional](graph-model.md#1-core-principles) —
  `A → B` and `B → A` are independent edges.
- [Edge tensor uniformity](graph-model.md#4-edge-structure) —
  every edge carries 2 dimensions + system dimensions; shape is
  the same across all edge types.
- [At most one structural edge per `(source, target)` pair](edges.md#2-structural-edges)
  — drives the `:TAGGING` / `:REFERENCES` carve-out.
- [Chat topology is always public](../instances/chats.md#1-mental-model-reset)
  — only message **bodies** are private, and only when encrypted.
- [No structural 1:1 chat uniqueness](../instances/chats.md#12-11-vs-group-chats)
  — two users may have multiple parallel 1:1 chats.
- [Inbound edges don't affect the receiver's feed](graph-model.md#7-directionality-inbound-edges-dont-affect-your-graph)
  — anti-bot foundation.
- [Topology is always public](graph-model.md#1-core-principles) —
  privacy of content is achieved via end-to-end encryption, never
  by hiding nodes or edges.
- [Memgraph owns topology, Postgres owns display content](../implementation/architecture.md#1-graph-db-owns-topology-postgres-owns-content)
  — UUIDs are the shared key; no content in Memgraph, no topology
  in Postgres.

## State and lifecycle

- [Graph structure is never deleted](layers.md#5-deletion-policy)
  — no node, edge, or layer is ever removed; absolute.
- [No silent deletion](layers.md#5-deletion-policy) — every
  redaction (graph-side or Postgres-side) leaves a visible mark.
- [Junction state is encoded in topology](graph-model.md#5-junction-node-flows)
  — claim only = pending; claim + approval = active; negative top
  layer on either = inactive. No status flag.
- [Every Collective has or has had ≥1 active member](../instances/collectives.md#9-lifecycle)
  — zero active members ≡ dissolved.
- [ItemOwnership forms an append-only chain](../instances/items.md#7-supersession-exactly-one-active-itemownership-per-item)
  — every past owner remains visible on the graph.
- [At most one active ItemOwnership per Item](../instances/items.md#7-supersession-exactly-one-active-itemownership-per-item)
  — identifying the current owner is a single-edge query.
- [No direct parallel co-ownership of an Item](../instances/items.md#9-shared-ownership-routes-through-a-collective)
  — shared ownership routes through a Collective.

## Authority and gates

- [Out-of-graph authority is confined to instance bootstrap](graph-model.md#1-core-principles)
  — the `:Network` singleton creation and the genesis moderator's
  `network_role` layer are the only two writes that escape the
  actor-gesture-or-governance rule.
- [Mod weight = member weight = 1; mod is a gate, not a weight](../instances/moderation.md#3-the-mod-gate-rule)
  — uniform across content moderation and moderator role changes.
- [Chat-key rotation on membership change is automatic, not voted](../instances/chats.md#9-encryption-as-the-privacy-mechanism)
  — only mid-epoch rotation runs through governance.
- [Chat-internal disavowal routes through a Proposal node](../instances/chats.md#10-moderation)
  — both Level 1 (message) and Level 2 (member) carry the
  `'node'` sentinel; no direct vote edge from a `ChatMember`
  drives the outcome.
- [Collective content-acts default permissive; governance-acts default deny](../instances/collectives.md#2-acting-through-the-collective)
  — asymmetry reflects reversibility.
- [Edges attributed to a Collective carry no per-edge record of the acting member](edges.md#1-actor-edges)
  — accountability lives in the social contract, not in edge
  attribution. Deliberate non-feature.

## Ranking

- [Ranking comes only from the graph](../implementation/architecture.md#3-all-ranking-comes-from-the-graph)
  — no materialized counters, popularity scores, or ML signals;
  ranking is computed at query time from the edge tensor.
- [Kill rule: a `0` in either dim zeros the path product](feed-ranking.md#32-zero-handling--kill-rule)
  — zeros are real multiplicative factors, never skipped; once a
  dim is zeroed on a path it cannot be revived downstream.
- [Hashtags do not participate in path products](../instances/hashtag.md#4-edges)
  — `:TAGGING` is pure topology for discovery, never traversed by
  feed ranking.

---

## How to extend this index

When adding a new invariant:

1. Tag it in the owning doc with a `**Invariant:** ...` line at
   the most contextually relevant spot. The owning doc is
   canonical — write the full statement and the "why" there.
2. Add one line to the matching theme above: a short version of
   the statement and a link with the anchor. **Do not duplicate
   prose** — the index is a pointer.

If an invariant is genuinely cross-cutting (e.g. an edge-tensor
property that lives in `edges.md` but is invoked by every
cluster doc), pick a single canonical home and have other docs
cross-reference it. The index lists each invariant exactly once.
