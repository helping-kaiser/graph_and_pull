# Chats

Chats on CoGra are **not** what they are on WhatsApp, Signal, or
iMessage. This doc exists because assuming otherwise leads to
wrong designs.

This doc is the per-node catalog for three related nodes: the
**Chat** container, the **ChatMessage** content node, and the
**ChatMember** junction. Mechanics those topics depend on stay
in their topical docs — this doc links rather than duplicates.

---

## 1. Mental model reset

In most messaging apps a chat is a **private, hidden space**.
Membership is invisible to outsiders; content is the only privacy
layer, via end-to-end encryption. The conversation effectively
does not exist from the outside.

In CoGra, a chat is a **public node on the graph**. Its
existence, its member list, and its message count are visible to
every actor on the graph (see the transparency principle in
[graph-model.md §1](../primitive/graph-model.md#1-core-principles)).
**Topology is always public**; what's private is the *content* of
individual messages, if the chat chose to encrypt them.

**Invariant:** Chat topology — the existence of the Chat node, its
member set, who-talks-to-whom — is always public. Only the **body**
of individual ChatMessages is private, and only when
`content_privacy = 'encrypted'` (§4.2). There is no "private chat"
mode that hides membership or message metadata from the graph.

Chats and ChatMessages are **first-class interactable nodes**.
Users can like them, comment on them, and rank them in feeds —
just like posts.

This feels wrong if you map "chat" onto "group DM." It feels
natural if you map "chat" onto "public discussion space that
happens to have members, some of which may choose to run with
encrypted content."

---

## 2. Creation

Founding a Chat is a single compound gesture. ChatMessages and
subsequent ChatMembers are created via the regular authoring and
join-flow patterns once a chat exists; ChatMember creation in
particular is covered in §11.

### 2.1 Chat

A Chat is created by a single compound gesture from one actor —
either a **User or a Collective**. Like a Collective or Item,
Chat creation is **compound**: it brings the Chat AND the
founder's first ChatMember into existence in one atomic step,
with the founder as the inaugural admin. A Collective founding a
Chat is the same gesture, initiated by an authorized member per
[collectives.md §2](collectives.md#2-acting-through-the-collective).

The gesture writes the following records atomically:

- A new `:Chat` node on the graph.
- The Postgres `chats` row carrying `name` (nullable for 1:1
  chats), `description`, and `image_id` (see
  [data-model.md](../implementation/data-model.md)).
- The founder's `User/Collective → Chat` actor edge — the
  **authorship edge** (§6.1).
- A new `:ChatMember` junction node for the founder, with
  `role = 'admin'`.
- The `ChatMember → User/Collective` `:BEARER` structural edge,
  binding the junction to the founder.
- The founder's `User/Collective → ChatMember` actor edge — the
  Shape A self-claim.
- The `ChatMember → Chat` claim edge.
- The `Chat → ChatMember` approval edge with positive top layer.

Because there is no prior member to approve the founder, the
[two-edge approval pattern](../primitive/graph-model.md#5-junction-node-flows)
collapses to its 1-of-1 special case: the founder's gesture acts
as both the claim and the approval. This is the same bootstrap
pattern used by Collective founders
([collectives.md "Creation"](collectives.md#1-creation)) and Item
authors ([items.md §1](items.md#1-creation)). Every subsequent
ChatMember addition follows §11, not this bootstrap.

The chat opens at epoch `E₁` (`Chat.epoch = 1`); as soon as a
second active member is approved, `Chat.epoch` advances per §9.

### 2.2 ChatMessage

A ChatMessage is created by a single authoring gesture from an
active ChatMember of the chat — the same shape a Post or Comment
uses. The gesture writes atomically:

- A new `:ChatMessage` node carrying `moderation_status` (§3.2).
  `content_privacy` is a Postgres-side property on the
  `chat_messages` row (§4.2).
- The Postgres `chat_messages` row carrying the message body
  (plaintext or ciphertext), the `epoch` index for encrypted
  messages, and any attached media (§4.2).
- The author's `User → ChatMessage` actor edge — the
  **authorship edge** (§6.2).
- A system-created `ChatMessage → Chat` `:CONTAINMENT` edge.
- One system-created `ChatMessage → X` `:REFERENCES` edge per
  embedded/quoted/mentioned node (§8 walks the embedding gesture
  end-to-end).

The graph never holds the message body; encryption is a Postgres
concern (§4.2, §9).

---

## 3. Graph-side properties

### 3.1 Chat

A Chat node carries:

- **`name`** — optional routing/display hint. Layered.
- **`join_policy`** — one of `open`, `invite-only`,
  `request-entry` (§11). Layered.
- **`invite_proposer_roles`** — list of `ChatMember.role`
  values whose bearers may propose a new ChatMember under
  `'invite-only'`. Inapplicable to `'open'` (no proposer needed)
  and `'request-entry'` (the would-be member proposes
  themselves). Layered.
- **`entry_approval_required_count`** — integer N ≥ 0. Number
  of Shape B approver votes the new ChatMember's junction must
  collect before the system writes the `Chat → ChatMember`
  approval edge. `0` under `'open'`; `1` for a standard
  `'invite-only'` or `'request-entry'`; higher values produce
  the multi-sig configuration shape per §11 "Higher N". Layered.
- **`entry_approval_eligible_roles`** — list of
  `ChatMember.role` values whose bearers' Shape B votes count
  toward `entry_approval_required_count`. Inapplicable to
  `'open'`. Layered.
- **`epoch`** — integer chat-key-rotation counter (§9). Default
  `1`. Layered. Advances by `1` on every membership change and
  on every passing mid-epoch rotation Proposal.
- **`moderation_status`** — `'normal'` / `'sensitive'` /
  `'illegal'`, layered, default `'normal'`. Universal across
  user-input-bearing nodes; mechanics in
  [nodes.md](../primitive/nodes.md#universal-moderation_status)
  and §13.1.

A Chat also carries a set of **governance-parameter properties**
— role weights, disavowal quorum/threshold pairs, property-change
quorum/threshold pairs, and the mid-epoch key-rotation
quorum/threshold — all layered and addressable as targets of
property-change Proposals. The full list with defaults lives in
the §10 tables.

Concrete property types and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

### 3.2 ChatMessage

A ChatMessage node carries:

- **`moderation_status`** — same shape as the Chat property.

Concrete types and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

### 3.3 ChatMember

A ChatMember junction carries:

- **`role`** — closed enum: `'admin'`, `'chat_mod'`, `'member'`.
  Closed because each value carries a fixed mechanical power set
  (§3.4) and a default weight on the Chat (§10) — open-ended role
  strings would require every chat to supply matching weight and
  power properties, which the per-property property-change Proposal
  shape cannot do without paying a uniformity cost.
  The `'chat_mod'` label is deliberately distinct from the
  Network-scope `User.network_role = 'moderator'`: chat
  moderators and Network moderators are different roles, with
  different scopes and different weights — see
  [governance.md §7](../primitive/governance.md#7-the-mod-gate).
  Layered. The default role weights are properties on the
  **Chat** (§3.1), not on the ChatMember — change the role to
  reassign the bearer; change the Chat's weight properties to
  re-tune all bearers of that role.
- **`voting_weight`** — optional. When present, overrides the
  role-based weight derivation at tally time (§10 "How roles fit
  in"). Layered.

Junction nodes carry no `moderation_status` per
[nodes.md](../primitive/nodes.md#universal-moderation_status) —
ChatMember has no user-input fields.

### 3.4 ChatMember roles and their powers

The three `ChatMember.role` values carry the following
mechanical powers. Every power is mediated by a `Chat`
property — none is hardcoded into the role.

| Power | Carrier property on `Chat` | Default for `'admin'` | Default for `'chat_mod'` | Default for `'member'` |
|---|---|---|---|---|
| Propose an invitation under `'invite-only'`            | `invite_proposer_roles` (§3.1)        | yes | yes | no  |
| Cast a counting approver vote (any join policy)        | `entry_approval_eligible_roles` (§3.1) | yes | yes | no  |
| Cast a counting vote in chat-internal disavowal (§10)  | role weight on `Chat` (§10 "How roles fit in") | weight `5` | weight `3` | weight `1` |
| Cast a counting vote on chat property-change Proposals | role weight on `Chat`                 | weight `5` | weight `3` | weight `1` |
| Vote-weight override on a per-bearer basis             | `ChatMember.voting_weight` (§3.3)     | nullable | nullable | nullable |

The defaults are starting points, not fixed rules: every
property above is a layered `Chat`-node property, amendable via
a property-change Proposal (§10 "Property and role changes via
Proposals"). A chat that wants admins to lose the unilateral
proposer privilege simply removes `'admin'` from
`invite_proposer_roles`; one that wants every member to count
as an approver adds `'member'` to `entry_approval_eligible_roles`.

No role grants a unilateral disavowal or a unilateral property
change: every act runs through a Proposal vote, weighted but
never veto-bearing. The admin's higher weight matters only at
the margin where a tally is close.

---

## 4. Postgres-side content

### 4.1 Chat

A Chat's display content lives in Postgres on the `chats` row,
linked to the graph Chat node by UUID. Edits are append-only per
[layers.md §4](../primitive/layers.md#4-layers-on-postgres-side-display-content):
a new version row, no overwrite.

- **`description`** — optional body text. The longer-form
  explanation of what the chat is for, beyond the short `name`
  routing hint (§3.1).
- **Image** — optional chat avatar/header, pointed at by the
  `image_id` column referencing one `media_attachments` asset,
  owned by the same author as the Chat (anti-hijack rule per
  [data-model.md "Why parents point at attachments"](../implementation/data-model.md#why-parents-point-at-attachments)).
  "Image" is the *concept* in chat docs; `image_id` is the SQL
  column — the same `concept → column` pattern as User `avatar` /
  `avatar_id`.

`description` and the image are the two display fields an
`'illegal'` Proposal can target; together with the graph-side
`name`, they make up the Chat's user-input field set per
[moderation.md §5](moderation.md#5-scope). Concrete schema lives
in [data-model.md](../implementation/data-model.md).

### 4.2 ChatMessage

A ChatMessage's body lives in Postgres on the `chat_messages`
row, linked to the graph ChatMessage node by UUID:

- **`content_privacy`** — `'plaintext'` / `'encrypted'`.
  Per-message, not per-chat — a single chat can mix both freely
  (§9). The frontend reads this with the body to decide whether
  to attempt decryption.
- **`content`** — the message body. For
  `content_privacy = 'plaintext'` this is readable text; for
  `content_privacy = 'encrypted'` this is a ciphertext blob
  under the chat's member-derived symmetric key for the
  message's epoch. The graph never reads either form.
- **`epoch`** — for encrypted messages, the index of the chat
  key the body was encrypted under (§9). NULL for plaintext rows
  under the schema CHECK in
  [data-model.md](../implementation/data-model.md).
- **Attachments** — images and other media via the
  `chat_message_attachments` junction table referencing
  `media_attachments`, same anti-hijack rule as Chat images.

Edits are append-only per
[layers.md §4](../primitive/layers.md#4-layers-on-postgres-side-display-content):
a correction writes a new version into the row; past versions
remain readable.

`content` and `attachments` are the two fields an `'illegal'`
Proposal can target per
[moderation.md §5](moderation.md#5-scope) — for both plaintext
and encrypted bodies (encrypted-message classification path in
[moderation.md "Encrypted message classification"](moderation.md#encrypted-message-classification)).

### 4.3 ChatMember

None. ChatMember is a pure graph-side junction node — no
Postgres-side display content. Membership state lives entirely
on the graph (the two-edge approval pair per §5.3) with role and
weight on the junction itself (§3.3).

---

## 5. Edges

This doc covers three nodes; each gets its own subsection.
Dimension labels, sub-category labels, and traversal semantics
are not duplicated here — see [edges.md](../primitive/edges.md).

### 5.1 Chat

#### As source (outgoing)

A Chat is not an actor and authors no actor edges. It carries
one outgoing structural edge type:

- **`Chat → ChatMember` (`:APPROVAL`)** — the approval side of
  the two-edge approval pattern. Created when the chat's
  `join_policy` is satisfied for an incoming `ChatMember` claim
  (§11). State transitions on this edge — voluntary leave (§11
  "Leaving and removal") and the system-written cascade from a
  passing Level 2 disavowal Proposal (§10) — append additional
  `dim1 < 0` layers per
  [graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows).
  See
  [edges.md §2 "Approval completion"](../primitive/edges.md#approval-completion).

#### As target (incoming)

A Chat receives:

- **Actor edges** from Users and Collectives per
  [edges.md §1](../primitive/edges.md#1-actor-edges) — sentiment
  and relevance toward the chat itself, used by
  [feed-ranking](../primitive/feed-ranking.md). The earliest of
  these is the authorship edge (§6.1).
- **`ChatMessage → Chat` (`:CONTAINMENT`)** — each message
  belongs to its chat.
- **`ChatMember → Chat` (`:CLAIM`)** — the claim side of the
  two-edge approval pattern, paired with the outgoing
  `Chat → ChatMember` (above).
- **`Comment → Chat` (`:CONTAINMENT`)** when a Comment is on the
  chat as a whole.
- **`ChatMessage / Post / Comment → Chat` (`:REFERENCES`)** when
  another content node embeds the Chat. See
  [edges.md §2 "Reference"](../primitive/edges.md#reference).
- **`Proposal → Chat` (`:TARGETS`)** when a Proposal targets a
  property on the Chat — `name`, `join_policy`, `epoch` (§9
  mid-epoch rotation), `moderation_status`, or any
  governance-parameter property (§10). See
  [edges.md §2 "Subject targeting"](../primitive/edges.md#subject-targeting).

### 5.2 ChatMessage

#### As source (outgoing)

A ChatMessage is not an actor and authors no actor edges. It
carries two outgoing structural edge types, both system-created:

- **`ChatMessage → Chat` (`:CONTAINMENT`)** — identifies the
  message's containing chat. Exactly one per ChatMessage, written
  at creation and never re-targeted.
- **`ChatMessage → any node` (`:REFERENCES`)** — one edge per
  embedded/quoted/mentioned node. **Hashtag is included** —
  unlike Post and Comment, ChatMessage has no `:TAGGING` edge
  type, so body-tag hashtags also go through `:REFERENCES`. The
  message's own **home Chat is excluded** by the single-
  structural-edge invariant: the `:CONTAINMENT` edge already
  encodes the `(this message, its home chat)` pair, so a message
  that embeds its own home chat does not write a parallel
  `:REFERENCES` edge — the embed renders from the existing
  `:CONTAINMENT` edge. Embedding *other* chats from a message
  remains a regular `:REFERENCES` edge. The carve-out rationale
  and traversal rules live in
  [edges.md §2 "Reference"](../primitive/edges.md#reference); §8
  walks the gesture end-to-end.

#### As target (incoming)

A ChatMessage receives:

- **Actor edges** from Users and Collectives per
  [edges.md §1](../primitive/edges.md#1-actor-edges) — the
  like/dislike surface plus per-viewer relevance. The earliest
  is the authorship edge (§6.2).
- **`Comment → ChatMessage` (`:CONTAINMENT`)** when a Comment is
  written on the specific message.
- **`ChatMessage / Post / Comment → ChatMessage` (`:REFERENCES`)**
  when another content node embeds this message.
- **`Proposal → ChatMessage` (`:TARGETS`)** when a Proposal
  targets the ChatMessage — `'sensitive'` against
  `moderation_status`, `'illegal'` against `content` or
  `attachments`, or the `'node'` sentinel for Level 1
  chat-internal disavowal (§10). The disavowal vote carrier is
  the voter's `ChatMember` junction (Shape B), so the chat
  stance stays decoupled from personal sentiment on
  `User → ChatMessage`.

### 5.3 ChatMember

#### As source (outgoing)

A ChatMember is a junction, not an actor. It carries one claim
edge, one bearer-binding edge, plus the Shape B vote edges its
bearer casts as a chat-eligible voter:

- **`ChatMember → Chat` (`:CLAIM`)** — the claim side of the
  two-edge approval pattern, closed by the chat's
  `Chat → ChatMember` approval edge (§5.1) once `join_policy` is
  satisfied (§11).
- **`ChatMember → User/Collective` (`:BEARER`)** — identity-
  binding edge written at junction creation, pointing at the
  actor the membership represents. Never re-pointed; the Shape A
  self-claim that activates the membership must originate from
  this actor (§11).
- **`ChatMember → Proposal` (Shape B vote)** — chat-eligible
  vote on any Proposal targeting a chat-internal subject: a
  chat property (§10 "Property and role changes"), a
  ChatMessage for Level 1 disavowal (§10), or a ChatMember
  junction for Level 2 disavowal (§10).
- **`ChatMember → ChatMember` (Shape B vote)** — admission
  vote on another chat member's membership at join time (§11
  Invite-only, Request-entry, multi-sig). Stance flips on this
  edge happen only during the open admission period — once a
  membership is active, disavowal flows through a Proposal
  (§10 Level 2), not through a re-layering of this edge.

#### As target (incoming)

A ChatMember receives:

- **Actor edges** from Users and Collectives per
  [edges.md §1](../primitive/edges.md#1-actor-edges). For the
  bearer themselves, the `User/Collective → ChatMember` edge is
  the **Shape A self-claim** that initiates the membership
  (§11). For other actors, these edges are personal sentiment
  about that membership — they do not drive the approval vote,
  which uses Shape B (above).
- **`ChatMember → ChatMember` (Shape B vote)** — incoming
  admission votes from other ChatMembers of the same chat
  during the open admission period (§11). Disavowal of an
  active member (§10 Level 2) flows through a Proposal, not
  through this edge.
- **`Chat → ChatMember` (`:APPROVAL`)** — the approval side of
  the two-edge pattern, paired with the outgoing
  `ChatMember → Chat` claim above. State transitions —
  voluntary leave, and the system-written cascade from a
  passing Level 2 disavowal Proposal (§10) — append
  `dim1 < 0` layers per
  [graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows).
- **`ChatMessage / Post / Comment → ChatMember`
  (`:REFERENCES`)** when a content node embeds the membership
  — e.g. a message highlighting a moderator.
- **`Proposal → ChatMember` (`:TARGETS`)** when a Proposal
  targets the ChatMember — `role` changes (promote / demote
  per §10), `voting_weight`, or the `'node'` sentinel for Level
  2 disavowal (§10).

---

## 6. Authorship

### 6.1 Chat

A Chat is authored under the standard rule
([authorship.md](../primitive/authorship.md)): the founder is
the actor whose incoming actor edge has the earliest layer-1
timestamp. The founder's `User/Collective → Chat` actor edge is
written in the same compound gesture as the Chat node (§2.1);
by construction it is the earliest, and it carries the
`:AUTHOR` sub-label. **Unlike other authored nodes, the Chat
does not cache its author Postgres-side** — no `author_id` is
materialized on the `chats` row. This is a deliberate deviation
from [authorship.md "Caching"](../primitive/authorship.md#caching):
a chat's meaningful identity is its membership set, not its
founder, and the rare "who founded this?" query can scan the
`:AUTHOR` edge on demand.

### 6.2 ChatMessage

Standard rule. The author's `User → ChatMessage` actor edge is
written in the same authoring gesture as the ChatMessage node
(§2.2) and carries the `:AUTHOR` sub-label; `author_id` is
cached on the `chat_messages` Postgres row for display.

### 6.3 ChatMember

ChatMember is a junction node and has no authorship in the
[authorship.md](../primitive/authorship.md) sense — it
represents a membership relationship, not an authored piece of
content. The same exemption applies to CollectiveMember and
ItemOwnership.

---

## 7. Chats as first-class content

A chat is not just a bag of messages. It has its own node
identity that can be reacted to and ranked:

- `User → Chat` actor edge — sentiment and relevance toward the
  chat itself ("this chat is a great space", "this chat has gone
  toxic").
- `Comment → Chat` structural edge — comment on the chat as a
  whole.
- Chats can appear in feeds alongside posts, ranked like any
  other content.

Recommending a chat to a friend is exactly the same graph
operation as recommending a post: a positive outgoing edge that
the ranking algorithm sees.

**Users control what node types show up in their feed.** Every
node type is different (Post, Comment, Chat, ChatMessage, Item,
…), and users choose via their frontend of choice which ones
appear. A user who only wants posts gets only posts; one who
wants "posts + chats" gets both. This is a general feed-display
feature, not a chat-specific one — chats simply participate in
it alongside every other node type.

---

## 8. ChatMessages as first-class content

Each individual message is also a node — not a row in a table
hidden inside the chat:

- `User → ChatMessage` actor edge — like, dislike, mark
  interesting.
- `Comment → ChatMessage` structural edge — comment on a
  specific message. Without this, pointing at "that wild take
  three messages up" requires prose description; with it, the
  comment links the exact message.
- If the chat is plaintext, a ChatMessage can be interacted with
  from *outside* the chat — it's public content like anything
  else.

The ChatMessage is the atomic unit. The chat is its container.

### Embedding other content

A ChatMessage can carry a **reference** to any other node — Post,
Item, User, Collective, Hashtag, Proposal, another ChatMessage,
a junction node, anything with a graph identity:

```
ChatMessage --[:REFERENCES]--> X
```

See
[edges.md §2 "Reference"](../primitive/edges.md#reference) for
the full catalog. Hashtag is included on this carrier because
ChatMessage has no `:TAGGING` edge type — the per-source
carve-out explanation lives in the same Reference subsection of
edges.md.

The actor's gesture is **authoring a ChatMessage with a target**:
one API call produces the ChatMessage (graph node + Postgres body
row), the author's `User → ChatMessage` actor edge, the
`ChatMessage → Chat :CONTAINMENT` edge, and the
`ChatMessage → X :REFERENCES` edge. The actor does not directly
set a stance on X by sharing — sharing is an authoring event for
the ChatMessage, not an actor edge to X. The referenced node
gains reach through the actor's network via traversal of
`User → ChatMessage → :REFERENCES → X`.

**Reference vs external sharing.** Sharing into a chat creates
graph state (a ChatMessage with `:REFERENCES`), so the
referenced node propagates through the actor's network. External
sharing (link copy, share-to-another-app, export) creates no
graph state — see
[graph-model.md §3](../primitive/graph-model.md#3-edge-categories)
— and so does not amplify reach within CoGra.

### Message edits

Message bodies are **display content**: they live in Postgres
(or a media server for attachments), not in the graph. Edits are
append-only — a correction writes a new version into the
Postgres row rather than overwriting the old one. Past versions
remain readable. See [layers.md](../primitive/layers.md) for the
project-wide append-only rule.

---

## 9. Encryption as the privacy mechanism

The graph carries chat **topology only** — it never holds
message bodies, encrypted or not. **Privacy is per-message, not
per-chat:** each ChatMessage's `chat_messages` row in Postgres
carries the `content_privacy` flag declared in §4.2 that tells
the frontend whether to attempt decryption. A single chat can
mix plaintext and encrypted messages freely.

### Chat keys, organized in epochs

A chat's key is not a single static secret. The lifetime of a
chat is partitioned into a sequence of **epochs** E₁, E₂, …,
each with its own symmetric chat key Kᵢ. The current epoch's
key is the one the frontend uses to encrypt new messages; past
epochs' keys live on, used for decrypting their respective
messages. `Chat.epoch` (§3.1) is the on-graph handle for "which
key is current."

Every membership-change event automatically closes the current
epoch and opens a new one — the system advances `Chat.epoch`:

- A new active member (`Chat → ChatMember` activates) — opens E_{i+1}.
- A member leaves voluntarily — opens E_{i+1}.
- A member-disavowal vote passes (§10 Level 2) — opens E_{i+1}.

The new key itself is derived collaboratively by the
post-change set of members using the underlying group-key
protocol (Signal/MLS-style key update) — implementation detail
of the messenger library, not a graph operation. **Rotation is
automatic and not voted**; otherwise an evicted member could
vote-block their own removal from future epochs, defeating the
point.

**Invariant:** Chat-key rotation on a membership change is
automatic — `Chat.epoch` advances by `1` the moment the
membership transition takes effect, without a separate vote.
"Takes effect" is pinned by topology: a join is the moment a
`:CLAIM` and matching `:APPROVAL` for the ChatMember are both
present with positive top layers; a leave (including
member-disavowal cascade) is the moment an active `:APPROVAL`
gets a `dim1 < 0` layer. Pending claims with no matching
approval, or expired junctions whose negative `:APPROVAL` was
already counted, do not re-trigger.

Concurrent membership transitions on the same Chat — two
approvals or a join racing a leave — serialize per Chat via
the same per-node lock primitive
[governance.md §6 "Tally serialization"](../primitive/governance.md#6-when-outcomes-take-effect)
uses for Proposal tallies. The first transition's `epoch`
write commits; the second runs against the post-commit state
and increments from there.

Mid-epoch rotation (§ "Mid-epoch rotation via Proposal" below)
is the only path that runs through governance; rotation
triggered by joins, leaves, or member-disavowal passes never
does.

This principle — *epoch advances automatically the moment the
membership transition takes effect; only mid-epoch rotation
runs through governance* — could conceivably generalize to any
junction-state-bearing node that wants topology-implied
advancement of a sibling counter. Chat is the only consumer
today, so it stays here as **instance-specific**: not promoted
to primitive until a second consumer surfaces.

Each encrypted ChatMessage's body row in Postgres carries an
`epoch` index pointing at the key it was encrypted under
(§4.2). The graph never reads it; the frontend uses it to pick
the right key.

### What members hold

- **Current members** can derive Kᵢ for the current epoch and
  hold (or can re-derive) the keys for every epoch they were
  active in.
- **A new joiner** receives only Kᵢ for the epoch they joined
  and onward. They do **not** automatically gain access to
  pre-join history; cryptography can't gift them a key they
  weren't entitled to. Reading older history requires an
  existing member to share the older key with them — a normal
  disclosure act.
- **Ex-members** keep the keys for the epochs they were active
  in. Cryptography can't forget those. They cannot derive keys
  for any subsequent epoch — the rotation is the technical
  enforcement of "you can leak what you saw, not what comes
  after."

### Disclosure and irreversibility

Any chat member can disclose any chat key they hold publicly —
through any normal authoring gesture: a Comment on the chat, a
public Post, a plaintext ChatMessage in the same chat, an
off-graph channel, anything. The system permits this by design.
Encryption protects against people *outside* a given epoch
reading content; a participant sharing what was said to them is
a normal social and legal posture the graph doesn't restrict.

Once disclosed, a key cannot be un-disclosed. Every message
encrypted under that epoch's key becomes readable to anyone in
possession of the leaked key. **Disclosure is scoped to the
disclosed epoch only** — leaking Kᵢ exposes E_i's messages but
not any earlier or later epoch.

### Mid-epoch rotation via Proposal

Members may also rotate the chat key **without a membership
change** — for example, after a member's device has been
compromised but before they have left the chat. The mechanism is
the ordinary property-change Proposal flow (§10): a Proposal
targets `Chat.epoch` with `proposed_value = current + 1`. On
threshold-cross, the property advances and current members
re-run the group-key-update procedure off-graph. The thresholds
themselves are the `Chat.rotate_key_quorum` and
`Chat.rotate_key_threshold` properties (§3.1, §10 tables) and
can be changed via Proposals targeting them — governance of
governance applies all the way down.

The advance commits regardless of who is online when the
Proposal passes — graph state is the source of truth, not member
presence. Key derivation is **lazy**: the new key is produced
the first time a current member reads into the new epoch
(decrypting an incoming message or composing one), not at
threshold-cross. An epoch with no usable key is acceptable; it
stays inert until a reader needs it, and the off-graph
group-key-update runs whenever current members next coincide.

**At most one open mid-epoch rotation Proposal per Chat.** The
service layer rejects a new rotation Proposal if an unresolved
one already targets this Chat's `epoch`. The new Proposal's
`proposed_value` is auto-set to the current `epoch + 1` rather
than a user-supplied value — there is no "rotate to epoch X+3"
gesture. Together the two rules close the collision where
Proposal B opens for the next epoch while Proposal A is still
counting votes: B can't open until A resolves.

Mid-epoch rotation is forward protection only, not a privacy
upgrade against prior leakage: messages already encrypted under
the previous key stay readable to anyone who holds it.
Append-only forbids re-encrypting historical content.

### Important properties

- **No layer of the system is a trusted party for decryption.**
  The graph holds no body content at all. The Postgres operator
  holds ciphertext for encrypted messages and never sees any
  key.
- **Privacy is key management, not a graph or database
  feature.** Once a member leaks an epoch's key, nothing in the
  system can prevent others from reading messages from that
  epoch — same as every E2EE system.
- **Metadata is public by design.** The fact that users A and B
  share a chat is a public graph fact. CoGra deliberately
  doesn't hide who talks to whom.

### Key management library

CoGra **does not reinvent crypto**. Key derivation, group-key
update on membership change, and forward secrecy use an
established open-source protocol (e.g. the Signal protocol,
MLS). No custom crypto. Picking the specific library is an
implementation decision, not a design decision.

### Searching and indexing

The graph layer holds no plaintext, so it can't be searched for
chat content. Search on plaintext chats therefore operates on
the **Postgres side**, where message bodies live. This is
transparent by design — anyone with database access (or who can
query the public API) can scan public chats. Cost management at
scale is a performance concern for later, not a design flaw
now.

---

## 10. Moderation

Open public chats face an obvious question: without an admin,
who stops a bad message from dominating? CoGra's answer reuses
the no-push principle from
[graph-model.md §7](../primitive/graph-model.md#7-directionality-inbound-edges-dont-affect-your-graph):

**The chat moves away from a message (or a member). It never
moves the message or the member away.**

Moderation happens at two levels, independently. Both are
instances of the weighted-voting primitive in
[governance.md](../primitive/governance.md), and both route
through a **Proposal** node — same mechanism every other
property-level governance decision uses (chat name, role,
mid-epoch key rotation, platform moderation in
[moderation.md](moderation.md)). Votes travel from the voter's
`ChatMember` junction to the Proposal (Shape B), so the chat
stance stays decoupled from personal sentiment on
`User → ChatMessage` or `User → ChatMember`. Both levels carry
the same Proposal shape: `target_property = 'node'`,
`proposed_value = 'disavowed'` — the `'node'` value is the
whole-node-targeting sentinel defined in
[nodes.md "Whole-node targeting"](../primitive/nodes.md#whole-node-targeting-the-node-sentinel).
What differs between the two levels is the cascade behavior on
threshold-cross, which dispatches on the target's node type.

**Invariant:** Chat-internal disavowal routes through a Proposal
node — both Level 1 (against a `ChatMessage`) and Level 2
(against a `ChatMember`) carry the `target_property = 'node'`,
`proposed_value = 'disavowed'` shape; no direct vote edge from a
`ChatMember` drives a disavowal outcome. This keeps tally
semantics uniform with every other chat governance act and makes
counter-Proposal reversal clean.

### Level 1 — Message disavowal

A Level 1 Proposal targets the offending `ChatMessage`. The
first reporter's authoring is their +1 vote per
[proposal.md §5](proposal.md#5-authorship); subsequent voters
cast `ChatMember → Proposal` Shape B votes on the existing
Proposal rather than authoring duplicates.

A ChatMessage carries no pre-existing approval-style edge from
the chat to layer over (unlike a ChatMember — see Level 2), so
**no separate outcome edge is written.** The chat's current
stance toward the message is derived from the existence of a
passed Level 1 Proposal targeting it: once the Proposal's tally
has crossed threshold, that pass-state is sticky per
[governance.md §6](../primitive/governance.md#6-when-outcomes-take-effect)
— the Proposal itself is the on-graph record, the same way
moderation reports are on-graph as the Proposal rather than as
a separate reports table
([moderation.md §2](moderation.md#2-reports--proposals-on-the-graph)).

The message body is **not** removed — append-only applies. A
reader who wants to see disavowed content still can; a reader
who treats the chat's current stance as authoritative simply
won't. A counter-Proposal targeting the same ChatMessage with
`proposed_value = 'normal'` reverses the disavowal —
governance applies symmetrically in both directions, consistent
with platform moderation's symmetric un-classification path
([governance.md §7](../primitive/governance.md#7-the-mod-gate)).

### Level 2 — Member disavowal

A chat can also move away from a *member*, not just a message.
This is the heavier decision and is a separate governance act —
not an automatic cascade from N message disavowals. A chat may
disavow a member after a pattern of incidents or after a single
severe one; the community decides when to escalate. Past Level
1 disavowals of a member's messages and any open Level 2
Proposals targeting their `ChatMember` are visible signals,
but never automatic triggers.

A Level 2 Proposal targets the member's `ChatMember` junction.
On threshold-cross, the cascade interpreter — seeing target =
`ChatMember`, `target_property = 'node'`,
`proposed_value = 'disavowed'` — writes a new `dim1 < 0` layer
on the existing `Chat → ChatMember` approval edge for the
target. The junction's full edge history stays in the graph;
only the top layer of the approval edge changes. Membership
state remains encoded in the topology per
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows)
— no new property is introduced. A counter-Proposal with
`proposed_value = 'normal'` reverses the disavowal, just like
Level 1.

### Default parameters

Starting points, not fixed rules:

| Parameter       | Message disavowal (Level 1)                                                                              | Member disavowal (Level 2)                                                                              |
|-----------------|----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Eligibility     | Active `ChatMember`s                                                                                     | Active `ChatMember`s excluding the member under review                                                  |
| Role weights    | `admin = 5`, `chat_mod = 3`, `member = 1`                                                                | Same                                                                                                    |
| Quorum          | ≥ 20% of total eligible weight has cast a vote                                                           | ≥ 40% of total eligible weight has cast a vote                                                          |
| Threshold       | > 50% of cast weight disavowing                                                                          | ≥ 2/3 of cast weight disavowing                                                                         |
| Proposal target | `:Proposal → :ChatMessage`, `target_property = 'node'`, `proposed_value = 'disavowed'`                   | `:Proposal → :ChatMember`, `target_property = 'node'`, `proposed_value = 'disavowed'`                   |
| Outcome         | No separate outcome edge; the chat's stance is the existence of the passed Proposal targeting the message | Cascade writes a new `dim1 < 0` layer on the `Chat → ChatMember` approval edge for the target          |
| Takes effect at | New-vote threshold-crossing ([governance.md §6](../primitive/governance.md#6-when-outcomes-take-effect)) | Same                                                                                                    |

**Every number above is a node property on the `Chat`** (§3.1).
Role weights, quorum %, threshold % — none of them are
hardcoded. Members change any of them via a Proposal node
targeting the chat's property, voted by the same eligibility
rules (see
[governance.md §2.1](../primitive/governance.md#21-subject)).

### How roles fit in

Roles (`admin`, `chat_mod`, `member`) are carried as the `role`
property on the `ChatMember` junction node (§3.3). The role-weights
table above is the default derivation; a `ChatMember` may also
carry an optional `voting_weight` property that sets per-member
weight directly, overriding the role-based derivation at tally
time.

An admin's disavowal weight is higher than a member's but it
is never a veto — in any chat large enough that an admin's
weight is a small fraction of the pool, an admin cannot
single-handedly disavow. Multiple admins simply stack their
weights under the same primitive; no separate M-of-N admin
rule is needed. A community can override an admin by crossing
the threshold without the admin's participation.

### Property and role changes via Proposals

The chat's other state changes use the same Proposal
mechanism as disavowal. `ChatMember.role` (promote / demote),
`Chat.name`, `Chat.join_policy`, and `Chat.epoch` (mid-epoch
key rotation, see §9) are all node properties; each change is
a Proposal voted on by chat members under chat-defined
parameters.

Suggested defaults (starting points, not fixed rules):

| Property change                    | Quorum | Threshold | Eligibility                                    |
|------------------------------------|--------|-----------|------------------------------------------------|
| `ChatMember.role`                  | ≥ 30%  | > 50%     | Active members, excluding the subject member   |
| `Chat.name`                        | ≥ 10%  | > 50%     | Active members                                 |
| `Chat.join_policy`                 | ≥ 30%  | ≥ 2/3     | Active members                                 |
| `Chat.epoch` (mid-epoch rotation)  | ≥ 50%  | ≥ 2/3     | Active members                                 |
| Disavowal thresholds (above)       | ≥ 30%  | ≥ 2/3     | Active members                                 |

These percentages are themselves node properties on the chat
(§3.1) and can be changed via Proposals targeting them —
governance of governance applies all the way down. Promoting
and demoting exclude the subject from voting (consistent with
the member-disavowal exclusion); cosmetic changes like the
title don't.

### Still no push

Even this flow is pull, not push. The chat moves away; nothing
forces the content or the member off the graph. Anyone reading
the graph directly still sees the disavowed message or the
departed member — they just see that the chat has moved away.

### Coexistence with platform moderation

A `ChatMessage` can be simultaneously subject to two governance
instances at different scopes — chat-internal disavowal
(this section) and the Network-scope platform moderation in
[moderation.md](moderation.md). They write to different state
and produce independent outcomes: chat-side stance vs.
node-property classification or per-field redaction. The
primitive treatment of this — why no conflict arises, and how
the pattern generalizes — lives in
[governance.md §9](../primitive/governance.md#9-coexistence-multiple-governance-instances-on-a-shared-subject).

---

## 11. Joining and leaving a chat

After the founder's bootstrap (§2.1), every ChatMember is
created via the **two-edge approval pattern** from
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows),
which combines two voting shapes
([governance.md §3](../primitive/governance.md#3-the-two-vote-shapes)):

- The **would-be member's Shape A self-claim** — their
  `User/Collective → new ChatMember` actor edge. Necessary
  because they have no ChatMember of this chat yet from which
  to cast a Shape B vote.
- **Zero or more Shape B approver votes** from existing
  ChatMembers — `ChatMember_approver → ChatMember_new` (`dim1 > 0`).
  The number required is `entry_approval_required_count` (§3.1);
  votes only count when the approver's `role` is listed in
  `entry_approval_eligible_roles`.

When the policy is satisfied the system writes the
`Chat → ChatMember` approval edge, and the membership is
active.

A ChatMember junction is bound to its bearer at creation by a
`ChatMember → User/Collective` `:BEARER` structural edge — see
[edges.md §2 "Bearer binding"](../primitive/edges.md#bearer-binding)
and
[graph-data-model.md `:ChatMember`](../implementation/graph-data-model.md#chatmember).
The Shape A self-claim that activates the membership must come
from the actor at the other end of `:BEARER`; mismatched claims
are rejected. The bearer can be a User or a Collective —
Collectives can be ChatMembers on the same footing as Users,
acting through their authorized members per
[collectives.md §2](collectives.md#2-acting-through-the-collective).
"What invites are pending toward me?" is then a single inbound
traversal from the User/Collective node along `:BEARER`.

**Every member's membership has the same shape:**
`User/Collective → ChatMember → Chat` (their own self-claim
plus the system-created claim edge). The only difference
between members is what's *also* pointing at their ChatMember:
non-founder members have approver Shape B edges from existing
ChatMembers; the founder has none, because they were the only
required vote.

**Why approvers vote Shape B and not Shape A.** A Shape A
approval (User/Collective → ChatMember) would persist
meaninglessly if the approver later left the chat — their
personal endorsement of someone's membership shouldn't outlive
their own membership. Shape B from the approver's `ChatMember`
junction ties the admission vote to their current membership
state: if the approver's own junction goes revoked, their
admission vote drops from any future tally per
[governance.md §2.2](../primitive/governance.md#22-eligibility).
The same Shape B carrier supports stance flips during the open
admission period — an approver can change their mind before
the threshold is crossed by appending a new layer to their
existing edge. Once a membership is active, disavowal flows
through a Proposal (§10 Level 2), not through a re-layering of
this admission edge.

### Open

The would-be member writes their `User/Collective → ChatMember`
Shape A self-claim. The system creates the `ChatMember → Chat`
claim edge. Because no Shape B approver vote is required, the
system immediately writes the `Chat → ChatMember` approval
edge. The membership is active.

### Invite-only

The **inviter** — an existing member whose `role` is listed in
`invite_proposer_roles` (§3.1) —
casts a `ChatMember_inviter → ChatMember_new` **Shape B vote**
(`dim1 > 0`) toward a new junction node. At the same time the
system writes the `ChatMember → invitee` `:BEARER` edge,
binding the junction to the prospective bearer, and the
`ChatMember → Chat` claim edge. The membership is pending: the
junction exists with the inviter's approval and a known bearer,
but the invitee has not yet self-claimed.

The **invitee** later writes their own
`User/Collective → ChatMember` **Shape A self-claim** to that
same junction; the API rejects the claim if it doesn't come
from the actor at the other end of `:BEARER`. Both required
edges (the inviter's Shape B approval and the invitee's
matching Shape A self-claim) are now present; the system writes
the `Chat → ChatMember` approval edge. The membership is active.

If the invitee never self-claims, the membership persists
pending; the junction node is never deleted.

### Request-entry

The **would-be member** writes their
`User/Collective → ChatMember` Shape A self-claim. The system
creates the `ChatMember → Chat` claim edge. The membership is
pending.

An existing ChatMember whose `role` is listed in
`entry_approval_eligible_roles` (§3.1) — typically `'admin'` or
`'chat_mod'` by default — casts a
`ChatMember_approver → ChatMember_new` **Shape B vote**
(`dim1 > 0`). The system writes the `Chat → ChatMember`
approval edge. The membership is active.

### Higher N (a.k.a. multi-sig)

Any of the variants above can require **multiple Shape B
approver votes** instead of one — e.g., "two admins must
approve before the membership activates." Same primitive, just
a higher `entry_approval_required_count` (§3.1) drawn from the
chat's policy, with `entry_approval_eligible_roles` continuing
to gate which approvers count. The junction stays pending until
the Nth qualifying Shape B vote crosses the threshold.
"Multi-sig" is a label for this configuration shape, not a
fourth flow.

### State encoding

A membership is **pending** when only the claim edge exists;
**active** when both claim and approval edges exist;
**revoked** when a `dim1 < 0` layer has been
appended to either edge. Per
[graph-model.md §5](../primitive/graph-model.md#5-junction-node-flows),
nothing is deleted — state transitions are encoded as new
layers on the existing structural edges.

### Leaving and removal

- **Voluntary leave** — the member writes a negative-`dim1`
  layer on their own `User/Collective → ChatMember` Shape A
  self-claim. The system appends a `dim1 < 0` layer on the
  **claim-side** structural edge.
- **Member disavowal** — eligible voters per §10 Level 2 cast
  `ChatMember → Proposal` Shape B votes on a Proposal targeting
  the member's `ChatMember` junction (`target_property = 'node'`,
  `proposed_value = 'disavowed'`). When the threshold is crossed,
  the cascade appends a `dim1 < 0` layer on the **approval-side**
  `Chat → ChatMember` structural edge for the target. The
  admission-time `ChatMember → ChatMember` edges remain on the
  graph in their original state — they are not re-layered by
  disavowal. There is no admin-unilateral disavowal; admins
  participate in the disavowal vote with their role-weighted
  vote alongside everyone else.

The ChatMember junction and its full edge history stay in the
graph; only the top layers change.

### Key rotation on membership change

Every transition into or out of active membership — new active
member, voluntary leave, member-disavowal pass — also triggers
the automatic chat-key rotation described in §9: `Chat.epoch`
advances by `1` and the new member set re-runs the
group-key-update procedure off-graph. A leaving member keeps
the keys for past epochs they were active in but cannot derive
the new one.

---

## 12. 1:1 vs group chats

There is **no structural difference**. A 1:1 chat is a chat
with exactly two members; a group chat is a chat with three or
more. The same node types, the same edges, the same flows.

**Invariant:** No structural 1:1 chat uniqueness. Two users may
have any number of parallel 1:1 chats; the graph imposes no
uniqueness constraint over `(actor_a, actor_b)` member pairs.
Uniformity over special-casing.

In particular, CoGra **does not enforce** that two users can
have at most one 1:1 chat with each other. Reasons:

- Enforcement adds a special case without preventing real abuse
  (three users can already have an unlimited number of group
  chats — 1:1 uniqueness wouldn't change that).
- Legitimate multi-chat exists (work + personal, public +
  private, different topics).
- Frontends can surface UX hints ("you already have a chat with
  Alice — open it?") without the graph layer forcing a single
  thread.

Uniformity over special-casing. The graph stays simple; the
frontend handles the UX questions.

---

## 13. Lifecycle

All three node types are **never deleted**. Per
[layers.md §5](../primitive/layers.md#5-deletion-policy), the
only permitted "removal" is in-place layer redaction on graph
properties or a tombstone version row on Postgres-side display
content; both preserve a visible record that the change
occurred.

### 13.1 Chat

`'illegal'`-classification target fields on a Chat: `name`
(graph-side layer redaction), `description` (Postgres tombstone
version row), the chat image (media tombstone + asset removal,
targeted via `image_id`), or the `'full'` shorthand per
[moderation.md §5](moderation.md#5-scope). A passing Proposal
fires the redaction cascade and auto-flips `moderation_status`
to `'illegal'`. The cascade does **not** propagate to the
chat's ChatMessages, ChatMembers, or any content node that
references the chat.

`'sensitive'` classification is a top-layer flip on
`moderation_status` only; no redaction, reversible by
counter-Proposal.

**Account deletion of the founder** does not affect the chat:
identity-level deletion redacts the founder's PII but the User
node persists, and content-level deletion does **not** sweep
up Chats (a chat is a public space, not first-person
expression per
[account-deletion.md §1](account-deletion.md#1-two-redaction-levels)).

### 13.2 ChatMessage

`'illegal'`-classification target fields on a ChatMessage:
`content` (Postgres tombstone version row), `attachments`
(media tombstone + asset removal), or the `'full'` shorthand.
The cascade applies regardless of whether the message is
plaintext or encrypted; encrypted messages are classifiable
once the relevant epoch key has been voluntarily disclosed per
§9 and
[moderation.md "Encrypted message classification"](moderation.md#encrypted-message-classification).

Chat-internal disavowal (§10 Level 1) is non-destructive — the
message body stays; the chat's stance moves away.

**Account deletion of the author** triggers content-level
redaction of the author's ChatMessages **only if** the author
opted in to content-level deletion per
[account-deletion.md §1](account-deletion.md#1-two-redaction-levels);
otherwise only the author's PII is redacted on the User node
and the message body persists.

### 13.3 ChatMember

ChatMember has no user-input fields and therefore no per-field
redaction triggers. Membership state transitions follow §11
"Leaving and removal" — new layers on the two-edge approval
pair, prior layers preserved.

**Account deletion** of the underlying User or Collective does
not remove the ChatMember; the actor node persists with
redacted PII and the junction continues to point at it.

---

## What this doc is not

- **Not the edge catalog.** Per-target-type edges with
  dimension labels live in [edges.md](../primitive/edges.md).
- **Not the governance primitive.** The Proposal mechanism, the
  weighted-voting shape, the two vote shapes (Shape A / Shape
  B), and threshold-policy mechanics live in
  [governance.md](../primitive/governance.md).
- **Not the moderation primitive.** The Network-level
  moderation instance — eligibility, mod gate,
  illegal-classification cascade, and per-node targetable-field
  set — lives in [moderation.md](moderation.md). This doc
  covers only the chat-internal disavowal instance (§10) and
  how it composes with platform moderation.
- **Not the deletion mechanism.** The redaction primitive
  lives in
  [layers.md §5](../primitive/layers.md#5-deletion-policy);
  the per-row legal hold and archive disposition live in
  [retention-archive.md](../primitive/retention-archive.md);
  the account-deletion flow lives in
  [account-deletion.md](account-deletion.md).
- **Not the Memgraph or Postgres schema.** Concrete property
  types, columns, indexes, and the `chats` / `chat_messages` /
  `chat_message_attachments` / `media_attachments` shapes live
  in
  [graph-data-model.md](../implementation/graph-data-model.md)
  and [data-model.md](../implementation/data-model.md).
- **Not the encryption protocol.** Key derivation, group-key
  update on membership change, and forward secrecy use an
  established open-source protocol; CoGra does not reinvent
  crypto (§9 "Key management library").
