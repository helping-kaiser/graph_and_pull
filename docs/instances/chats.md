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

## 2. Join policy — who can become a member

Specified in [graph-model.md §5](../primitive/graph-model.md) as the
two-edge approval pattern. Four shapes:

- **Open** — anyone joins, no approval required.
- **Invite-only** — existing member/admin invites, invitee accepts.
- **Request-entry** — user requests, admin approves.
- **Multi-sig** — N approvers required (governance-heavy chats).

Privacy of message contents is **per-message**, not per-chat — see
§5. A single chat can carry both plaintext and encrypted messages
freely.

---

## 3. Chats as first-class content

A chat is not just a bag of messages. It has its own node identity that
can be reacted to and ranked:

- `User → Chat` actor edge — sentiment and relevance toward the chat
  itself ("this chat is a great space", "this chat has gone toxic").
- `Comment → Chat` structural edge — comment on the chat as a whole.
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

- `User → ChatMessage` actor edge — like, dislike, mark interesting.
- `Comment → ChatMessage` structural edge — comment on a specific
  message. Without this, pointing at "that wild take three messages up"
  requires prose description; with it, the comment links the exact
  message.
- If the chat is plaintext, a ChatMessage can be interacted with from
  *outside* the chat — it's public content like anything else.

The ChatMessage is the atomic unit. The chat is its container.

### Embedding other content

A ChatMessage can carry a **reference** to any other node — Post,
Item, User, Collective, Hashtag, Proposal, another ChatMessage, a
junction node, anything with a graph identity. The graph encodes
this as

```
ChatMessage --[:REFERENCES]--> X
```

See [edges.md §2 Reference](../primitive/edges.md) for the full
catalog.

The actor's gesture is **authoring a ChatMessage with a target**:
one API call produces the ChatMessage (graph node + Postgres body
row with optional caption text), the author's
`User → ChatMessage` actor edge, the
`ChatMessage → Chat :CONTAINMENT` edge, and the
`ChatMessage → X :REFERENCES` edge. The actor does not directly set
a stance on X by sharing — sharing is an authoring event for the
ChatMessage, not an actor edge to X. The referenced node gains
reach through the actor's network via traversal of
`User → ChatMessage → :REFERENCES → X`.

**Reference vs external sharing.** Sharing into a chat creates
graph state (a ChatMessage with `:REFERENCES`), so the referenced
node propagates through the actor's network via path traversal.
External sharing (link copy, share-to-another-app, export) creates
no graph state — see
[graph-model.md §3](../primitive/graph-model.md) — and so does not
amplify reach within CoGra.

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

The graph carries chat **topology only** — it never holds message
bodies, encrypted or not. Per §4, every body lives in Postgres.
**Privacy is per-message, not per-chat:** each ChatMessage carries
a `content_privacy` flag (`plaintext` / `encrypted`) that tells
the frontend whether to attempt decryption. A single chat can mix
plaintext and encrypted messages freely.

For an encrypted message, the body row in Postgres is a
**ciphertext blob**; for a plaintext message it's readable text.

### Chat keys, organized in epochs

A chat's key is not a single static secret. The lifetime of a
chat is partitioned into a sequence of **epochs** E₁, E₂, …,
each with its own symmetric chat key Kᵢ. The current epoch's key
is the one the frontend uses to encrypt new messages; past
epochs' keys live on, used for decrypting their respective
messages. The chat carries an integer property `Chat.epoch`
(default `1`) that advances by `1` on every rotation; it is
the on-graph handle for "which key is current."

Every membership-change event automatically closes the current
epoch and opens a new one — the system advances `Chat.epoch`:

- A new active member (`Chat → ChatMember` activates) — opens E_{i+1}.
- A member leaves voluntarily — opens E_{i+1}.
- A member-disavowal vote passes (§6 Level 2) — opens E_{i+1}.

