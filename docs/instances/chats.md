# Chats

Chats on CoGra are **not** what they are on WhatsApp, Signal, or iMessage.
This doc exists because assuming otherwise leads to wrong designs.

---

## 1. Mental model reset

In most messaging apps a chat is a **private, hidden space**. Membership
is invisible to outsiders; content is the only privacy layer, via
end-to-end encryption. The conversation effectively does not exist from
the outside.

In CoGra, a chat is a **public node on the graph**. Its existence, its
member list, and its message count are visible to every actor on the
graph (see the transparency principle in
[graph-model.md §1](../primitive/graph-model.md)). **Topology is always
public**; what's private is the *content* of individual messages, if
the chat chose to encrypt them.

Chats and ChatMessages are **first-class interactable nodes**. Users
can like them, comment on them, and rank them in feeds — just like
posts.

This feels wrong if you map "chat" onto "group DM." It feels natural
if you map "chat" onto "public discussion space that happens to have
members, some of which may choose to run with encrypted content."

---

## 2. The two orthogonal axes

A chat's behavior is defined by two independent choices:

### Join policy — who can become a member

Specified in [graph-model.md §5](../primitive/graph-model.md) as the
two-edge approval pattern. Four shapes:

- **Open** — anyone joins, no approval required.
- **Invite-only** — existing member/admin invites, invitee accepts.
- **Request-entry** — user requests, admin approves.
- **Multi-sig** — N approvers required (governance-heavy chats).

### Content privacy — who can read messages

- **Plaintext** — ChatMessage payloads are stored as readable text.
  Anyone walking the graph can read them.
- **End-to-end encrypted (E2EE)** — ChatMessage payloads are
  ciphertext. Only members holding the decryption key can read. The
  graph layer never sees plaintext.

The two axes are independent. Every combination is valid:

| Join × privacy | Plaintext                                 | E2EE                                                  |
|----------------|-------------------------------------------|-------------------------------------------------------|
| Open           | Public forum (anyone joins, anyone reads) | Open group with private content                       |
| Invite-only    | Members-only forum                        | Classic private group; 1:1 DM is this with 2 members  |
| Request-entry  | Request-only forum                        | Gated private community                               |

---

## 3. Chats as first-class content

A chat is not just a bag of messages. It has its own node identity that
can be reacted to and ranked:

- `User -> Chat` actor edge — sentiment and relevance toward the chat
  itself ("this chat is a great space", "this chat has gone toxic").
- `Comment -> Chat` structural edge — comment on the chat as a whole.
- Chats can appear in feeds alongside posts, ranked like any other
  content.

Recommending a chat to a friend is exactly the same graph operation as
recommending a post: a positive outgoing edge that the ranking
algorithm sees.

**Users control what node types show up in their feed.** Every node
type is different (Post, Comment, Chat, ChatMessage, Item, …), and
users choose via their frontend of choice which ones appear. A user who
only wants posts gets only posts; one who wants "posts + chats" gets
both. This is a general feed-display feature, not a chat-specific one —
chats simply participate in it alongside every other node type.

---

## 4. ChatMessages as first-class content

Each individual message is also a node — not a row in a table hidden
inside the chat:

- `User -> ChatMessage` actor edge — like, dislike, mark interesting.
- `Comment -> ChatMessage` structural edge — comment on a specific
  message. Without this, pointing at "that wild take three messages up"
  requires prose description; with it, the comment links the exact
  message.
- If the chat is plaintext, a ChatMessage can be interacted with from
  *outside* the chat — it's public content like anything else.

The ChatMessage is the atomic unit. The chat is its container.

### Message edits

Message bodies are **display content**: they live in Postgres (or a
media server for attachments), not in the graph. The graph has the
ChatMessage node (authorship edge, timestamp, structural edge to the
Chat) but not the text itself.

Edits are append-only. A correction writes a new version into the
Postgres row rather than overwriting the old one. Past versions remain
readable. This is consistent with the project-wide append-only
principle: you cannot retroactively erase what you wrote, you can only
add what you meant to write. See [layers.md](../primitive/layers.md) for the
project-wide append-only rule across edges, node properties, and
Postgres-side display content.

---

## 5. Encryption as the privacy mechanism

For an E2EE chat, the ChatMessage payload stored on the graph is a
**ciphertext blob**. The graph layer — and anyone reading the graph
from outside — sees:

- That the ChatMessage exists.
- Its author (see [authorship.md](../primitive/authorship.md)).
- Its creation timestamp.
- Its structural position (`ChatMessage -> Chat`).
- A ciphertext blob as the payload.

They do **not** see the plaintext. Decryption requires the per-chat
symmetric key held by members.

