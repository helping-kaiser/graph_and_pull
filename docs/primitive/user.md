# User

The **User** is the actor node representing a person on the
platform. It is one of two actor node types — the other is
[Collective](../instances/collectives.md). Both share the same
outgoing-edge catalog and the same authorship mechanics; the
distinction lives in what stands behind each (§1).

This doc is the per-node catalog for the User: creation,
graph-side and Postgres-side state, edges, lifecycle.

---

## 1. User vs Collective

Both User and Collective are actor nodes
([nodes.md §1](nodes.md#1-actor-nodes)) and the graph treats them
identically: same outgoing actor-edge catalog
([edges.md §1](edges.md#1-actor-edges)), same authorship rule
([authorship.md](authorship.md)), same ability to author content
and participate in junctions. The distinction is what stands
behind each on the off-graph side.

- A **User** is a person. They hold off-graph credentials
  (password hash, verified email, refresh-token sessions — see
  [auth.md](../implementation/auth.md)) that authenticate the API
  requests originating their edges.
- A **Collective** is a group acting through a single graph
  identity. It has no credentials of its own; its actions
  originate from one or more authenticated Users, mediated by
  [CollectiveMember](../instances/collectives.md#3-graph-side-properties).
  Collectives can nest as CollectiveMembers of other Collectives,
  so the chain may be deep.

Every Collective ultimately acts on behalf of one or more Users:
the graph records the action as the Collective's own; the
authentication that produced it belongs to a User.

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
- **Genesis bootstrap.** A fresh instance has its genesis User
  created by the bootstrap migration that also writes the
  [:Network singleton](network.md#2-creation) and the
  `bot-defense` Hashtag — three nodes, one atomic step. The
  migration runs once at instance creation; no self-registration
  path produces the first User. All subsequent Users come in via
  invitation.

The credential and email-verification flow that wraps both paths
lives in [auth.md](../implementation/auth.md). The graph-side
edge-creation pattern is in [invitations.md](invitations.md).

**Invariant: no User node before verification.** A User node
either exists with full standing or does not exist — no
"unverified" or "pending" partial actorhood. An interim state
would add semantics no other primitive uses, and the ranking
math ([feed-ranking.md](feed-ranking.md)) is not designed for
actor-edges with provisional weight. Pre-verification state is
held off-graph (a pending-registration record in auth's
storage); on verification, the User node and its invitation
edges are written atomically. See
[auth.md "Account lifecycle"](../implementation/auth.md#account-lifecycle).

---

## 3. Graph-side properties

Every authored property on the User node is layered per
[layers.md §3](layers.md#3-layers-on-nodes).

- **`username`** — the handle used for mentions and lookups.
- **`network_role`** — `member` (default) / `moderator`. Backs
  platform-wide governance per
  [network.md §8](network.md#8-membership-and-roles); changes run
  through the multi-sig Proposal pattern in
  [network.md §9](network.md#9-mod-role-changes-via-multi-sig-proposal).

Per-field moderation-status properties cover each user-filled
profile field — `username_status` (for the data-sibling
`username`), `display_name`, `bio`, `avatar`, `website_url` —
plus the node-level `moderation_status` cache. Universal mechanics
in [nodes.md](nodes.md#universal-per-field-moderation-status).

Concrete types, constraints, and indexes live in
[graph-data-model.md](../implementation/graph-data-model.md).

---

## 4. Postgres-side content

The User's display material — `display_name`, `bio`, `avatar`,
cover image, `website_url`, and any other profile content — lives
in Postgres, linked to the graph User node by UUID. Edits are
append-only per
[layers.md §4](layers.md#4-layers-on-postgres-side-display-content):
new version rows, no overwrite. Concrete schema in
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
edge originates from them. On the graph that edge carries the
`:AUTHOR` sub-label, the only representation of authorship on the
graph side. Caches on the node and in Postgres are rebuildable
from it. See [authorship.md](authorship.md).

---

## 7. Network membership

Every registered User is automatically a member of the
[Network](network.md) — no approval gate, no junction. The
`network_role` property carries the role; the User node itself
is the eligibility carrier for Network-scope votes (Shape A,
see [network.md §10](network.md#10-network-wide-governance)).

Whether Collectives can carry `network_role` is deferred per
[network.md §8](network.md#8-membership-and-roles). Today only
Users do.

---

## 8. Lifecycle

User nodes are **never deleted**. The only permitted "removal"
is in-place layer redaction per
[layers.md §5](layers.md#5-deletion-policy).

Three triggers can produce a redaction on a User today; more are
planned:

- **Account deletion (user-initiated).** The User requests
  redaction of their own PII. Two levels — identity-only by
  default, content-level on opt-in — with a 7-day grace period.
  Originals go to the [retention archive](retention-archive.md)
  under per-row legal hold. Full mechanism in
  [account-deletion.md](../instances/account-deletion.md).
- **Moderation.**
  [moderation.md](../instances/moderation.md) targets either a
  User-node field (`bio`, `avatar`, `username_status`, …) or a
  field of content the User authored. Either path leaves the
  User node otherwise intact; redaction of authored content does
  not propagate to the User node unless the same Proposal also
  targets a User-node field.

Future triggers — court order, next-of-kin under applicable
inheritance law, network-admin emergency action — are listed in
[account-deletion.md](../instances/account-deletion.md) as
planned reusers of the same mechanism with their own
authorization rules.

The User's UUID is stable across every redaction; authorship
caches and edges keep pointing at the same node. A redacted User
is anonymized but still graph-resident, not removed.

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
