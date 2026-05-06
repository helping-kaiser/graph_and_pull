# Moderation

CoGra moderates publicly-visible content via the same governance
primitive everything else uses: any User can create a Proposal
classifying content as `sensitive` or `illegal`; the Network votes
Shape B; threshold-cross applies the classification via cascade.
**No privileged moderator role with extra weight** — mods exist as
a gate, not as weighted voters.

The defense against bot-driven flooding lives in the gate: every
classification change requires **at least one moderator's positive
vote** in the tally. Bots can flood the community side but cannot
cross the gate without compromising a real moderator.

Messages in end-to-end-encrypted chats are out of scope — encrypted
bodies are not readable by the community, so they are moderated
chat-internally via [chats.md §6](../instances/chats.md) disavowal.

## 1. The three classifications

`moderation_status` is a graph-side property on every user-input-
bearing node — User, Collective, Post, Comment, ChatMessage
(plaintext only), Chat, Item, Hashtag (see
[nodes.md](nodes.md)). Three values:

| Value | Meaning | Effect |
|---|---|---|
| `normal` | Default; community has not classified. | None. |
| `sensitive` | Community-classified mature / disturbing / etc. | Soft flag. Frontend respects each viewer's `content_filtering_severity_level` (see [data-model.md](../implementation/data-model.md) "User preferences"). Content stays. |
| `illegal` | Community-classified illegal under the platform guidelines. | Redaction cascade per [layers.md §5](layers.md) — graph-side in-place redaction, Postgres-side tombstone. |

The property is layered, so the full classification history is
preserved.

## 2. Reports = Proposals on the graph

A user reporting content **is** the act of creating a Proposal:

- **Subject:** a Proposal node ([nodes.md](nodes.md)) with target =
  the content node (via `:TARGETS` edge), `target_property =
  'moderation_status'`, `proposed_value = 'sensitive'` or
  `'illegal'`.
- **First reporter** authors the Proposal — the system reads the
  authoring as their +1 vote.
- **Subsequent reporters** cast Shape B votes
  ([governance.md §3](governance.md)) on the existing Proposal
  rather than authoring duplicates.
- **Threshold-cross** triggers the cascade: the proposed value is
  written to `moderation_status`, and on `'illegal'` the
  layers.md §5 redaction cascade fires.

There is **no separate Postgres reports table**. Reports live on
the graph as Proposal authoring + Shape B vote layers — fully
transparent, fully auditable, append-only by construction.

## 3. The mod-gate rule

For **any** moderation Proposal to cross threshold, the tally must
include **at least one positive vote from a User with
`network_role = 'moderator'`**. This is not a weight — mods count
as 1, same as everyone else — it is a gate.

The rule applies uniformly across both `sensitive` and `illegal`,
and symmetrically to un-classification (returning content to
`'normal'`):

- Without a mod gate on `sensitive`, a small coordinated group
  could flood-flag legitimate content, forcing endless
  re-moderation.
- Without a mod gate on `illegal`, bot networks could mass-vote
  redactions of legitimate content.
- Without a mod gate on un-classification, bots could strip
  moderation flags from legitimately-classified content.

Same mechanism in every direction. Mods are validators, not
weighted-voters.

## 4. Eligibility, weights, thresholds

The Network ([network.md](network.md)) is the eligibility-and-
voting body for moderation Proposals.

- **Eligibility:** all active Network members (every User).
- **Vote weight:** 1 per voter — mod or member.
- **Vote shape:** Shape B from the voter's User node directly.
  See [governance.md §3](governance.md) for the relaxation
  that permits a User node (rather than a junction) to carry
  the vote for Network-level governance.
- **Default thresholds (starting points, not fixed rules):**

  | Action | Quorum (% of active Network) | Pass-threshold (of cast) | Mod gate |
  |---|---|---|---|
  | Classify `sensitive` | ≥1% | >50% | ≥1 mod positive |
  | Classify `illegal` | ≥2% | ≥2/3 | ≥1 mod positive |
  | Un-classify back to `normal` | symmetric to the original action | symmetric | ≥1 mod positive |

Quorum percentages are deliberately low so decisions can actually
finish — at network scale, even 1-2% participation in a specific
decision is high. The mod gate carries the integrity guarantee;
quorum just keeps a single mod from acting unilaterally.

Every number above is a Network-level parameter, itself amendable
via the same Proposal primitive ([governance.md §2.4](governance.md)).
Defaults exist to bootstrap; they are not fixed rules.

## 5. Scope

**In scope** (`moderation_status` exists on these node types):

- **User, Collective** — for the user-authored fields (avatar,
  bio, profile text, name).
- **Post, Comment** — content bodies and media.
- **ChatMessage** — in plaintext chats.
- **Chat** — name, description, image.
- **Item** — name, description, media.
- **Hashtag** — the canonical name itself.

**Out of scope:**

- ChatMessages in end-to-end-encrypted chats — the community
  cannot read encrypted bodies, so platform-wide moderation
  cannot classify them. They are moderated chat-internally via
  the disavowal mechanism in [chats.md §6](../instances/chats.md).
- Junction nodes (`ChatMember`, `CollectiveMember`,
  `ItemOwnership`) and `Proposal` nodes — they carry no
  user-authored content fields.

## 6. Coexistence with chat-internal moderation

Two distinct mechanisms can apply to a plaintext chat message:

- **Platform moderation (this doc).** Network-level
  classification. Drives the redaction cascade for `illegal`.
  Eligibility = every User.
- **Chat-internal disavowal** ([chats.md §6](../instances/chats.md)).
  The chat's stance toward a message or member. Eligibility =
  active ChatMembers of that chat.

Both can apply to the same content and produce different outcomes
— a chat-disavowed message is still platform-`'normal'` until the
Network classifies it, and a message classified `'illegal'`
platform-wide stays in any chat that hasn't disavowed it. The
platform outcome is destructive (`illegal` → redaction); the
chat-internal outcome is non-destructive (the chat moves away;
the message stays).

## 7. Platform guidelines

The Network publishes normative platform guidelines covering what
counts as `illegal`, what counts as `sensitive`, and what is "not
a problem" — voters reference these when deciding their position
on a moderation Proposal.

The guidelines themselves are a **separate document, deferred to a
follow-up PR**. They are amendable via the same Proposal
primitive (eligibility = Network members; threshold tuned higher
for guideline-level changes).

## What this doc is not

- **Not the Network primitive.** Membership, the moderator role,
  and how mods come and go are in [network.md](network.md).
- **Not the redaction mechanism.** The illegal-only redaction
  cascade is defined in [layers.md §5](layers.md) — this primitive
  provides the community-driven authorization that §5 was missing
  ([open-questions.md Q9](../open-questions.md) resolved here).
- **Not the platform guidelines themselves** (forthcoming).
