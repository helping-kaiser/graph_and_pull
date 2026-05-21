# User

The **User** is the actor node representing a person on the
platform. It is one of two actor node types — the other is
[Collective](../instances/collectives.md). Both share the same
outgoing-edge catalog and the same authorship mechanics; the
distinction lives in what stands behind each (§1).

This doc is the per-node catalog for the User: how it is created,
what it carries on the graph and in Postgres, what edges it can
participate in, and how it ends. The mechanics those topics depend
on stay in their topical docs — this doc links rather than
duplicates.

---

## 1. User vs Collective

Both User and Collective are actor nodes
([nodes.md §1](nodes.md#1-actor-nodes)) and the graph treats them
identically as actors: same outgoing actor-edge catalog
([edges.md §1](edges.md#1-actor-edges)), same authorship rule
([authorship.md](authorship.md)), same ability to author content
and participate in junctions. The distinction is what stands
behind each on the off-graph side.

- A **User** is a person. The User holds off-graph credentials
  (password hash, verified email, refresh-token sessions — see
  [auth.md](../implementation/auth.md)) that authenticate the API
  requests originating their edges.
- A **Collective** is a group acting through a single graph
  identity. A Collective has no credentials of its own; its
  actions originate from one or more Users authenticated through
  their own sessions, mediated by membership in the Collective
  via [CollectiveMember](../instances/collectives.md#3-graph-side-properties).
  A Collective can itself be a CollectiveMember of another
  Collective, so the chain may be nested.

**Every Collective acts on behalf of one or more Users.** A
Collective is not itself a User; it is a graph-side persona whose
actions trace, possibly through nested CollectiveMember chains,
back to Users. The graph records the action as the Collective's
own; the off-graph authentication that produced it belongs to a
User.

---

## 2. Creation

Two paths produce a User node, both gated on email verification:

- **Invitation (default).** An existing actor generates a
  time-gated invite link (single-use or multi-use), the invitee
  registers and verifies their email, and the system atomically
  creates the User node together with the two invitation edges
  per [invitations.md](invitations.md). The invitee is never an
  isolated node — they have outgoing reach from the moment they
  exist.
- **First-user genesis.** A fresh instance has no accounts; the
  first User self-registers without a token and is also installed
  as the genesis moderator of the
  [:Network singleton](network.md#2-creation). All subsequent
  Users come in via invitation.

The credential and email-verification flow that wraps both paths
lives in [auth.md](../implementation/auth.md). The graph-side
edge-creation pattern is in [invitations.md](invitations.md).

**Invariant: no User node before verification.** The graph has
no "unverified" or "pending" User state and no concept of
partial actorhood. A User node either exists with full standing
or it does not exist. This is the no-half-state spirit of
[layers.md](layers.md) applied at the node-existence level: an
"unverified" holding state would add semantics no other
primitive uses, and the ranking math
([feed-ranking.md](feed-ranking.md)) is not designed for actors
whose actor-edges have provisional weight. Pre-verification
state is held off-graph (a pending-registration record in
auth's storage); on email verification, the User node and its
invitation edges are written atomically. See
[auth.md "Account lifecycle"](../implementation/auth.md#account-lifecycle)
for the implementation.

---

## 3. Graph-side properties

Every authored property on the User node is layered per
[layers.md §3](layers.md#3-layers-on-nodes); changes append a new
layer rather than overwriting.

- **`username`** — the handle used for mentions and lookups.
  Layered, so name-change history is preserved.
- **`network_role`** — `member` (default) / `moderator`. Backs
  platform-wide governance per
  [network.md §8](network.md#8-membership-and-roles); promotion
  and demotion run through the multi-sig Proposal pattern in
  [network.md §9](network.md#9-mod-role-changes-via-multi-sig-proposal).
- **`moderation_status`** — `normal` / `sensitive` / `illegal`,
  default `normal`. Universal across all nodes that carry
  user-authored content; the per-node mechanics (set by Proposal,
  auto-flipped by redaction) are described in
  [nodes.md "Universal: moderation_status"](nodes.md#universal-moderation_status).

Additional authored properties (display-name fields, verified
flags, etc.) can be added later — each would layer independently
under the append-only rule. Concrete property types,
constraints, and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 4. Postgres-side content

The User's display material — `display_name`, `bio`, `avatar`,
cover image, `website_url`, and any other profile content — lives
in Postgres and is linked to the graph User node by UUID. Like
graph-side properties, edits to display content are append-only
per [layers.md §4](layers.md#4-layers-on-postgres-side-display-content):
new version rows, no overwrite. Concrete schema lives in
[data-model.md](../implementation/data-model.md).

---

## 5. Edges

### As actor source (outgoing)

A User can author actor edges toward every other node category.
The full per-target-type catalog with dimension labels lives in
[edges.md §1 "User as actor"](edges.md#user-as-actor). Targets
include:

- Other actors: User, Collective.
- Content: Post, Comment, Chat, ChatMessage, Item, Hashtag,
  Proposal.
- Junctions: ChatMember, CollectiveMember, ItemOwnership.

Some compound gestures defined in other docs reduce to creating
or layering an outgoing actor edge: authoring a node
([authorship.md](authorship.md)), joining or leaving a junction
([graph-model.md §5](graph-model.md#5-junction-node-flows)),
inviting a new actor ([invitations.md](invitations.md)), and
casting a governance vote
([governance.md](governance.md)).

The User does not create structural edges directly — those are
all system-generated as a side effect of the rules in
[graph-model.md §3](graph-model.md#3-edge-categories).

### As target (incoming)

A User receives:

- **Actor edges** from other actors — opinions about them
  (sentiment + interest). See
  [edges.md §1](edges.md#1-actor-edges) for both source-side
  catalogs.
- **`ChatMember / CollectiveMember / ItemOwnership → User`**
  (`:BEARER`) — identity-binding edges from junction nodes the
  User bears (active or pending). One inbound traversal lists
  every membership and ownership the User holds, including
  invitations not yet accepted. See
  [edges.md §2 "Bearer binding"](edges.md#bearer-binding).
- **`ChatMessage / Post / Comment → User`** (`:REFERENCES`)
  when a content node embeds or mentions the User — a chat
  message sharing them, a Post or Comment naming them in the
  body. See [edges.md §2 "Reference"](edges.md#reference).
- **`Proposal → User`** (`:TARGETS`) when a Proposal targets one
  of the User's graph-side properties — typically a
  `network_role` change per
  [network.md §9](network.md#9-mod-role-changes-via-multi-sig-proposal).

A User's relationship to a junction node (ChatMember,
CollectiveMember, ItemOwnership) runs through both the User's
*outgoing* actor edge to the junction (the Shape A self-claim,
once authored) and the *incoming* `:BEARER` edge from the
junction (identity, written at junction creation). See
[graph-model.md §5](graph-model.md#5-junction-node-flows).

---

## 6. Authorship

A User is the author of any node whose earliest incoming actor
edge originates from them. Authorship is derived from the graph
(earliest incoming layer-1 timestamp), cached on the node and in
Postgres metadata for query efficiency. The graph is the source
of truth; caches are rebuildable. See
[authorship.md](authorship.md).

---

## 7. Network membership

Every registered User is automatically a member of the
[Network](network.md) — no approval gate, no separate junction
node. Their `network_role` graph property carries the role.

The User node serves as the eligibility carrier for
network-wide governance directly — this is the single relaxation
of the Shape B junction-carrier rule, justified by the Network
having no per-member junction. See
[network.md §10](network.md#10-network-wide-governance).

Whether Collectives can carry `network_role` (i.e. participate
in platform-wide governance as actors in their own right) is
deferred per [network.md §8](network.md#8-membership-and-roles).
Today only Users do.

---

## 8. Lifecycle

User nodes are **never deleted**. Per
[layers.md §5](layers.md#5-deletion-policy), the only permitted
"removal" on the graph is in-place layer redaction, which
preserves the layer's timestamp, layer number, and position
while replacing the value with a marker.

Three triggers can produce a redaction on a User today; more are
planned:

- **Account deletion (user-initiated).** The User requests
  redaction of their own PII. Two redaction levels —
  identity-only by default, content-level on opt-in — with a
  7-day grace period before execution. Originals go to the
  [retention archive](retention-archive.md) under per-row legal
  hold. Full mechanism in
  [account-deletion.md](../instances/account-deletion.md).
- **Moderation redaction of authored content.** Network
  governance can classify a Post, Comment, ChatMessage, or other
  content node authored by the User as `'illegal'`, triggering
  per-field redaction on the offending node and auto-flipping
  its `moderation_status`. The User node itself is unaffected
  unless the same Proposal targets a User-node property. See
  [moderation.md](../instances/moderation.md).
- **Moderation redaction of a User-node property.** The same
  Proposal mechanism can target a User's own user-input property
  (e.g. `username`). The redaction marker is written to the
  affected layer; surrounding layers stay.

Future triggers — court order, next-of-kin under applicable
inheritance law, network-admin emergency action — are listed in
[account-deletion.md](../instances/account-deletion.md) as
planned reusers of the same redaction mechanism with their own
authorization rules.

The User's UUID is stable across every redaction. Authorship
caches keyed on UUID stay valid; outgoing and incoming edges
keep pointing at the same node. A redacted User is an
anonymized but still-graph-resident actor, not a removed one.

---

## What this doc is not

- **Not the invitation mechanic.** The two-edge invitation
  pattern, link modes, and the bot-cluster trade-off live in
  [invitations.md](invitations.md).
- **Not the network spec.** The `:Network` singleton, mod role
  changes, and platform-wide governance live in
  [network.md](network.md).
- **Not the authentication spec.** Credentials, sessions,
  registration flow, and password reset live in
  [auth.md](../implementation/auth.md).
- **Not the deletion mechanism.** The redaction primitive lives
  in [layers.md §5](layers.md#5-deletion-policy); the
  user-initiated authorization path lives in
  [account-deletion.md](../instances/account-deletion.md).
- **Not the edge catalog.** Per-target-type edges with dimension
  labels live in [edges.md](edges.md).
- **Not the Memgraph or Postgres schema.** Concrete property
  types, columns, and indexes live in
  [graph-data-model.md](../implementation/graph-data-model.md)
  and [data-model.md](../implementation/data-model.md).