Important to be explicit about:

- **The graph layer is never a trusted party for decryption.** It has
  no access to plaintext at any point.
- **Privacy is key management, not a graph feature.** If a member
  leaks the key or forwards decrypted content, nothing in the graph can
  prevent it — same as every E2EE system.
- **Metadata is public by design.** The fact that users A and B share
  a chat is a public graph fact. This is a deliberate departure from
  apps that try to hide who talks to whom.

### Key management

CoGra **does not reinvent crypto**. Key exchange, per-member key
rotation, and forward secrecy use an established open-source protocol
(e.g. the Signal protocol, MLS). No custom crypto. Picking the specific
library is an implementation decision, not a design decision.

### Searching and indexing

The graph layer holds no plaintext, so it can't be searched for chat
content. Search on plaintext chats therefore operates on the **Postgres
side**, where message bodies live. This is transparent by design —
anyone with database access (or who can query the public API) can scan
public chats.

That is potentially expensive at scale. Cost management is a
performance concern for later, not a design flaw now. If a cheap
"don't index" signal turns out to be needed, it can be added as a
property without changing the graph model.

---

## 6. Moderation

Open public chats face an obvious question: without an admin, who
stops a bad message from dominating? CoGra's answer reuses the
no-push principle from [graph-model.md §7](../primitive/graph-model.md):

**The chat moves away from a message (or a member). It never moves
the message or the member away.**

Moderation happens at two levels, independently. Both are instances
of the weighted-voting primitive in [governance.md](../primitive/governance.md),
both use Shape B (the vote travels from the voter's `ChatMember`
junction to the subject, so the chat stance stays decoupled from
personal sentiment).

### Level 1 — Message disavowal (`Chat -> ChatMessage`)

Members vote to disavow a specific `ChatMessage`. If the vote
passes, a new layer on the `Chat -> ChatMessage` structural edge
signals that the chat no longer associates itself with the message.
The message is **not** removed — append-only applies. A reader who
wants to see disavowed content still can; a reader who treats the
chat's current stance as authoritative simply won't.

### Level 2 — Member disavowal (`Chat -> ChatMember`)

A chat can also move away from a *member*, not just a message.
This is the heavier decision and is a separate governance act —
not an automatic cascade from N message disavowals. A chat may
disavow a member after a pattern of incidents or after a single
severe one; the community decides when to escalate. The count of
incoming `disavow` edges on a `ChatMember` is a visible signal,
but never a trigger.

If the vote passes, a new layer on the `Chat -> ChatMember`
structural edge reflects that the chat no longer accepts the
member. The full membership history stays in the graph; only the
current stance changes.

### Default parameters

Starting points, not fixed rules:

| Parameter       | Message disavowal                                                     | Member disavowal                                                      |
|-----------------|-----------------------------------------------------------------------|-----------------------------------------------------------------------|
| Eligibility     | Active `ChatMember`s                                                  | Active `ChatMember`s excluding the member under review                |
| Role weights    | `admin = 5`, `mod = 3`, `member = 1`                                  | Same                                                                  |
| Quorum          | ≥ 20% of total eligible weight has cast a vote                        | ≥ 40% of total eligible weight has cast a vote                        |
| Threshold       | > 50% of cast weight disavowing                                       | ≥ 2/3 of cast weight disavowing                                       |
| Outcome         | New layer on `Chat -> ChatMessage`                                    | New layer on `Chat -> ChatMember`                                     |
| Takes effect at | New-vote threshold-crossing ([governance.md §6](../primitive/governance.md))       | Same                                                                  |

**Every number above is a node property on the `Chat`.** Role
weights, quorum %, threshold % — none of them are hardcoded.
Members change any of them via a Proposal node targeting the
chat's property, voted by the same eligibility rules (see
[governance.md §2.1](../primitive/governance.md)). The defaults above exist
to bootstrap new chats; they are not fixed rules, and the same
primitive that governs disavowals also governs changes to the
disavowal rules themselves.

### How roles fit in

Roles (admin, mod, member) are carried as properties on the
`ChatMember` junction node (see [graph-model.md §2](../primitive/graph-model.md)).
An admin's disavowal weight is higher than a member's but it is
never a veto — in any chat large enough that an admin's weight is
a small fraction of the pool, an admin cannot single-handedly
disavow. Multiple admins simply stack their weights under the same
primitive; no separate M-of-N admin rule is needed.

A community can override an admin by crossing the threshold
without the admin's participation. That falls naturally out of
the primitive, not from a special rule.

### Property and role changes via Proposals

