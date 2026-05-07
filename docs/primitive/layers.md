# Layers

CoGra is **append-only everywhere that matters**. Every piece of
authored or expressed state is layered rather than overwritten —
edges, node properties, and display content in Postgres all follow
the same rule. The current state is the top layer; the full history
is always available.

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

See [graph-model.md §4](graph-model.md) for the edge structure and
[graph-model.md §8](graph-model.md) for edge-specific history
details.

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
truth, never layered. Examples:

- `author_id` cached on a Post — derived from the earliest incoming
  edge (see [authorship.md](authorship.md)).
- `member_count` on a Chat — derived from counting active
  ChatMembers.

If the underlying graph changes, rebuild the cache. Layering them
would duplicate history that already lives in the source data.

### Operational filter state — explicit exception

`user_view_log` (per-viewer seen-list, see
[feed-ranking.md §8](feed-ranking.md)) is **operational filter
state**, not graph history. It is exempt from append-only and
runs a periodic compaction (1-year default — see
[feed-ranking.md §8.5](feed-ranking.md)). The trace it leaves is
the visible "history" UI surface fed by the same data, not a
preserved layer stack.

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

Append-only is the norm, but not absolute everywhere. Three tiers,
each with its own rule:

### Graph structure is never deleted

Nodes, edges, and the layer stacks themselves are **never removed**.
No node deletion, no edge deletion, no layer removal, ever. This is
absolute. The graph's job is to be the transparent auditable record;
erasing from it would defeat the whole point.

### Layer contents on node properties — redactable

The contents of a specific layer on a node property can be redacted
**in place** when an authorized takedown applies (illegal-content
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

### The operating principle

**No silent deletion, ever.** Whether a takedown happens via
in-place layer redaction or via Postgres version tombstones, the
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
content does not bypass the moderation mechanism; it prompts a
moderator to start the same Proposal, which the community
resolves on the graph. Court-ordered user-anonymization is a
separate path planned in account-deletion.md.

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
