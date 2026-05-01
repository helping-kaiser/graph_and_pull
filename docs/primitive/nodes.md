# Nodes

The full catalog of node types in CoGra, plus what each type carries
on the graph and where its display content lives.

For the conceptual model — the three categories (actor, content,
junction) and why they matter — see
[graph-model.md §2](graph-model.md). For the edges that connect
nodes, see [edges.md](edges.md).

---

## What lives on the graph

Every node has:

- An identifier (UUID).
- Layer metadata on any authored properties it carries — timestamps
  and layer numbers come from the append-only layer system, not from
  per-node fields. See [layers.md](layers.md).
- Its edges (separate records, not properties — see
  [edges.md](edges.md)).

What the graph does **not** store as node properties:

- **Display content** — bios, post bodies, profile text, images,
  videos, chat descriptions — lives in Postgres or media servers
  and is linked by UUID. See [data-model.md](../implementation/data-model.md).
- **Derived values** — authorship, member counts, current-owner
  pointers — are rebuilt from the graph, not stored as independent
  primitives. Caches of these values may exist on nodes for query
  efficiency; they don't layer and they don't count as authored
  properties. See [layers.md §3](layers.md).

Authored graph-side properties are listed per-type below only where
the graph genuinely needs them. Most content nodes have minimal
graph-side properties because their substance is display content.

---

## 1. Actor nodes

Entities that take actions and create edges.

| Node type | Description |
|-----------|-------------|
| **User** | A person on the platform. |
| **Collective** | Any group of people that needs a single graph identity to act through — a household, band, co-op, studio, partnership, NGO, company. Fully equivalent to Users as actors: can do everything Users can (author content, be followed, post items, create edges toward other nodes, be members of other collectives). See [collectives.md](../instances/collectives.md). |

### Graph-side properties

- **User**: `username` (the handle used for mentions and lookups).
- **Collective**: `name` (the handle used for mentions and lookups,
  analogous to `username` on User).

Additional authored properties for either type (display name,
verified flag, etc.) can be added later — each would layer
independently under the append-only rule.

### Postgres-side content

Profile text (bio, description), profile image, cover image, contact
info, and other display material. Linked by UUID. See
[data-model.md](../implementation/data-model.md).

---

## 2. Content nodes

Entities that are acted upon by actors.

| Node type | Description |
|-----------|-------------|
| **Post** | Content authored by a User or Collective (text, image, video). |
| **Comment** | A response to another content node — Post, Comment, Chat, ChatMessage, or Item. See [edges.md](edges.md) for the full list of valid comment targets. A full node because comments can be liked, disliked, and replied to. |
| **Chat** | A conversation container (group or 1:1). See [chats.md](../instances/chats.md). |
| **ChatMessage** | A single message within a chat. See [chats.md](../instances/chats.md). |
| **Item** | A physical or digital good. See [items.md](../instances/items.md). |
| **Hashtag** | A topic tag. Also covers concepts like places (e.g. `#berlin`) — if places ever need dedicated properties they can become their own node type later. |
| **Proposal** | A proposed change to a graph-side property on another node — the subject carrier for property-level governance votes (see [governance.md §2.1](governance.md)). Carries the target, the property name, and the proposed new value. When the vote crosses threshold, a cascade writes a new layer on the target property. |

### Graph-side properties

Most content nodes have minimal graph-side properties — the substance
lives in Postgres. Specific cases:

- **Chat**: `title` (if needed for routing or display hints) and
  `content_privacy` (plaintext vs E2EE — the graph needs this to
  know what to route). See [chats.md](../instances/chats.md).
- **Hashtag**: its tag string — the tag *is* the identifier. The
  UUID is content-addressed (`UUIDv5` of the canonical name with
  a fixed namespace); see
  [data-model.md "Node identity strategies"](../implementation/data-model.md)
  for the full mechanism and the federation implications.
- **Proposal**: `target_node_id`, `target_property`,
  `proposed_value`. A Proposal is fully specified by these three —
  no display content in Postgres. See
  [governance.md §2.1](governance.md) for the mechanism.

Post bodies, Comment bodies, ChatMessage payloads, Item descriptions
and media, Chat descriptions all live in Postgres — not on the graph.

---

## 3. Junction nodes

Junction nodes represent relationships that have **roles**, need
**approval flows** (multi-sig), and can themselves be interacted
with (liked, voted on, etc.).

| Node type | Connects | Why it's a node |
|-----------|----------|-----------------|
| **ChatMember** | Chat <-> User/Collective | Has roles (admin, mod, member). Entry can require multi-sig approval (invite-only chats). Can be interacted with (vote to kick, promote to admin). See [chats.md](../instances/chats.md). |
| **CollectiveMember** | Collective <-> User/Collective | Has roles (founder, shareholder, worker, band member, subsidiary, partner, member). Multi-sig for adding/removing members. Ownership stakes where applicable. Collectives can be members of other collectives (holdings, subsidiaries, label rosters, households as members of co-ops). See [collectives.md](../instances/collectives.md). |
| **ItemOwnership** | Item <-> User/Collective | Represents ownership claim. Multi-sig for transfer (acquirer requests, current owner approves). Full ownership history. See [items.md](../instances/items.md). |

### Why junction nodes exist

Junction nodes eliminate the need for parallel edges between the
same two nodes. A user's **membership** in a chat and their
**opinion** of that chat are edges to different nodes:

```
Jakob -[actor edge]-> ChatMember_Jakob_Chat1 -[structural]-> Chat1   (membership)
Jakob -[actor edge]-> Chat1                                          (opinion)
```

### Graph-side properties

**Junction nodes carry typed properties** — role and
role-attached quantities — as properties on the node itself, not
encoded in edge dimensions. Categorical data belongs in categorical
fields; quantities need more range and resolution than the bipolar
`[-1, +1]` edge dimensions provide. Multi-sig weighting for
approvals is derived from these role properties when actor edges
toward the junction are evaluated.

Per-type properties committed so far:

- **ChatMember**: `role` (`admin` / `mod` / `member`).
- **CollectiveMember**: `role` (`founder` / `shareholder` / `worker` /
  `band member` / `subsidiary`), plus `ownership_pct` where the
  role carries an equity stake.

### Postgres-side content

Junction nodes are lightweight relationship carriers and carry no
display content in Postgres. The entities they connect (Chats,
Collectives, Items) own the display content.

---

## What this doc is not

- **Not the conceptual model.** The three categories (actor,
  content, junction) and why they matter are in
  [graph-model.md §2](graph-model.md).
- **Not the Postgres schema.** Actual column definitions, version
  tables, and display-content shapes live in
  [data-model.md](../implementation/data-model.md).
- **Not the edge catalog.** For relationships between nodes, see
  [edges.md](edges.md).
