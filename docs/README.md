# CoGra Docs

## Layers

- **[`primitive/`](primitive/)** — what the graph IS and how it
  BEHAVES. The rules, mechanisms, and catalogs that describe the
  foundation independent of any specific use case.
- **[`instances/`](instances/)** — concrete applications of the
  primitive. Always reference `primitive/` for mechanics; only
  contain what's specific to that use case.
- **[`implementation/`](implementation/)** — system and code-level
  concerns: Postgres schema, dev commands, deployment, API spec.

[`open-questions.md`](open-questions.md) lives at the root of
`docs/` because it's cross-cutting — unresolved design questions
span all three layers.

## Suggested reading order

1. [`primitive/graph-model.md`](primitive/graph-model.md) for the
   foundation.
2. Any [`instances/`](instances/) doc to see the primitive applied
   (chats and collectives are the most worked-out examples).
3. Other [`primitive/`](primitive/) docs (governance, edges,
   layers, …) as the need arises.
4. [`implementation/`](implementation/) when getting ready to
   write code.

## Layer rule

When a new doc is added or content shifts, ask: **does this
describe the graph itself, an application of it, or how it runs?**
The answer puts it in exactly one folder. A new mechanism inside
an `instances/` doc is a sign the mechanism belongs in
`primitive/` — move it.

## Index

### `primitive/`

- [graph-model](primitive/graph-model.md) — node categories, edge
  categories, dimensions, append-only, junction approval pattern.
- [governance](primitive/governance.md) — weighted role-based voting
  primitive: five components, two vote shapes, sticky outcomes,
  Proposal nodes, multi-candidate decisions.
- [nodes](primitive/nodes.md) — full node catalog with per-type
  graph-side properties.
- [edges](primitive/edges.md) — full edge catalog plus the
  relationship-label scheme at the graph layer.
- [structural-edge-map](primitive/structural-edge-map.md) —
  matrix + mermaid diagram of every structural edge in the
  catalog; visual companion to [edges](primitive/edges.md).
- [layers](primitive/layers.md) — append-only across edges, node
  properties, and Postgres-side display content; deletion policy.
- [retention-archive](primitive/retention-archive.md) — universal
  disposition for redacted originals; per-row legal hold;
  statutory hard-delete on expiry; legal-admin access path.
- [feed-ranking](primitive/feed-ranking.md) — ranking algorithm.
- [authorship](primitive/authorship.md) — how authorship is derived
  from the earliest incoming edge.
- [invitations](primitive/invitations.md) — two-edge onboarding
  pattern for new actors.
- [network](primitive/network.md) — the global community of all
  users on an instance; `network_role` (member / moderator);
  genesis-mod bootstrap; multi-sig role changes.
- [user](primitive/user.md) — per-node doc for the User actor
  node; on-behalf-of distinction with Collective; creation,
  edges, network membership, lifecycle.
- [invariants](primitive/invariants.md) — thin index of the
  load-bearing protocol invariants; each entry links into the
  owning doc's `**Invariant:**` call-out, which is canonical.

### `instances/`

- [chats](instances/chats.md) — chats and ChatMessages as
  first-class public content; E2EE privacy of content only;
  message + member disavowal.
- [collectives](instances/collectives.md) — collectives as actors;
  social-contract governance with example configurations
  (corporate, household, co-op).
- [items](instances/items.md) — items as content; ItemOwnership
  transfer flow; single-owner invariant.
- [moderation](instances/moderation.md) — `sensitive` and
  `illegal` both per-field on a per-field moderation-status
  property; reports as Proposals on the graph;
  mod-vote-required-for-every-classification gate; per-field
  redaction cascade; node-level `moderation_status` cache holds
  the max severity.
- [platform-guidelines](instances/platform-guidelines.md) — the
  normative document the Network references when classifying
  content; bucket contents; amendment procedure pinned by
  `:Network` version + SHA-256 hash.
- [account-deletion](instances/account-deletion.md) — user-initiated
  PII redaction; identity-default and content-opt-in scope;
  7-day grace period; reuses redaction mechanism + archive
  primitives.
- [post](instances/post.md) — per-node doc for the Post content
  node; primary public-content surface; creation, edges,
  authorship, lifecycle.
- [comment](instances/comment.md) — per-node doc for the Comment
  content node; universal threading primitive that attaches to
  Post, Comment, Chat, ChatMessage, or Item.
- [hashtag](instances/hashtag.md) — per-node doc for the Hashtag
  content node; content-addressed UUID makes creation implicit
  and federation reconciliation-free.
- [proposal](instances/proposal.md) — per-node doc for the
  Proposal content node; subject carrier for property-level
  governance votes (target, target_property, proposed_value).

### `implementation/`

- [architecture](implementation/architecture.md) — system design,
  dual-database split, data flow.
- [data-model](implementation/data-model.md) — Postgres schema for
  display content (plus a few operational-metadata tables).
- [graph-data-model](implementation/graph-data-model.md) — Memgraph
  schema: node labels, edge labels, properties, indexes, constraints.
- [development](implementation/development.md) — local setup,
  tools, workflows.
- [api-spec](implementation/api-spec.md) — GraphQL spec
  (outdated, pending redesign).
- [auth](implementation/auth.md) — server-side credentials,
  invitation-based registration, JWT access + Postgres refresh
  tokens, sessions.
- [graph-db-options](implementation/graph-db-options.md) — why
  Memgraph; alternatives considered.

### Cross-cutting

- [open-questions](open-questions.md) — consolidated index of
  unresolved design calls.