The new key itself is derived collaboratively by the post-change
set of members using the underlying group-key protocol
(Signal/MLS-style key update) — implementation detail of the
messenger library, not a graph operation. **Rotation is automatic
and not voted**; otherwise an evicted member could vote-block
their own removal from future epochs, defeating the point.

Each encrypted ChatMessage's body row in Postgres carries an
`epoch` index pointing at the key it was encrypted under. The
graph never reads it; the frontend uses it to pick the right key.

### What members hold

- **Current members** can derive Kᵢ for the current epoch and
  hold (or can re-derive) the keys for every epoch they were
  active in.
- **A new joiner** receives only Kᵢ for the epoch they joined and
  onward. They do **not** automatically gain access to pre-join
  history; cryptography can't gift them a key they weren't
  entitled to. Reading older history requires an existing member
  to share the older key with them — a normal disclosure act.
- **Ex-members** keep the keys for the epochs they were active
  in. Cryptography can't forget those. They cannot derive keys
  for any subsequent epoch — the rotation is the technical
  enforcement of "you can leak what you saw, not what comes
  after."

An observer's view:

- **Graph:** the ChatMessage exists, its author (see
  [authorship.md](../primitive/authorship.md)), its creation
  timestamp, its structural position (`ChatMessage → Chat`).
- **Postgres:** the body row — ciphertext + `epoch` index if
  `content_privacy = 'encrypted'`, plaintext otherwise.
- **Non-members** see only what the graph and Postgres expose —
  plaintext bodies are readable to anyone with API access;
  encrypted bodies appear as ciphertext.

### Disclosure and irreversibility

Any chat member can disclose any chat key they hold publicly —
through any normal authoring gesture: a Comment on the chat, a
public Post, a plaintext ChatMessage in the same chat, an
off-graph channel, anything. The system permits this by design.
Encryption protects against people *outside* a given epoch
reading content; a participant sharing what was said to them is a
normal social and legal posture the graph doesn't restrict.

Once disclosed, a key cannot be un-disclosed. Every message
encrypted under that epoch's key becomes readable to anyone in
possession of the leaked key. **Disclosure is scoped to the
disclosed epoch only** — leaking Kᵢ exposes E_i's messages but
does not expose any earlier or later epoch.

### Mid-epoch rotation via Proposal

Members may also rotate the chat key **without a membership
change** — for example, after a member's device has been
compromised but before they have left the chat. The mechanism is
the ordinary property-change Proposal flow (§6, [governance.md
§2.1](../primitive/governance.md)): a Proposal targets
`Chat.epoch` with `proposed_value = current + 1`. On
threshold-cross, the property advances and current members re-run
the group-key-update procedure off-graph. No new mechanism is
introduced — `Chat.epoch` is just another node property like
`Chat.name` or `Chat.join_policy`.

Suggested defaults (starting points, not fixed rules):

| Parameter   | Mid-epoch rotation                |
|-------------|-----------------------------------|
| Eligibility | Active `ChatMember`s              |
| Quorum      | ≥ 50%                             |
| Threshold   | ≥ 2/3 of cast weight in favor     |

These percentages are themselves node properties on the chat
(`Chat.rotate_key_quorum`, `Chat.rotate_key_threshold`) and can be
changed via Proposals targeting them — governance of governance
applies all the way down, same as everywhere else in §6.

Mid-epoch rotation is forward protection only, not a privacy
upgrade against prior leakage: messages already encrypted under
the previous key stay readable to anyone who holds it.
Append-only forbids re-encrypting historical content.

### Important properties

- **No layer of the system is a trusted party for decryption.**
  The graph holds no body content at all. The Postgres operator
  holds ciphertext for encrypted messages and never sees any key.
- **Privacy is key management, not a graph or database feature.**
  Once a member leaks an epoch's key, nothing in the system can
  prevent others from reading messages from that epoch — same
  as every E2EE system.
