# Items

An **Item** is a content node representing a physical or digital good
— something that can be owned, transferred, and talked about. Items
are interactable content: they can be liked, disliked, commented on,
and tagged with hashtags (see
[edge-tensor-model.md §5](edge-tensor-model.md) for the relevant
edges).

Items are a **future** concern in the sense that the first iterations
of CoGra focus on posts and chats; marketplace-like item flows will
build on top of the graph model once the base is running. The model
below is committed to regardless.

## Ownership: ItemOwnership

An `ItemOwnership` is a junction node (see
[edge-tensor-model.md §2](edge-tensor-model.md)) representing a
specific ownership claim. Each transfer creates a **new**
ItemOwnership node — old ones are never removed. Together they form
an **append-only chain of an item's ownership history**.

This means a single Item typically has many ItemOwnership nodes over
its lifetime, one per transfer event. The current owner is derived
from the most recent approved ItemOwnership (see below).

## Transfer flow

ItemOwnership uses the **two-edge approval pattern** described in
[edge-tensor-model.md §6](edge-tensor-model.md):

1. **Acquirer** (User or Company) creates an actor edge toward a new
   **ItemOwnership** node.
2. System creates `ItemOwnership -> Item` (claim, pending).
3. **Current owner** creates an actor edge toward the same
   ItemOwnership node with positive sentiment (approval).
4. Approval policy is satisfied; system creates
   `Item -> ItemOwnership` (approval).
5. Transfer is complete; the new ItemOwnership is now the active one.

No one can take ownership without the current owner's explicit
approval — there is no "take" operation in the graph.

## Identifying the current owner

Earlier ItemOwnership nodes keep their `Item -> ItemOwnership` edges
from when they were current — append-only applies, so those edges
aren't removed when a transfer happens. The **current** owner is
identified by whichever ItemOwnership has the most recent
`Item -> ItemOwnership` approval edge.

This is analogous to how authorship is derived from the *earliest*
incoming edge on a node (see [authorship.md](authorship.md)) — except
here we care about the *latest* approval rather than the earliest
claim.

## Leaving / superseded ownership

The sequential-chain shape means ItemOwnership departures are
relatively simple: a new ItemOwnership supersedes the old one by being
the most recent approved claim. That said, formal encoding of
"this ItemOwnership is no longer current" — so the graph is explicit
rather than relying only on timestamp comparisons — is part of the
cross-junction state-transition question. See
[edge-tensor-model.md §10 Q#4](edge-tensor-model.md).