Beyond disavowal, the chat's other state changes use the Proposal
mechanism (see [governance.md §2.1](../primitive/governance.md)). `ChatMember.role`
(promote / demote), `Chat.name`, `Chat.content_privacy`, and
`Chat.join_policy` are all node properties; each change is a
Proposal voted on by chat members under chat-defined parameters.

Suggested defaults (starting points, not fixed rules):

| Property change                    | Quorum | Threshold | Eligibility                                    |
|------------------------------------|--------|-----------|------------------------------------------------|
| `ChatMember.role`                  | ≥ 30%  | > 50%     | Active members, excluding the subject member   |
| `Chat.name`                        | ≥ 10%  | > 50%     | Active members                                 |
| `Chat.content_privacy`             | ≥ 50%  | ≥ 2/3     | Active members                                 |
| `Chat.join_policy`                 | ≥ 30%  | ≥ 2/3     | Active members                                 |
| Disavowal thresholds (the table above) | ≥ 30% | ≥ 2/3 | Active members                                 |

These percentages are themselves node properties on the chat and
can be changed via Proposals targeting them — governance of
governance applies all the way down. Promoting and demoting
exclude the subject from voting (consistent with the member-disavowal
exclusion); cosmetic changes like the title don't.

### Still no push

Even this flow is pull, not push. The chat moves away; nothing
forces the content or the member off the graph. Anyone reading the
graph directly still sees the disavowed message or the departed
member — they just see that the chat has moved away.

---

## 7. Join flows

*(This content was moved from [graph-model.md §5](../primitive/graph-model.md).
The generic two-edge approval pattern these flows instantiate remains
there as a graph-level primitive.)*

### Open chat
1. User creates an actor edge toward a new **ChatMember** node.
2. System creates `ChatMember -> Chat` (claim).
3. Policy satisfied immediately; system creates `Chat -> ChatMember`
   (approval).
4. User is an active member.

### Invite-only chat
1. Admin (or member with invite rights) creates an actor edge toward a
   new **ChatMember** node for the invitee.
2. System creates `ChatMember -> Chat` (claim, pending).
3. Invitee creates an actor edge toward the same ChatMember node
   (accepting).
4. Policy satisfied; system creates `Chat -> ChatMember` (approval).
5. User is an active member.

### Request-entry chat
1. User creates an actor edge toward a new **ChatMember** node
   (request).
2. System creates `ChatMember -> Chat` (claim, pending).
3. An admin creates an actor edge toward the same ChatMember node
   (approving).
4. Policy satisfied; system creates `Chat -> ChatMember` (approval).
5. User is an active member.

Invite-only and request-entry are **topologically identical** — only
who initiates differs. Multi-sig variants are just a higher N in the
approval policy.

---

## 8. Membership lifecycle

Membership is encoded in the two-edge approval pattern:

- Only the claim (`ChatMember -> Chat`) exists → pending.
- Both edges exist → active.

### Departures (leaving, kicked, revoked)

Append-only means the existing approval edge cannot be removed once
created. State transitions are instead **encoded as new layers on the
structural edges themselves** — the formal rule is in
[graph-model.md §5](../primitive/graph-model.md) ("Revocation and
state transitions"). For a chat membership:

- **Voluntary leave.** The user adds a new layer on their actor edge
  toward the ChatMember (negative sentiment = withdrawing their claim)
  and the system adds a new layer on `ChatMember -> Chat` reflecting
  the retraction. The top layers now disagree with the membership
  being active. Self-determined; no governance vote.
- **Removed by community.** Members vote to disavow the ChatMember
  via the member-disavowal instance defined in §6 — Shape B vote
  with the chat's configured quorum and threshold. When the
  threshold is crossed, the system adds a new layer on
  `Chat -> ChatMember` reflecting that the chat no longer accepts
  the member. There is no admin-unilateral kick path; admins
  participate in the disavowal vote with their role-weighted vote
  alongside everyone else.

In both cases, the relationship is active iff both edges' top layers
have `dim1 > 0`. The full history — including the moment of
departure and the votes that drove it — stays visible, as
everywhere else in the graph.

---

## 9. 1:1 vs group chats

There is **no structural difference**. A 1:1 chat is a chat with
exactly two members; a group chat is a chat with three or more. The
same node types, the same edges, the same flows.

In particular, CoGra **does not enforce** that two users can have at
most one 1:1 chat with each other. Reasons:

- Enforcement adds a special case without preventing real abuse
  (three users can already have an unlimited number of group chats —
  1:1 uniqueness wouldn't change that).
- Legitimate multi-chat exists (work + personal, public + private,
  different topics).
- Frontends can surface UX hints ("you already have a chat with
  Alice — open it?") without the graph layer forcing a single thread.

Uniformity over special-casing. The graph stays simple; the frontend
handles the UX questions.
