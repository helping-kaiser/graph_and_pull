# Structural Edge Map

A visual reference for every structural edge in the CoGra graph,
plus an audit of `(source, target)` pairs where two different
structural edge types could overlap.

The catalog this doc visualizes lives in
[edges.md §2](edges.md#2-structural-edges). The invariant the
audit feeds into is
[edges.md §2 "at most one edge label per `(source, target)`
pair"](edges.md#2-structural-edges), surfaced in
[invariants.md](invariants.md#topology-and-visibility). This doc
visualizes the structural slice of that rule; the actor edges
covered by the same rule live in
[edges.md §1](edges.md#1-actor-edges). This doc adds no new
mechanism — it makes the existing rules navigable.

For the conceptual model (categories, dimensions, append-only),
see [graph-model.md](graph-model.md). For the per-node edge
catalogs that this doc aggregates, see each node's per-node doc
listed in [nodes.md](nodes.md).

---

## 1. Matrix

Rows are **source** node types; columns are **target** node
types. Cells list every structural edge label that can run from
that source to that target. `—` marks pairs with no structural
edge.

`:STRUCTURAL` denotes Shape B vote edges
([edges.md §2 "Voting (Shape B)"](edges.md#voting-shape-b)) —
the only structural-edge family that doesn't take one of the
seven sub-category labels.

Sources and targets with no structural edges in either direction
(`Network`) are still listed so the absence is explicit.

|                      | User       | Coll.      | Post       | Comment    | Chat       | ChatMsg    | Item       | Hashtag    | Proposal   | ChatMbr    | CollMbr    | ItemOwn    | Network    |
|----------------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|
| **User**             | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          |
| **Collective**       | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | `:APPROVAL`| —          | —          |
| **Post**             | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:TAGGING` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` |
| **Comment**          | `:REFERENCES` | `:REFERENCES` | `:CONTAINMENT` `:REFERENCES` | `:CONTAINMENT` `:REFERENCES` | `:CONTAINMENT` `:REFERENCES` | `:CONTAINMENT` `:REFERENCES` | `:CONTAINMENT` `:REFERENCES` | `:TAGGING` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` |
| **Chat**             | —          | —          | —          | —          | —          | —          | —          | —          | —          | `:APPROVAL`| —          | —          | —          |
| **ChatMessage**      | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:CONTAINMENT` `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` | `:REFERENCES` |
| **Item**             | —          | —          | —          | —          | —          | —          | —          | `:TAGGING` | —          | —          | —          | `:APPROVAL`| —          |
| **Hashtag**          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          |
| **Proposal**         | `:TARGETS` | `:TARGETS` | `:TARGETS` | `:TARGETS` | `:TARGETS` | `:TARGETS` | `:TARGETS` | `:TARGETS` | —          | `:TARGETS` | `:TARGETS` | `:TARGETS` | `:TARGETS` |
| **ChatMember**       | `:BEARER`  | `:BEARER`  | —          | —          | `:CLAIM`   | —          | —          | —          | `:STRUCTURAL` | `:STRUCTURAL` | —      | —          | —          |
| **CollectiveMember** | `:BEARER`  | `:BEARER`  | —          | —          | —          | —          | —          | —          | `:STRUCTURAL` | —      | `:STRUCTURAL` | —      | —          |
| **ItemOwnership**    | `:BEARER`  | `:BEARER`  | —          | —          | —          | —          | `:CLAIM`   | —          | —          | —          | —          | `:STRUCTURAL` | —       |
| **Network**          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          |

**Reading cells with two labels.** Cells that list both
`:CONTAINMENT` and `:REFERENCES` show two valid edge types **at
the class level** — a Comment can in general be in a containment
relationship with some Post and in a reference relationship with
a different Post. For any **specific instance** pair, only one of
the two fires, per the rule in
[edges.md §2 "Reference"](edges.md#reference): `:REFERENCES` is
not written when another structural edge already encodes the same
`(source, target)` pair, so `:CONTAINMENT` wins when both would
otherwise apply.

The two-label cells in the matrix are:

- `Comment → (Post | Comment | Chat | ChatMessage | Item)` —
  `:CONTAINMENT` for the Comment's parent (every Comment has
  exactly one, fixed at creation per
  [comment.md §4](../instances/comment.md#4-edges)); `:REFERENCES`
  for embed/quote/mention of *any other* node of the same type.
- `ChatMessage → Chat` — `:CONTAINMENT` for the message's home
  chat ([chats.md §5.2](../instances/chats.md#52-chatmessage));
  `:REFERENCES` for embedding any *other* chat (the
  personal-newsfeed shape from
  [chats.md §8](../instances/chats.md#8-chatmessages-as-first-class-content)).

`Proposal → Proposal` is `—` because a Proposal never targets
another Proposal — moderation can't target it and no governance
application proposes changes to a Proposal's own properties (per
[proposal.md §4](../instances/proposal.md#4-edges)).

`Proposal → Network` is `:TARGETS` (the `:Network` singleton is
targeted by parameter-amendment Proposals per
[network.md §11](network.md#11-amending-network-parameters)).

`Proposal → Hashtag` is `:TARGETS` but only moderation
classification Proposals reach Hashtag — `name` is immutable
outside the redaction cascade
([hashtag.md §5](../instances/hashtag.md#5-lifecycle)):

- `'sensitive'` classification:
  `target_property = 'moderation_status'`,
  `proposed_value = 'sensitive'`. Flips the flag, no redaction.
- `'illegal'` classification: `target_property ∈ {'name', 'node'}`
  (the two are equivalent for hashtag because `name` is the only
  user-input field — `'node'` is the whole-node sentinel per
  [nodes.md "Whole-node targeting"](nodes.md#whole-node-targeting-the-node-sentinel)),
  `proposed_value = 'illegal'`. Fires the redaction cascade per
  [moderation.md §1](../instances/moderation.md#1-the-two-classification-paths).

A property-amendment Proposal with `target_property = 'name'` and
any other `proposed_value` is inadmissible.

The three junction-to-Proposal `:STRUCTURAL` rows are Shape B
vote edges to a Proposal whose subject the junction is eligible
on — including the chat-internal disavowal Proposals (both Level
1 against a `ChatMessage` and Level 2 against another
`ChatMember`) that flow through `ChatMember → Proposal` per
[chats.md §10](../instances/chats.md#10-moderation). The
junction-to-same-type-junction `:STRUCTURAL` cells are the
admission and (for `CollectiveMember` / `ItemOwnership`)
removal vote edges; the `ChatMember → ChatMember` row is
admission-only since chat disavowal routes through a Proposal
instead.

---

## 2. Diagrams

Same information as the matrix, split one diagram per edge-label
family. A single combined diagram is dominated by `:REFERENCES`
(~35 edges) and `:TARGETS` (12 edges) fan-outs and reads as a
hairball; splitting by family makes each family's shape visible.
The matrix above remains the canonical reference.

### 2.1. `:CONTAINMENT`

Parent-pointer edges: every Comment contains into its parent (any
of the five content types); every ChatMessage contains into its
home Chat
([edges.md "Containment / belonging"](edges.md#containment--belonging)).

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart LR
    Comment[Comment]:::content
    ChatMessage[ChatMessage]:::content
    Post[Post]:::content
    Chat[Chat]:::content
    Item[Item]:::content

    Comment -->|CONTAINMENT| Post
    Comment -->|CONTAINMENT| Comment
    Comment -->|CONTAINMENT| Chat
    Comment -->|CONTAINMENT| ChatMessage
    Comment -->|CONTAINMENT| Item
    ChatMessage -->|CONTAINMENT| Chat

    classDef content fill:#fff3e0,stroke:#ef6c00,color:#e65100;
```

### 2.2. Junction triad: `:CLAIM`, `:APPROVAL`, `:BEARER`

Every junction sits in the same three-legged shape: junction →
parent (`:CLAIM`), parent → junction (`:APPROVAL`), junction →
bearing actor (`:BEARER`). Three junction types instantiate the
pattern (see
[edges.md "Approval completion"](edges.md#approval-completion) and
[edges.md "Bearer binding"](edges.md#bearer-binding)).
Note `Collective` plays two roles — bearing actor for all three
junctions, and parent of `CollectiveMember`.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart LR
    User[User]:::actor
    Collective[Collective]:::actor

    Chat[Chat]:::content
    Item[Item]:::content

    ChatMember[ChatMember]:::junction
    CollectiveMember[CollectiveMember]:::junction
    ItemOwnership[ItemOwnership]:::junction

    ChatMember -->|CLAIM| Chat
    Chat -->|APPROVAL| ChatMember
    ChatMember -->|BEARER| User
    ChatMember -->|BEARER| Collective

    CollectiveMember -->|CLAIM| Collective
    Collective -->|APPROVAL| CollectiveMember
    CollectiveMember -->|BEARER| User
    CollectiveMember -->|BEARER| Collective

    ItemOwnership -->|CLAIM| Item
    Item -->|APPROVAL| ItemOwnership
    ItemOwnership -->|BEARER| User
    ItemOwnership -->|BEARER| Collective

    classDef actor    fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
    classDef content  fill:#fff3e0,stroke:#ef6c00,color:#e65100;
    classDef junction fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c;
```

### 2.3. `:TAGGING`

Three content types tag Hashtags directly; Hashtags never tag
back ([edges.md "Tagging"](edges.md#tagging)).

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart LR
    Post[Post]:::content
    Comment[Comment]:::content
    Item[Item]:::content
    Hashtag[Hashtag]:::content

    Post -->|TAGGING| Hashtag
    Comment -->|TAGGING| Hashtag
    Item -->|TAGGING| Hashtag

    classDef content fill:#fff3e0,stroke:#ef6c00,color:#e65100;
```

### 2.4. `:TARGETS`

Single-source fan-out: a Proposal points at the subject of its
proposed change, which can be any node category — including the
`Network` singleton and any junction — but never another Proposal
([edges.md "Subject targeting"](edges.md#subject-targeting)).

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart TD
    Proposal[Proposal]:::content

    User[User]:::actor
    Collective[Collective]:::actor
    Post[Post]:::content
    Comment[Comment]:::content
    Chat[Chat]:::content
    ChatMessage[ChatMessage]:::content
    Item[Item]:::content
    Hashtag[Hashtag]:::content
    ChatMember[ChatMember]:::junction
    CollectiveMember[CollectiveMember]:::junction
    ItemOwnership[ItemOwnership]:::junction
    Network[Network]:::system

    Proposal -->|TARGETS| User
    Proposal -->|TARGETS| Collective
    Proposal -->|TARGETS| Post
    Proposal -->|TARGETS| Comment
    Proposal -->|TARGETS| Chat
    Proposal -->|TARGETS| ChatMessage
    Proposal -->|TARGETS| Item
    Proposal -->|TARGETS| Hashtag
    Proposal -->|TARGETS| ChatMember
    Proposal -->|TARGETS| CollectiveMember
    Proposal -->|TARGETS| ItemOwnership
    Proposal -->|TARGETS| Network

    classDef actor    fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
    classDef content  fill:#fff3e0,stroke:#ef6c00,color:#e65100;
    classDef junction fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c;
    classDef system   fill:#eceff1,stroke:#455a64,color:#263238;
```

### 2.5. `:REFERENCES`

Three carriers — `Post`, `Comment`, `ChatMessage` — can reference
any node with graph identity (everything except `Network`).
`Post` and `Comment` use `:TAGGING` for Hashtag instead, so
Hashtag is excluded from their fan-out;
`ChatMessage`'s fan-out includes Hashtag
([edges.md "Reference"](edges.md#reference)). The no-duplicate
rule means `:REFERENCES` is suppressed for `(source, target)`
pairs that already carry another structural edge — see
[§1](#1-matrix) for the cells where this applies.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart LR
    ChatMessage[ChatMessage]:::content
    Post[Post]:::content
    Comment[Comment]:::content

    User[User]:::actor
    Collective[Collective]:::actor
    Chat[Chat]:::content
    Item[Item]:::content
    Hashtag[Hashtag]:::content
    Proposal[Proposal]:::content
    ChatMember[ChatMember]:::junction
    CollectiveMember[CollectiveMember]:::junction
    ItemOwnership[ItemOwnership]:::junction

    ChatMessage -->|REFERENCES| User
    ChatMessage -->|REFERENCES| Collective
    ChatMessage -->|REFERENCES| Post
    ChatMessage -->|REFERENCES| Comment
    ChatMessage -->|REFERENCES| Chat
    ChatMessage -->|REFERENCES| ChatMessage
    ChatMessage -->|REFERENCES| Item
    ChatMessage -->|REFERENCES| Hashtag
    ChatMessage -->|REFERENCES| Proposal
    ChatMessage -->|REFERENCES| ChatMember
    ChatMessage -->|REFERENCES| CollectiveMember
    ChatMessage -->|REFERENCES| ItemOwnership

    Post -->|REFERENCES| User
    Post -->|REFERENCES| Collective
    Post -->|REFERENCES| Post
    Post -->|REFERENCES| Comment
    Post -->|REFERENCES| Chat
    Post -->|REFERENCES| ChatMessage
    Post -->|REFERENCES| Item
    Post -->|REFERENCES| Proposal
    Post -->|REFERENCES| ChatMember
    Post -->|REFERENCES| CollectiveMember
    Post -->|REFERENCES| ItemOwnership

    Comment -->|REFERENCES| User
    Comment -->|REFERENCES| Collective
    Comment -->|REFERENCES| Post
    Comment -->|REFERENCES| Comment
    Comment -->|REFERENCES| Chat
    Comment -->|REFERENCES| ChatMessage
    Comment -->|REFERENCES| Item
    Comment -->|REFERENCES| Proposal
    Comment -->|REFERENCES| ChatMember
    Comment -->|REFERENCES| CollectiveMember
    Comment -->|REFERENCES| ItemOwnership

    classDef actor    fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
    classDef content  fill:#fff3e0,stroke:#ef6c00,color:#e65100;
    classDef junction fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c;
```

### 2.6. `:STRUCTURAL` (Shape B vote edges)

Junctions cast Shape B votes: each junction type votes on
Proposals targeting subjects it is eligible on, and
`CollectiveMember` / `ItemOwnership` also vote directly on
other junctions of the same type (member removal /
co-ownership approval). `ChatMember → ChatMember` is
admission-only — chat disavowal routes through a Proposal per
[chats.md §10](../instances/chats.md#10-moderation). See
[edges.md "Voting (Shape B)"](edges.md#voting-shape-b).

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
flowchart LR
    ChatMember[ChatMember]:::junction
    CollectiveMember[CollectiveMember]:::junction
    ItemOwnership[ItemOwnership]:::junction
    Proposal[Proposal]:::content

    ChatMember -->|STRUCTURAL Shape B| ChatMember
    ChatMember -->|STRUCTURAL Shape B| Proposal
    CollectiveMember -->|STRUCTURAL Shape B| CollectiveMember
    CollectiveMember -->|STRUCTURAL Shape B| Proposal
    ItemOwnership -->|STRUCTURAL Shape B| ItemOwnership

    classDef content  fill:#fff3e0,stroke:#ef6c00,color:#e65100;
    classDef junction fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c;
```

---

## 3. Feed-ranking traversability

Per label, whether feed-ranking paths cross it and which rule
governs. The matrix above shows *which* structural label sits at
each `(source, target)` pair; the table below summarizes *whether
and how* the ranking walk crosses them.

| Label | Crossable for feed ranking? | Where the rule lives |
|---|---|---|
| `:ACTOR` / `:AUTHOR` | Yes — carry opinion content, contribute factors | [feed-ranking.md §3.1](feed-ranking.md#31-which-edges-contribute-factors) |
| `:CONTAINMENT` | Yes — counts toward `R`, no factor contribution | [feed-ranking.md §3.1](feed-ranking.md#31-which-edges-contribute-factors) |
| `:CLAIM` | Yes — gated by own top-layer `dim1 > 0` (state-bearing) | [feed-ranking.md §3.1](feed-ranking.md#31-which-edges-contribute-factors) |
| `:APPROVAL` | **No outbound** — state-bearing identity, not transit | [feed-ranking.md §3.5 rule 1](feed-ranking.md#35-traversal-restrictions) |
| `:BEARER` | **No** — identity binding, not transit | [feed-ranking.md §3.5 rule 2](feed-ranking.md#35-traversal-restrictions) |
| `:TARGETS` | **No outbound** — governance reference, not relevance | [feed-ranking.md §3.5 rule 3](feed-ranking.md#35-traversal-restrictions) |
| `:TAGGING` | **No** — cosmetic discovery only | [hashtag.md §4](../instances/hashtag.md#4-edges) |
| `:REFERENCES` | Yes — endpoint-restricted (User/Collective ⇒ terminate after one `:AUTHOR` hop) + fanout-budget composition | [feed-ranking.md §3.5 rules 4 & 5](feed-ranking.md#35-traversal-restrictions) |
| `:STRUCTURAL` (Shape B) | Yes — sibling-case note in feed-ranking §3.5 | [feed-ranking.md §3.5](feed-ranking.md#35-traversal-restrictions) |

Forward-only traversal is the foundation
([feed-ranking.md §3 invariant](feed-ranking.md#3-per-edge-composition-along-a-path));
the per-label restrictions above close the bot-amplification gaps
forward-only alone doesn't cover. See the rule bodies in
[feed-ranking.md §3.5](feed-ranking.md#35-traversal-restrictions)
for the specific attack each rule closes.

---

## What this doc is not

- **Not the catalog.** Row-level meanings, label assignments,
  and dimension semantics live in
  [edges.md](edges.md). This doc is a visual aggregation.
- **Not the conceptual model.** Categories, directionality,
  append-only — see [graph-model.md](graph-model.md).
- **Not the Memgraph schema.** Concrete edge-property types
  live in
  [graph-data-model.md](../implementation/graph-data-model.md).
