# Nodes

The catalog of node types in CoGra. Each row gives a one-line
description and links to the dedicated doc where the per-node
mechanics live — creation flow, graph-side and Postgres-side
properties, edges, authorship, and lifecycle.

For the conceptual framing — the four categories (actor, content,
junction, system) and why they matter — see
[graph-model.md §2](graph-model.md#2-node-categories). For the
edges that connect nodes, see [edges.md](edges.md). For concrete
property types, constraints, and indexes, see
[graph-data-model.md](../implementation/graph-data-model.md). For
Postgres-side display-content shapes, see
[data-model.md](../implementation/data-model.md).

The one cross-cutting topic that lives in this doc rather than in
any single per-node doc is the universal `moderation_status`
property — same shape and same mechanism across every node type
that carries user-authored content.

---

## Universal: `moderation_status`

Every node type that carries open user-authored content — avatars,
profile text, post bodies, comment bodies, message bodies, chat
descriptions, item descriptions, hashtag names — carries an
additional `moderation_status` graph property:
`'normal'` / `'sensitive'` / `'illegal'`, default `'normal'`,
layered. The Network-wide governance instance described in
[moderation.md](../instances/moderation.md) is what sets it.

The three values are the platform's content-classification
buckets. Their *meanings* and *behavioural consequences* are
fixed at the primitive level; the *examples of what falls in
each* are platform policy and live in
[platform-guidelines.md §1](../instances/platform-guidelines.md#1-the-three-buckets):

- **`'normal'`.** Default. Carries no special treatment — no
  filter, no redaction. Not an enumerated category; the absence
  of any other classification.
- **`'sensitive'`.** Lawful content that warrants a viewer-side
  filter (graphic, mature, disturbing). The content stays — no
  redaction. Frontends respect each viewing user's
  `content_filtering_severity_level` preference
  ([data-model.md](../implementation/data-model.md)) when
  rendering. Reversible via
  [counter-Proposal](governance.md#counter-proposals) back to
  `'normal'`.
- **`'illegal'`.** Content the Network treats as unlawful or so
  universally prohibited that hosting it is itself a harm.
  Triggers the redaction cascade in
  [layers.md §5](layers.md#5-deletion-policy) and the
  archive-with-legal-hold disposition in
  [retention-archive.md](retention-archive.md). Not reversible,
  because the underlying redaction markers are append-only.

The two non-default values reach the node by different paths:

- `'sensitive'` — set directly by a passing `'sensitive'`
  classification Proposal.
- `'illegal'` — set automatically by the system when any field
  on the node receives a redaction marker per
  [layers.md §5](layers.md#5-deletion-policy). Illegal-content
  classification itself is **per-field**, not per-node — the
  Proposal targets one specific field, or the `'node'` sentinel
  (covering every user-input field plus every attached media on
  the target — see "Whole-node targeting" below), and each
  targeted field's top layer is replaced with a redaction marker.
  The
  auto-flip on `moderation_status` exists so frontends can
  distinguish three filter states: normal content, soft-filterable
  sensitive content, and partially-or-fully-redacted illegal
  content (which a viewing user may want hidden entirely).

`'illegal'` is the strongest state — it isn't downgraded by a
later `'sensitive'` Proposal while redacted fields remain.

The property applies to: **User, Collective, Post, Comment,
ChatMessage, Chat, Item, Hashtag**. Junction nodes (`ChatMember`,
`CollectiveMember`, `ItemOwnership`) have no user-input fields
and so carry no `moderation_status`. **Proposal** is in the same
position — its substance is just `target_property` +
`proposed_value` + the `:TARGETS` edge, with no user-input field
to redact and no Postgres-side display content either. The
**`:Network` singleton** is in the same position for the same
reason: pure configuration state with no user-input fields. See
[network.md §3](network.md#3-graph-side-properties).

**Distinct from chat-internal disavowal.** Moderation status —
the `'normal'` / `'sensitive'` / `'illegal'` value set described
above — is a Network-scope graph property (or per-field redaction
state) maintained by the moderation flow in
[moderation.md](../instances/moderation.md). Chat-internal
disavowal is a **separate value system**: a Chat-scope
Proposal-mediated state with the value set `'normal'` /
`'disavowed'`, set on `:APPROVAL` edges (Level 2) or as the
existence of a passed disavowal Proposal targeting a
ChatMessage (Level 1) via the `'node'` sentinel below. The two
share no values, no scope, and no graph property — see
[moderation.md §"Vocabulary: moderation vs disavowal"](../instances/moderation.md#vocabulary-moderation-vs-disavowal)
for the boundary.

---

## Whole-node targeting: the `'node'` sentinel

A Proposal's `target_property` normally carries the name of one
graph property on the target node — `'name'`, `'role'`,
`'moderation_status'`, `'network_role'`, and so on. The sentinel
value `'node'` reserves `target_property` for a whole-node
operation rather than a single property: the Proposal targets the
node itself, and the cascade interpreter dispatches on the
target's node type instead of writing a layer on a named property.

The sentinel exists because the value space of `target_property`
is the graph-property names on the target node, and there is no
graph-property name that means "the whole node." A reserved value
extends that space without overloading any real property name.

The cascade dispatch — what the interpreter actually writes when
a `'node'` Proposal passes threshold — is specific to the
mechanism that uses the sentinel. The primitive registers the
sentinel and its meaning; the per-cascade behaviour lives with
the instance:

- **Illegal-content classification** — see
  [moderation.md §1](../instances/moderation.md#1-the-two-classification-paths).
  `proposed_value` is `'illegal'`. The cascade interprets the
  sentinel as "every user-input field plus every attached media"
  on the target — see
  [moderation.md §5](../instances/moderation.md#5-scope) for the
  per-node field coverage.
- **Chat-internal disavowal** — see
  [chats.md §10](../instances/chats.md#10-moderation).
  `proposed_value` is `'disavowed'` (or `'normal'` on
  counter-Proposal); dispatch differs for `ChatMessage` and
  `ChatMember` targets.

A future mechanism that needs whole-node operations on a
different node type can register its own cascade against the
same `'node'` sentinel rather than inventing parallel scaffolding.

---

## 1. Actor nodes

Entities that take actions and create edges.

| Node type | Description |
|-----------|-------------|
| **User** | A person on the platform — off-graph credentials authenticate the API requests that originate their edges. See [user.md](user.md). |
| **Collective** | A group acting through a single graph identity (household, band, co-op, studio, partnership, NGO, company). Created by one founding User; every subsequent gesture is initiated by an authorized CollectiveMember per the Collective's social contract. Same outgoing-edge catalog as a User. See [collectives.md](../instances/collectives.md). |

---

## 2. Content nodes

Entities that are acted upon by actors.

| Node type | Description |
|-----------|-------------|
| **Post** | Content (text and/or media) authored by an actor (User or Collective). The primary public-content surface and the canonical [feed-ranking](feed-ranking.md) target. See [post.md](../instances/post.md). |
| **Comment** | A response authored on another content node — Post, Comment (reply), Chat, ChatMessage, or Item. The platform's universal threading primitive. See [comment.md](../instances/comment.md); per-target containment list in [edges.md §2](edges.md#containment--belonging). |
| **Chat** | A conversation container (1:1 or group) — a first-class interactable node visible on the graph, not a private hidden space. Topology (membership, who-talks-to-whom) is public by design; only message bodies are private, and only when encrypted. See [chats.md](../instances/chats.md). |
| **ChatMessage** | A single message within a Chat, itself a first-class node — likeable, commentable, embed-able. Carries a `content_privacy` flag (`plaintext` / `encrypted`) per message; a single chat can mix both freely. See [chats.md](../instances/chats.md). |
| **Item** | A physical or digital good — ownable (via ItemOwnership), transferable, and talked about. See [items.md](../instances/items.md). |
| **Hashtag** | A topic tag whose identity is content-addressed (UUIDv5 of the canonical name), brought into existence implicitly by the first `:TAGGING` edge. Also covers concepts like places (e.g. `#berlin`). The only content node with no authorship — exempt from [authorship.md](authorship.md)'s earliest-incoming-edge rule. See [hashtag.md](../instances/hashtag.md). |
| **Proposal** | The subject carrier for property-level governance votes — targets one graph property on another node via `:TARGETS`. The one content node with no user-input fields: carries no `moderation_status` and has no Postgres-side display content. See [proposal.md](../instances/proposal.md); the primitive itself is in [governance.md §2.1](governance.md#21-subject). |

---

## 3. Junction nodes

Junction nodes represent relationships that have **roles**, need
**approval flows** (multi-sig), and can themselves be interacted
with (liked, voted on, etc.). They eliminate the need for parallel
edges between the same two nodes — see
[graph-model.md §2](graph-model.md#2-node-categories) for the
framing and §5 for the approval flow.

| Node type | Connects | Description |
|-----------|----------|-------------|
| **ChatMember** | Chat ↔ User/Collective | Membership in a Chat with role (admin/mod/member). Entry can require multi-sig approval per the chat's `join_policy`; can itself be voted on (kick, promote). See [chats.md](../instances/chats.md). |
| **CollectiveMember** | Collective ↔ User/Collective | Membership in a Collective with role and role-attached quantities (e.g. `ownership_pct`). Collectives can themselves be CollectiveMembers — nesting is unlimited. See [collectives.md](../instances/collectives.md). |
| **ItemOwnership** | Item ↔ User/Collective | A specific ownership claim. Each transfer creates a new ItemOwnership; together they form the item's append-only ownership history. See [items.md](../instances/items.md). |

---

## 4. System nodes

A small fourth category for **singleton, instance-level
configuration** that doesn't fit actor / content / junction.

| Node type | Description |
|-----------|-------------|
| **Network** | Singleton per instance. Carries Network-level configuration (moderation thresholds, role-change quorums, eligibility definitions). Targeted by Proposals when those parameters are changed. See [network.md](network.md). |

