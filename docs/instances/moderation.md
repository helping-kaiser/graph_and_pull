# Moderation

CoGra moderates publicly-visible content via the same governance
primitive everything else uses: any User can create a Proposal
classifying content as `sensitive` (per-node soft flag) or
`illegal` (per-field redaction); the Network votes Shape A;
threshold-cross applies the classification via cascade.
**No privileged moderator role with extra weight** — mods exist as
a gate, not as weighted voters.

The defense against bot-driven flooding lives in the gate: every
classification change requires **at least one moderator's positive
vote** in the tally. Bots can flood the community side but cannot
cross the gate without compromising a real moderator.

Encrypted ChatMessages can technically be voted on at any time —
the protocol does **not** block a Proposal against a ciphertext.
"Moderate only after disclosure" is a normative requirement on
moderators, not a protocol invariant; voting blind is a
mod-conduct violation, addressable through the same primitive
that handles any mod misconduct (see §5). Until the relevant
chat key has been disclosed, chat-internal disavowal
([chats.md §10](chats.md#10-moderation)) is the only meaningful
recourse.

### Vocabulary: moderation vs disavowal

**Invariant — scope reservation.** "Moderation" is Network-scope:
the act of flipping a node's `moderation_status` to `'sensitive'`
or `'illegal'` via the governance flow in this doc. "Disavowal"
is Chat-scope: a `dim1 < 0` layer landed (via the Level 1 / Level 2
Proposals in [chats.md §10](chats.md#10-moderation)) on a
ChatMessage- or ChatMember-targeting outcome. The two differ in
eligibility (Network actives vs chat members), cascade (redaction
vs `:APPROVAL` edge layer), and reversibility. In chat contexts,
use "disavowal" — not "removal," "kick," "fire," or "expel."

## 1. The two classification paths

`sensitive` and `illegal` operate on different units. Both are
authorized by the same governance instance below; only the
cascade outcome differs.

### `sensitive` — node-level soft flag

Every user-input-bearing node carries a `moderation_status` graph
property (`'normal'` / `'sensitive'` / `'illegal'`, default
`'normal'`, layered — see [nodes.md](../primitive/nodes.md)). A
passing `'sensitive'` Proposal flips the top layer of this
property to `'sensitive'`. Effect: frontend respects each viewing user's
`content_filtering_severity_level` (see
[data-model.md](../implementation/data-model.md) "User
preferences"); content stays. Reversible via a counter-Proposal
back to `'normal'`.

### `illegal` — per-field redaction

Illegal-content classification is per-field, not per-node. A
passing `'illegal'` Proposal targets either (a) one specific
user-input field of the content node (e.g., a Post's `content`,
a User's `username`, a Chat's `name`), or (b) the whole-node
`'node'` sentinel, which targets every user-input field plus
every attached media on that node. Threshold-cross fires the
redaction cascade:

1. Each targeted field's top layer is replaced with a redaction
   marker per [layers.md §5](../primitive/layers.md#5-deletion-policy). For media
   targets, the underlying `media_attachments` row is
   tombstoned and the asset is removed from object storage.
2. Each redacted original is written to the
   [retention archive](../primitive/retention-archive.md)
   automatically. The `legal_hold_until` value is set
   asynchronously by `legal_admin` — a member of the host's
   operations team, not a graph role, see
   [retention-archive.md §4](../primitive/retention-archive.md#4-access-path) —
   after case review. The cascade itself does not block on this
   decision. `legal_admin` chooses what happens next: report to
   authorities, retain for prosecution evidence, schedule
   statutory hard-delete, etc. The handoff is post-redaction;
   `legal_admin` has no path back into the live graph.
3. The node's `moderation_status` is auto-flipped to
   `'illegal'` so frontends can distinguish a partially-or-fully
   redacted node from a merely sensitive one and hide it
   entirely if the viewing user prefers. This is a system-side
   derivation, not a separate Proposal. `'illegal'` is the
   strongest state and is not downgraded by a later
   `'sensitive'` Proposal while any redacted fields remain.

The cascade is bounded to what the Proposal targeted and does
**not** propagate to descendants. Classifying a Post's body
illegal does not redact the Post's Comments; each requires its
own classification.

## 2. Reports = Proposals on the graph

A user reporting content **is** the act of creating a Proposal:

- **Subject:** a Proposal node
  ([nodes.md](../primitive/nodes.md)) with target = the content
  node (via `:TARGETS` edge). `target_property` and
  `proposed_value` depend on the path:
  - `'sensitive'` Proposal: `target_property = 'moderation_status'`,
    `proposed_value = 'sensitive'`.
  - `'illegal'` Proposal: `target_property` is a specific
    user-input field name on the target node (e.g., `'username'`,
    `'bio'`, `'content'`, `'avatar'`) — or the whole-node `'node'`
    sentinel to redact every user-input field plus every attached
    media on the node. `proposed_value = 'illegal'`.
- **First reporter** authors the Proposal — the system reads the
  authoring as their +1 vote.
- **Subsequent reporters** cast Shape A votes
  ([governance.md §3](../primitive/governance.md#3-the-two-vote-shapes)) on the existing
  Proposal rather than authoring duplicates. A reporter who
  wants a different target field on the same content node (e.g.,
  one Proposal already targets `content`, they want `'node'`)
  authors a separate Proposal — these are independent
  classifications, not duplicates.
- **Threshold-cross** triggers the cascade described in §1.

There is **no separate Postgres reports table**. Reports live on
the graph as Proposal authoring + Shape A vote layers — fully
transparent, fully auditable, append-only by construction.

## 3. The mod-gate rule

Every moderation Proposal — content classification (`sensitive`
or `illegal`) and un-classification back to `normal` — runs
through the **mod-gate**: at least one positive vote from a User
with `network_role = 'moderator'` must be present in the tally
before the outcome can take effect.

The primitive definition lives in
[governance.md §7](../primitive/governance.md#7-the-mod-gate),
which states the invariant "mod weight = member weight = 1; mod
is a gate, not a weight," and names the failure modes each side
of the multi-gate pattern closes off. The same component reappears
in moderator role changes
([network.md §9](../primitive/network.md#9-mod-role-changes-via-multi-sig-proposal))
and `:Network` parameter amendments
([network.md §11](../primitive/network.md#11-amending-network-parameters)).

Instance-specific arithmetic — `moderation_sensitive_*` and
`moderation_illegal_*` quorum/threshold defaults — lives in §4.

## 4. Eligibility, weights, thresholds

The Network ([network.md](../primitive/network.md)) is the eligibility-and-
voting body for moderation Proposals.

- **Eligibility:** all active Network members (every User with at
  least one outgoing actor edge inside the
  `Network.active_threshold_days` window).
- **Vote weight:** 1 per voter — mod or member.
- **Vote shape:** Shape A — the `User → Proposal` actor edge
  carries the vote. Network membership has no per-member
  junction (see [network.md §8](../primitive/network.md#8-membership-and-roles)),
  so the User node is itself the eligibility carrier. See
  [governance.md §3](../primitive/governance.md#3-the-two-vote-shapes).
- **Tally:** petition-style — only positive votes contribute. See
  [governance.md §3 "Petition-style tally and dual quorum"](../primitive/governance.md#petition-style-tally-and-dual-quorum-network-scope-only).
- **Dual-quorum bars (read from the `:Network` singleton — see
  [graph-data-model.md](../implementation/graph-data-model.md)).**
  A Proposal passes when
  `positive_count ≥ min(P × |active members|, K)`:

  | Action | `P` (`*_quorum_fraction`) | `K` (`*_quorum_count`) | Mod gate |
  |---|---|---|---|
  | Classify `sensitive`                       | `Network.moderation_sensitive_quorum_fraction` (default `0.25`) | `Network.moderation_sensitive_quorum_count` (default `5000`) | ≥1 mod positive |
  | Classify `illegal`                         | `Network.moderation_illegal_quorum_fraction` (default `0.50`) | `Network.moderation_illegal_quorum_count` (default `10000`) | ≥1 mod positive |
  | Un-classify `sensitive` → `normal`         | symmetric to the original action (`moderation_sensitive_*`)     | symmetric                                                       | ≥1 mod positive |

  `'illegal'` is **not** reversible. The redaction markers on
  the targeted fields are append-only per
  [layers.md §5](../primitive/layers.md#5-deletion-policy), and
  `moderation_status = 'illegal'` is a system-derived consequence
  of those markers existing on the node — flipping the status back
  while markers remain would misrepresent the node's state. A
  later `'sensitive'` Proposal also does not downgrade the status
  while any redacted fields remain (see
  [nodes.md](../primitive/nodes.md)).

The fractional bar `P` governs while the network is small (a real
majority of active members is required to pass). Once membership
scales past `K / P` active members, the absolute bar `K` takes
over (a fixed engagement-level positive-vote count is sufficient).
The mod gate carries the integrity guarantee independently of
either bar.

Every number above is a property of the `:Network` singleton,
amendable via the rules in
[network.md §11](../primitive/network.md#11-amending-network-parameters) — the
`moderation_illegal_*` parameters fall in the critical bucket
(higher fractional bar, larger absolute count) because their
abuse drives the redaction cascade; the `moderation_sensitive_*`
parameters fall in the baseline bucket. Defaults exist to
bootstrap; they are not fixed rules.

## 5. Scope

`moderation_status` (the sensitive flag) is a graph-side property
on every user-input-bearing node — User, Collective, Post,
Comment, ChatMessage, Chat, Item, Hashtag.

**Illegal-classification target fields** — valid `target_property`
values for an `'illegal'` Proposal:

| Node | Targetable fields | `'node'` covers |
|---|---|---|
| **User** | `username`, `display_name`, `bio`, `avatar`, `website_url` | all of the above |
| **Collective** | `name`, `display_name`, `description`, `avatar`, `website_url` | all of the above |
| **Post** | `content`, `attachments` (all attached media on the post) | both |
| **Comment** | `content`, `attachments` | both |
| **ChatMessage** | `content`, `attachments`. Both `plaintext` and `encrypted` per [chats.md §9](chats.md#9-encryption-as-the-privacy-mechanism); encrypted messages are classifiable once readable (see "encrypted message classification" below) | both |
| **Chat** | `name`, `description`, `image` | all three |
| **Item** | `name`, `description`, `attachments` | all of the above |
| **Hashtag** | `name` | n/a (only field) |

The field-name set per node type tracks the user-input fields
enumerated in [nodes.md](../primitive/nodes.md) and
[data-model.md](../implementation/data-model.md). Adding a new
user-input field to any of these node types automatically adds it
to the valid `target_property` set; the cascade handler must
know how to redact it (graph-side layer marker, Postgres
tombstone version row, or media tombstone + asset removal).

Per-attachment targeting (redacting one specific attachment on a
Post that has several) is a future refinement — the current shape
redacts all attachments under `target_property = 'attachments'`.

**Out of scope:**

- Junction nodes (`ChatMember`, `CollectiveMember`,
  `ItemOwnership`) and `Proposal` nodes — they carry no
  user-authored content fields.

### Encrypted message classification

For a moderation Proposal targeting an encrypted ChatMessage to be
useful, voters need to be able to read the body. The disclosure
path is **independent of the moderation primitive** — any chat
member can release the relevant epoch's chat key (per
[chats.md §9](chats.md#9-encryption-as-the-privacy-mechanism)) through any normal authoring
gesture: a Comment on the chat, a public Post, a plaintext
ChatMessage in the same chat, an off-graph channel, anything. The
system permits voluntary disclosure by participants by design.
Disclosure is scoped to the disclosed epoch only; leaking Kᵢ
exposes E_i's messages and no others.

This matters in practice for cases like contracts in private chats
(forthcoming with the economics) where one party may need to
surface the other's misbehavior.

#### Why this is a norm, not a protocol gate

The protocol does not block a Proposal authored against an opaque
ciphertext, nor votes cast on it. A bot swarm can `+1` encrypted
bodies all day, and a malicious moderator can cross the gate (§3)
without reading anything. What prevents this is the role
definition, not the code:

- **Bot voting on ciphertext** is the same noise-vs-consistency
  problem as any other bot voting (§7) — the mod gate guarantees
  consistency, since no Proposal crosses without a real
  moderator's positive vote.
- **A moderator voting on undisclosed ciphertext** is a
  mod-conduct violation. The remedy is the same Proposal
  primitive applied to that User's `network_role` — the Network
  votes the offender out of the moderator role
  ([network.md](../primitive/network.md)).

The integrity guarantee is a **two-part claim**: the mod gate (§3)
blocks the consistency attack; the de-mod-ing path addresses
moderator misconduct. Together they make "moderate only after
disclosure" a load-bearing norm rather than a protocol invariant
— the most we can offer without graph-level guards that would be
both too weak (off-graph disclosure exists, and the graph cannot
detect it) and too strict (legitimate cases like contract disputes
would be blocked).

**The cascade fires regardless of disclosure state.** If a
Proposal targeting an encrypted ChatMessage crosses threshold —
including the mod-gate `+1` — the redaction cascade in §1 runs
whether or not any voter actually read the body. The protocol
inspects the tally, not decryption state. A Network whose
moderators wave through cascades on opaque ciphertext is already
broken; the remedy is the de-mod-ing path above, not a protocol
veto. The robustness guarantee rests on moderator judgment, not
on the protocol second-guessing it.

## 6. Coexistence with chat-internal moderation

Platform moderation (this doc) and chat-internal disavowal
([chats.md §10](chats.md#10-moderation)) can both apply to the
same `ChatMessage`. They sit at different scopes (Network vs
chat), eligibility differs (every active Network member vs
active `ChatMember`s), and outcomes write to different graph
state — so they do not conflict.

The primitive coexistence rule — scope decides the state
written, instances at different scopes never compete for the
same write — lives in
[governance.md §9](../primitive/governance.md#9-coexistence-multiple-governance-instances-on-a-shared-subject).
The chat-vs-platform pairing is the worked example in that
section.

## 7. Noise vs consistency — what the mod gate does and doesn't solve

A bot net could try to flood the system by **mass-creating**
moderation Proposals against legitimate content and **mass-voting**
on each other's Proposals. Two distinct concerns, only one of
which the mod gate addresses:

- **Consistency.** No spam Proposal can apply without a real
  moderator's positive vote (§3). A million bot-authored
  Proposals against legitimate content cannot cross threshold.
  The classification cannot drift from `'normal'` without mod
  consent. The mod gate fully covers this.
- **Noise (operational).** Mods reviewing the queue could be
  drowned in bot-authored Proposals, with real reports buried in
  the noise. The mod gate doesn't address this directly.

Noise is handled out-of-graph by the same mechanisms used for the
rest of the platform:

- **Feed-ranking.** Moderator UIs surface Proposals through the
  same per-viewer ranking ([feed-ranking.md](../primitive/feed-ranking.md))
  used for content. Bot-authored Proposals from severed clusters
  land at zero `h(t)` and never surface to honest mods. Real
  reports surface because they originate from non-severed users
  with real reach into the moderator's network.
- **API rate limits.** Per-author throttling on Proposal creation
  is an operational concern, same as login rate limits — it lives
  in the API layer, not the graph primitive.

Premature graph-level defenses (e.g. a `vote-restricted` role)
are deliberately not added. If real-world experience proves the
operational mechanisms insufficient, a graph-level role can be
added later — but adding it speculatively would risk being wrong
about the real attack shape.

## 8. Platform guidelines

The Network publishes normative platform guidelines covering what
counts as `illegal`, what counts as `sensitive`, and what is
`normal` — voters reference these when deciding their position
on a moderation Proposal.

The guidelines live in
[platform-guidelines.md](platform-guidelines.md). They are
amendable via the same Proposal primitive (eligibility = Network
members; dual-quorum bars in
`Network.guidelines_change_quorum_fraction` /
`Network.guidelines_change_quorum_count`, tuned higher than
single-content classification because an amendment shifts the
normative frame for *all future* moderation). The current version
is pinned on the graph by
`Network.guidelines_version` + `Network.guidelines_hash` (SHA-256
of the canonical document bytes).

## What this doc is not

- **Not the Network primitive.** Membership, the moderator role,
  and how mods come and go are in [network.md](../primitive/network.md).
- **Not the redaction mechanism.** The redaction cascade is
  defined in [layers.md §5](../primitive/layers.md#5-deletion-policy) and the
  archive disposition in
  [retention-archive.md](../primitive/retention-archive.md); this
  doc provides the community-driven authorization for
  illegal-content classification (resolving
  [open-questions.md Q9](../open-questions.md); account-deletion
  is a separate user-initiated authorization path).
- **Not the platform guidelines themselves.** The bucket contents
  and amendment procedure are in
  [platform-guidelines.md](platform-guidelines.md).
