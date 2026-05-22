# Layers

CoGra is **append-only everywhere that matters**. Every piece of
authored or expressed state is layered rather than overwritten —
edges, node properties, and display content in Postgres all follow
the same rule. The current state is the top layer; the full history
is always available.

---

## Append-only vocabulary

"Append-only" in CoGra covers three distinct mechanisms, all
sharing the principle that history is preserved rather than
overwritten:

1. **Append-only edges** — see [§2](#2-layers-on-edges).
2. **Append-only node properties** — see [§3](#3-layers-on-nodes).
3. **Versioned-row Postgres display content** — see
   [§4](#4-layers-on-postgres-side-display-content).

Other docs link the word "append-only" to this section as a
shared alias.

---

## 1. Why layers everywhere

The append-only principle isn't about edges specifically — it's
about never erasing what was. Transparency and auditability matter
more than the convenience of being able to "delete" something.

Concrete consequences:

- You cannot hide that you disliked a post in the past; you can only
  add a newer layer that changes your current opinion.
- You cannot hide that you used to be a member of a chat; you can
  leave, but the record of having been a member stays.
- You cannot hide that your username used to be something else; name
  changes add a new layer.
- You cannot delete a message you sent; you can add a new version
  (correction, edit) but past versions are preserved.

This applies equally to edges, node properties, and display content.

---

## 2. Layers on edges

Every edge is a stack of layers. Each interaction adds a new layer
with its own dimension values, timestamp, and layer number. The top
layer is the current state; the full history is available for any
algorithm that needs it (e.g. detecting opinion shifts or weighting
by interaction frequency).

See [graph-model.md §4](graph-model.md#4-edge-structure) for the edge structure and
[graph-model.md §8](graph-model.md#8-append-only-history-edges) for edge-specific history
details.

**"Revoked" names the negative-top-layer state.** An approval-pair
structural edge is **revoked** when its top layer carries
`dim1 < 0` (a removed CollectiveMember, a disavowed ChatMember,
an ItemOwnership replaced by the next ownership). Older drafts
used "inactive" or "superseded" for the same state; both are
aliases for revoked. The *mechanism* producing the negative
layer (supersession cascade, voluntary leave, governance
threshold-cross) varies; the *state* it produces is always
"revoked." See
[graph-model.md §5](graph-model.md#5-junction-node-flows).

---

## 3. Layers on nodes

Nodes can change over time — a user's username, a chat's name, a
ChatMember's role, a CollectiveMember's ownership percentage. These
changes add layers to the **specific property** that changed, not to
the whole node.

### Per-property layering

If Alice changes her username from `alice` to `alice_the_dev`, that's
a new layer on the `username` property of her User node. Her other
properties are untouched — only fields that actually change
accumulate layers. Her edges are separate records and are not node
properties; they have their own independent layer stacks.

A node's current properties are the top layer of each property.
History is preserved per field, independent of other fields.

### What properties belong on graph nodes

Only what the graph **actually needs** for traversal, ranking, or
routing. Example authored properties that layer:

- User: `username` (the handle used for mentions/lookups).
- Chat: `name` (if needed for routing/display hints), `join_policy`
  (read by the system when an actor's claim toward a `ChatMember`
  arrives).
- ChatMember / CollectiveMember: `role`, role-attached quantities
  (`ownership_pct`, `voting_weight`).

If the graph doesn't need a field to compute anything, it doesn't
belong on the graph.

### What does NOT belong on the graph

Display content — bios, profile text, post bodies, message bodies,
image and video URLs — lives in Postgres, not on graph nodes. The
layering rule still applies to those, but it applies to Postgres
rows (see §4).

### Derived caches do not layer

Values derived from graph state are rebuilt from the source of
truth, never layered. For example, `member_count` on a Chat is
derived from counting active ChatMembers — if the underlying
graph changes, rebuild the cache. Layering it would duplicate
history that already lives in the source data.

Named carve-outs to append-only exist only on the Postgres
side and only for operational state (not history) — see
[§5 "Scope of the invariant"](#scope-of-the-invariant). The
graph itself has no carve-outs.

---

## 4. Layers on Postgres-side display content

Display content — message bodies, post text, profile text,
attachment metadata — lives in Postgres (see
[data-model.md](../implementation/data-model.md)). The append-only rule still applies:
an edit writes a **new version row**, not an overwrite. The graph
node stays the same; the Postgres row for that content gets a new
version with the edited text, the old version preserved. Readers see
the current version by default; past versions stay accessible to
anyone who wants the history.

Implementation specifics (schema, version columns, how queries pick
the current version) belong in `data-model.md`. The **rule** lives
here: Postgres display content is append-only too.

---

## 5. Deletion policy

Append-only is the norm, but not absolute on every surface.
"Scope of the invariant" below states which surface is bound to
which rule.

### Redaction vs severance — two different vocabularies

**Invariant:** Redaction and severance describe two different
mechanisms with two different scopes; they are not interchangeable.

- **Redaction** — a content-level mark on a graph layer or a
  Postgres row, applied in place under the authorization paths
  below. Layered, leaves the topology intact, leaves a visible
  marker. The three tiers below describe how it works per surface.
- **Severance** — a `(0, 0)` actor-edge layer one actor writes
  toward another node. Affects path traversal *for that viewing
  user* per
  [feed-ranking.md §3.6](feed-ranking.md#36-bot-resistance-via-the-0-0-severance-edge),
  touches no content, and is per-viewer rather than global.

This section covers redaction only. "Takedown" is not a CoGra
term — older drafts used it as a synonym for redaction; sweep it
in favor of "redaction" wherever encountered.

### Scope of the invariant

Append-only is the rule, but not every system in the stack falls
under it identically. Three surfaces, three rules:

- **The graph (Memgraph): nothing is removed.** No node, no
  edge, no layer, ever. There is no API path, no admin escape
  hatch, no court-order path that deletes graph topology. State
  transitions (revocation, departure, supersession) are encoded
  as new layers on existing edges, not as deletions. The graph's
  job is to be the transparent auditable record; erasing from it
  would defeat the whole point.
- **Postgres: almost nothing is removed.** Display content (post
  bodies, message bodies, profile text, media metadata) is
  append-only; edits write new version rows. Redaction
  tombstones the row (see "Postgres / media display content —
  tombstonable" below), and the tombstone itself stays. The
  named carve-outs to append-only are limited and listed
  explicitly here:
  - `user_view_log` — per-viewer seen-list, operational filter
    state rather than history, compacted on a 1-year default
    per
    [feed-ranking.md §8.5](feed-ranking.md#85-compaction--drop-entries-older-than-1-year-frontend-convention).

  Additions to this list require a named exception added here.
- **Frontends, miners, indexers, and off-graph systems: not
  governed by this invariant.** Whatever they cache, summarize,
  or discard is their concern, not the graph's. The graph is the
  canonical record; downstream consumers may keep, project, or
  drop their copies on their own contracts.

"Deletion" in CoGra always means in-place layer redaction
(graph layer) or a Postgres tombstone version row — see the
mechanism subsections below.

### Layer contents on node properties — redactable

The contents of a specific layer on a node property can be redacted
**in place** when an authorized redaction applies (illegal-content
classification or user-requested account deletion — see
"Authorization paths" below). Redaction replaces the stored value
with a marker like `[redacted — <reason>, removed at T=X]`; the
layer's timestamp, layer number, and position in the stack are
preserved. The fact that a layer existed at that time, and that
something there was redacted, stays visible.

Example — Alice's username history after Layer 2 is taken down
for illegal content:

```
User_Alice.username:
  Layer 1 (T=0):  "alice"
  Layer 2 (T=5):  "[redacted — illegal content, removed at T=11]"
  Layer 3 (T=10): "alice_the_dev"
```

The node itself is untouched. Other property layer stacks are
untouched. Only the offending layer's content is replaced.

### Postgres / media display content — tombstonable

Display content (message bodies, post text, profile text, images,
videos) can be removed from public Postgres or media-server
surfaces under the same two authorization paths (see "Authorization
paths" below). The public surface shows a tombstone version row or
equivalent marker in either case, so the history reflects that
content existed and was removed. The original is moved to the
[retention archive](retention-archive.md) with a per-row legal
hold; archive content is hard-deleted at hold expiry (immediately
in cases like content that is illegal to retain at all).
Implementation specifics belong in
[data-model.md](../implementation/data-model.md).

### Two redaction levels — identity vs content

A redaction action declares its **level**, which fixes the
*scope* of fields and rows the per-surface mechanisms above
touch. Two levels exist:

- **Identity-level.** Touches identity-bearing fields on the
  actor node only — the actor's name layer on the graph side
  and the actor's profile/identity fields and avatar on the
  Postgres side. Authored content bodies are not touched.
- **Content-level.** Identity-level *plus*, for each Post /
  Comment / ChatMessage attributed to the actor, the body
  version row in Postgres and any attached media rows. The
  graph nodes, authoring edges, and layer stacks stay; only the
  bodies and their attachments become unavailable to public
  readers.

The level is a property of the *authorization path*, not of the
mechanism — the in-place layer marker and the Postgres-tombstone
version row apply the same way at either level. What differs is
the *set* of fields and rows the path elects to touch.

The two paths today use the level distinction differently:

- **Illegal-content** redactions per
  [moderation](../instances/moderation.md) choose specific
  fields (one named field, or the `'node'` sentinel covering
  every user-input field plus attached media). Whether the
  result is identity-equivalent or content-equivalent follows
  from the field set chosen.
- **Account deletion** per
  [account-deletion](../instances/account-deletion.md) defaults
  to identity-level and offers content-level as an explicit
  opt-in. The default is identity-only because content was
  publicly authored — PII control happened at write time — and
  mass-redacting bodies would destroy other actors' record of
  conversations they participated in. Content-level is the
  explicit choice for an actor who later regrets what they
  wrote.

Per-row archive holds (typically short for ordinary PII, longer
for content with statutory retention obligations) are set by the
authorization path's policy, not by the level itself. See
[retention-archive.md](retention-archive.md) for the disposition
mechanism.

### The operating principle

**Invariant:** No silent deletion. Every redaction — graph-side
layer marker or Postgres tombstone version row — leaves a visible
record that the change happened. A reader scanning the graph or the
content tables can always tell that something was there and was
removed, even when they cannot see the original content.

**No silent deletion, ever.** Whether a redaction happens via
in-place layer markers or via Postgres version tombstones, the
fact of the deletion is recorded. You can always see that a change
happened, even when the illegal content itself is gone.

The hope is that the community's graph-level mechanisms (voting to
move away from messages, down-weighting, social feedback) handle
most bad content without ever needing the deletion exception. The
exception exists because append-only alone cannot solve "this layer
4 content is still illegal and still findable."

### Authorization paths

Layers.md defines the redaction *mechanism*; the *authorization* —
who decides what gets redacted, by what process — runs through
separate instance docs by scope. Two paths exist today:

- **Illegal content.** Network-level governance per
  [moderation](../instances/moderation.md). Any User can author
  a Proposal classifying content as `'illegal'`; threshold-cross
  requires at least one moderator's positive vote, a community
  quorum, and a supermajority. The cascade then triggers the
  redaction defined above.
- **Personal data on user request.** A User can request that
  their own account's PII be redacted from public surfaces, per
  [account-deletion](../instances/account-deletion.md). The
  redaction is identity-level by default and content-level on
  opt-in.

External pressure (court orders, legal demands) for illegal
content does not bypass the moderation mechanism; the principle
that all external demands enter as ordinary Proposals lives in
[governance.md §7 "External demands enter as Proposals"](governance.md#external-demands-enter-as-proposals).
Court-ordered user-anonymization is a separate path planned in
account-deletion.md, also routed through Proposals.

Disposition of the redacted original (preserve vs. destroy) is
the same mechanism in both paths — the
[retention archive](retention-archive.md) — with per-row hold
values set per case.

### Side note on long-term storage

Append-only means every interaction adds a layer, forever. In
principle this is unbounded; in practice, **typical actor
behavior bounds it tightly**. People update an edge a handful of
times over its lifetime, not hundreds. Node properties change
even less frequently — most don't change at all. The storage
worst case (an actor edge or node property accumulating dozens
or hundreds of layers) is a corner case, not a typical user.
The plausible scenarios for genuine accumulation — e.g., a
decades-old company restructuring constantly through its
CollectiveMember edges — are precisely the cases where
**preserving the full history is the value**, not a cost worth
optimizing away.

If a real instance ever runs into a storage problem from layer
accumulation, compaction-friendly approaches exist that don't
break the no-silent-deletion principle (e.g., a rollup layer
that summarizes a window of past layers while leaving a visible
marker that compaction occurred). That's an
implementation-time decision contingent on real data, not a
design-time one. We are not designing for it preemptively.
