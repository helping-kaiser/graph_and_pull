# Structural Edge Map

A visual reference for every structural edge in the CoGra graph,
plus an audit of `(source, target)` pairs where two different
structural edge types could overlap.

The catalog this doc visualizes lives in
[edges.md §2](edges.md#2-structural-edges). The invariant the
audit feeds into is
[edges.md §2 "at most one structural edge per `(source, target)`
pair"](edges.md#2-structural-edges), surfaced in
[invariants.md](invariants.md#topology-and-visibility). This doc
adds no new mechanism — it makes the existing rules navigable.

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
| **ChatMember**       | `:BEARER`  | `:BEARER`  | —          | —          | `:CLAIM`   | `:STRUCTURAL` | —       | —          | `:STRUCTURAL` | `:STRUCTURAL` | —      | —          | —          |
| **CollectiveMember** | `:BEARER`  | `:BEARER`  | —          | —          | —          | —          | —          | —          | `:STRUCTURAL` | —      | `:STRUCTURAL` | —      | —          |
| **ItemOwnership**    | `:BEARER`  | `:BEARER`  | —          | —          | —          | —          | `:CLAIM`   | —          | —          | —          | —          | `:STRUCTURAL` | —       |
| **Network**          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          | —          |

**Notes on cells with two labels:**

- `Comment → (Post | Comment | Chat | ChatMessage | Item)` —
  `:CONTAINMENT` is the parent-of-comment edge (every Comment
  has exactly one, fixed at creation, per
  [comment.md §4](../instances/comment.md#4-edges));
  `:REFERENCES` is the embed/quote/mention edge. The same node
  pair can in principle host both — see §3.
- `ChatMessage → Chat` — `:CONTAINMENT` is the message's home
  chat ([chats.md §5.2](../instances/chats.md#52-chatmessage));
  `:REFERENCES` would be the message embedding its own home chat
  ([edges.md §2 "Reference"](edges.md#reference)). See §3.

`Proposal → Proposal` is `—` because a Proposal never targets
another Proposal — moderation can't target it and no governance
application proposes changes to a Proposal's own properties (per
[proposal.md §4](../instances/proposal.md#4-edges)).

`Proposal → Network` is `:TARGETS` (the `:Network` singleton is
targeted by parameter-amendment Proposals per
[network.md §11](network.md#11-amending-network-parameters)).

The `ChatMember → ChatMessage` Shape B edge is the
message-disavowal vote
([chats.md §10](../instances/chats.md#10-moderation)). The three
junction-to-Proposal `:STRUCTURAL` rows are Shape B vote edges
to a Proposal whose subject the junction is eligible on. The
junction-to-same-type-junction `:STRUCTURAL` cells are the
membership/ownership approver/removal vote edges.

---

## 2. Diagram

Same information visualized. The diagram groups edges by label;
the matrix above is the canonical reference.

```mermaid
flowchart LR
    %% Actor nodes
    User[User]:::actor
    Collective[Collective]:::actor

    %% Content nodes
    Post[Post]:::content
    Comment[Comment]:::content
    Chat[Chat]:::content
    ChatMessage[ChatMessage]:::content
    Item[Item]:::content
    Hashtag[Hashtag]:::content
    Proposal[Proposal]:::content

    %% Junction nodes
    ChatMember[ChatMember]:::junction
    CollectiveMember[CollectiveMember]:::junction
    ItemOwnership[ItemOwnership]:::junction

    %% System nodes
    Network[Network]:::system

    %% :CONTAINMENT
    Comment -->|CONTAINMENT| Post
    Comment -->|CONTAINMENT| Comment
    Comment -->|CONTAINMENT| Chat
    Comment -->|CONTAINMENT| ChatMessage
    Comment -->|CONTAINMENT| Item
    ChatMessage -->|CONTAINMENT| Chat

    %% :CLAIM (junction → parent)
    ChatMember -->|CLAIM| Chat
    CollectiveMember -->|CLAIM| Collective
    ItemOwnership -->|CLAIM| Item

    %% :APPROVAL (parent → junction)
    Chat -->|APPROVAL| ChatMember
    Collective -->|APPROVAL| CollectiveMember
    Item -->|APPROVAL| ItemOwnership

    %% :BEARER (junction → bearing actor)
    ChatMember -->|BEARER| User
    ChatMember -->|BEARER| Collective
    CollectiveMember -->|BEARER| User
    CollectiveMember -->|BEARER| Collective
    ItemOwnership -->|BEARER| User
    ItemOwnership -->|BEARER| Collective

    %% :TAGGING
    Post -->|TAGGING| Hashtag
    Comment -->|TAGGING| Hashtag
    Item -->|TAGGING| Hashtag

    %% :TARGETS (one node, fan-out to every other category)
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

    %% :REFERENCES — three carriers (ChatMessage, Post, Comment) to every node
    %% with graph identity except Hashtag for Post and Comment (:TAGGING carve-out).
    %% Edges drawn out per carrier for completeness.
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

    %% :STRUCTURAL — Shape B vote edges
    ChatMember -->|STRUCTURAL Shape B| ChatMember
    ChatMember -->|STRUCTURAL Shape B| ChatMessage
    ChatMember -->|STRUCTURAL Shape B| Proposal
    CollectiveMember -->|STRUCTURAL Shape B| CollectiveMember
    CollectiveMember -->|STRUCTURAL Shape B| Proposal
    ItemOwnership -->|STRUCTURAL Shape B| ItemOwnership

    classDef actor    fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
    classDef content  fill:#fff3e0,stroke:#ef6c00,color:#e65100;
    classDef junction fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c;
    classDef system   fill:#eceff1,stroke:#455a64,color:#263238;
```

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
