# Items

An **Item** is a content node representing a physical or digital good
— something that can be owned, transferred, and talked about. Items
are interactable content: they can be liked, disliked, commented on,
and tagged with hashtags (see
[edges.md](../primitive/edges.md) for the relevant
edges).

Items are a **future** concern in the sense that the first iterations
of CoGra focus on posts and chats; marketplace-like item flows will
build on top of the graph model once the base is running. The model
below is committed to regardless.

## Ownership: ItemOwnership

An `ItemOwnership` is a junction node (see
[graph-model.md §2](../primitive/graph-model.md)) representing a
specific ownership claim. Each transfer creates a **new**
ItemOwnership node — old ones are never removed. Together they form
an **append-only chain of an item's ownership history**.

This means a single Item typically has many ItemOwnership nodes over
its lifetime, one per transfer event. The current owner is derived
from the most recent approved ItemOwnership (see below).

## Transfer flow

ItemOwnership uses the **two-edge approval pattern** described in
[graph-model.md §5](../primitive/graph-model.md):

1. **Acquirer** (User or Collective) creates an actor edge toward a new
   **ItemOwnership** node.
2. System creates `ItemOwnership -> Item` (claim, pending).
3. **Current owner** creates an actor edge toward the same
   ItemOwnership node with positive sentiment (approval).
4. Approval policy is satisfied; system creates
   `Item -> ItemOwnership` (approval).
5. Transfer is complete; the new ItemOwnership is now the active one.

No one can take ownership without the current owner's explicit
approval — there is no "take" operation in the graph.

## Supersession: exactly one active ItemOwnership per item

When a transfer completes and the new `Item -> ItemOwnership` approval
edge is created, the system **automatically** adds a new layer on the
**previous** ItemOwnership's `Item -> ItemOwnership` approval edge
with `dim1 < 0` — marking it revoked/superseded. This uses the
general state-transition mechanism on structural edges described in
[graph-model.md §5](../primitive/graph-model.md).

The invariant is: **at most one ItemOwnership per item has a positive
top layer on its approval edge at any time.** Identifying the current
owner is therefore a single-edge query — "find the ItemOwnership
whose `Item -> ItemOwnership` top layer has `dim1 > 0`" — with no
timestamp comparisons required.

The cascade is why this works under append-only: the old approval
edge isn't removed, it just has a newer layer that flips its state to
revoked.

An item with **no** active ItemOwnership (no positive top layer on
any `Item -> ItemOwnership` edge) is considered abandoned. The
history of all previous owners remains visible in the layer stacks.

## Shared ownership routes through a Collective

The single-owner invariant is deliberate. There is **no
direct co-ownership** of an Item by multiple parallel
ItemOwnership junctions. When several actors want to share an
item, the established pattern is to make the owner a **Collective**
that the sharing actors are CollectiveMembers of (see
[collectives.md](collectives.md)).

A married couple co-owning a car, three roommates sharing a coffee
machine, a band co-owning equipment, a co-op holding tools — all
of these are modeled as: a Collective node, the sharing actors as
its CollectiveMembers, the Collective as the holder of the
ItemOwnership. Internal disputes about the shared item are
resolved by the Collective's own social contract — its own
governance instances — not by parallel-ItemOwnership voting on the
graph.

This keeps the Item-side query model simple (one active owner per
item) and uses the existing Collective primitive for collective
ownership rather than inventing a parallel mechanism.
