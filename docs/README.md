# CoGra Docs

Three layers of documentation, each in its own folder.

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

1. Start with [`primitive/graph-model.md`](primitive/graph-model.md)
   for the foundation.
2. Read any [`instances/`](instances/) doc to see the primitive
   applied (chats and collectives are the most worked-out examples).
3. Loop back into other [`primitive/`](primitive/) docs (governance,
   edges, layers, …) as the need arises.
4. Read [`implementation/`](implementation/) when you're getting
   ready to write code.

## Layer rule

When a new doc is added or content shifts, ask: **does this
describe the graph itself, an application of it, or how it runs?**
The answer puts it in exactly one folder.

If you find yourself defining a new mechanism inside an
`instances/` doc, that's a sign the mechanism belongs in
`primitive/`. Move it.

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

### `instances/`

- [chats](instances/chats.md) — chats and ChatMessages as
  first-class public content; E2EE privacy of content only;
  message + member disavowal.
- [collectives](instances/collectives.md) — collectives as actors;
  social-contract governance with example configurations
  (corporate, household, co-op).
- [items](instances/items.md) — items as content; ItemOwnership
  transfer flow; single-owner invariant.
- [moderation](instances/moderation.md) — content classifications
  (`normal` / `sensitive` / `illegal`); reports as Proposals on
  the graph; mod-vote-required-for-every-classification gate;
  redaction cascade for illegal.
- [account-deletion](instances/account-deletion.md) — user-initiated
  PII redaction; identity-default and content-opt-in scope;
  7-day grace period; reuses redaction mechanism + archive
  primitives.

### `implementation/`

- [architecture](implementation/architecture.md) — system design,
  dual-database split, data flow.
- [data-model](implementation/data-model.md) — Postgres schema for
  display metadata.
- [graph-data-model](implementation/graph-data-model.md) — Memgraph
  schema: node labels, edge labels, properties, indexes, constraints.
- [development](implementation/development.md) — local setup,
  tools, workflows.
- [api-spec](implementation/api-spec.md) — GraphQL spec
  (outdated, pending redesign).
- [graph-db-options](implementation/graph-db-options.md) — why
  Memgraph; alternatives considered.

### Cross-cutting

- [open-questions](open-questions.md) — consolidated index of
  unresolved design calls.