- **Metadata is public by design.** The fact that users A and B
  share a chat is a public graph fact. CoGra deliberately doesn't
  hide who talks to whom.

### Key management library

CoGra **does not reinvent crypto**. Key derivation, group-key
update on membership change, forward secrecy use an established
open-source protocol (e.g. the Signal protocol, MLS). No custom
crypto. Picking the specific library is an implementation
decision, not a design decision.

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

### Level 1 — Message disavowal (`Chat → ChatMessage`)

Members vote to disavow a specific `ChatMessage`. If the vote
passes, a new layer on the `Chat → ChatMessage` structural edge
signals that the chat no longer associates itself with the message.
The message is **not** removed — append-only applies. A reader who
wants to see disavowed content still can; a reader who treats the
chat's current stance as authoritative simply won't.

### Level 2 — Member disavowal (`Chat → ChatMember`)

A chat can also move away from a *member*, not just a message.
This is the heavier decision and is a separate governance act —
not an automatic cascade from N message disavowals. A chat may
disavow a member after a pattern of incidents or after a single
severe one; the community decides when to escalate. The count of
incoming `disavow` edges on a `ChatMember` is a visible signal,
but never a trigger.

If the vote passes, a new layer on the `Chat → ChatMember`
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
| Outcome         | New layer on `Chat → ChatMessage`                                    | New layer on `Chat → ChatMember`                                     |
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
(promote / demote), `Chat.name`, `Chat.join_policy`, and `Chat.epoch`
(mid-epoch key rotation, see §5) are all node properties; each change
is a Proposal voted on by chat members under chat-defined parameters.

Suggested defaults (starting points, not fixed rules):

| Property change                    | Quorum | Threshold | Eligibility                                    |
|------------------------------------|--------|-----------|------------------------------------------------|
| `ChatMember.role`                  | ≥ 30%  | > 50%     | Active members, excluding the subject member   |
| `Chat.name`                        | ≥ 10%  | > 50%     | Active members                                 |
| `Chat.join_policy`                 | ≥ 30%  | ≥ 2/3     | Active members                                 |
| `Chat.epoch` (mid-epoch rotation)  | ≥ 50%  | ≥ 2/3     | Active members                                 |
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

The two-edge approval pattern from
[graph-model.md §5](../primitive/graph-model.md) instantiates as
three chat-specific shapes. The mechanics — actor edge to a
ChatMember junction, system writes the claim, both edges required
for active membership — are identical to the primitive. The
chat-specific piece is **who creates the actor edge(s)**:

| Variant         | Actor edges required                                                |
|-----------------|---------------------------------------------------------------------|
| Open chat       | Would-be member alone; policy satisfied immediately on the claim.   |
| Invite-only     | Admin (or invite-rights member) **and** invitee, both required.     |
| Request-entry   | Would-be member **and** an approving admin, both required.          |

Invite-only and request-entry are **topologically identical** — only
who initiates first differs. Multi-sig variants are just a higher N
in the approval policy.

---

## 8. Membership lifecycle

Membership is encoded in the two-edge approval pattern: claim-only
edge → pending; both edges → active. State transitions (voluntary
leave, community removal) follow the primitive — see
[graph-model.md §5](../primitive/graph-model.md) ("Revocation and
state transitions") for the rule.

The chat-specific configuration is the removal instance: chats use
the member-disavowal instance defined in §6 — a Shape B vote with
the chat's configured quorum and threshold. There is no
admin-unilateral kick path; admins participate in the disavowal
vote with their role-weighted vote alongside everyone else.

Every membership-change event — voluntary leave, member-disavowal
pass, or new active member — also triggers the automatic chat-key
rotation described in §5: `Chat.epoch` advances by 1 and the new
member set re-runs the group-key-update procedure off-graph. The
leaving member keeps the keys for past epochs they were active in
but cannot derive the new one. This is the mechanism that limits
ex-member leakage to the time they were actually in the chat.

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
