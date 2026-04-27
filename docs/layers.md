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

Nodes can change over time — a user's username, a chat's title, a
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
- Chat: `title` (if needed for routing/display hints), content-
  privacy setting (plaintext vs E2EE — the graph needs this to
  know what to route).
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

---

## 4. Layers on Postgres-side display content

Display content — message bodies, post text, profile text,
attachment metadata — lives in Postgres (see
[data-model.md](data-model.md)). The append-only rule still applies:
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

### Layer contents on node properties — redactable for illegal content only

The contents of a specific layer on a node property can be redacted
**in place** when the content itself is illegal (an illegal username,
an illegal chat title, etc.). Redaction replaces the stored value
with a marker like `[redacted — illegal content, removed at T=X]`;
the layer's timestamp, layer number, and position in the stack are
preserved. The fact that a layer existed at that time, and that
something there was redacted, stays visible.

Example — Alice's username history after Layer 2 is taken down:

```
User_Alice.username:
  Layer 1 (T=0):  "alice"
  Layer 2 (T=5):  "[redacted — illegal content, removed at T=11]"
  Layer 3 (T=10): "alice_the_dev"
```

The node itself is untouched. Other property layer stacks are
untouched. Only the offending layer's content is replaced.

### Postgres / media display content — deletable for illegal content only

Display content (message bodies, post text, profile text, images,
videos) can be deleted from Postgres or media servers in the same
narrow exception: illegal material, court-ordered takedowns, legally
mandated removals. The deletion still leaves a visible trace — a
tombstone version row or equivalent marker — so the history shows
that content existed and was removed. Implementation specifics
belong in [data-model.md](data-model.md).

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

### Out of scope

Two questions are deliberately not answered here and are tracked
separately:

- **Who authorizes a redaction, and through what process** — a policy
  question (thresholds, legal process, appeal rights). See
  [open-questions.md Q9](open-questions.md).
- **Retention / pruning for storage cost** — an implementation
  optimization concern separate from the principle. See
  [open-questions.md Q10](open-questions.md).
