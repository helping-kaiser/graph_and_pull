# Account deletion

A User can request that their account be removed from public view.
Account deletion **does not delete the User node, edges, or layer
stacks** — it triggers a redaction of personally-identifying
information (PII) from public surfaces while preserving the graph
structure that depends on the user's existence. Original PII is
moved to a **retention archive** so the platform can satisfy
statutory retention obligations (e.g., the German 10-year retention
on tax and economic records, forthcoming with the economics
primitive) before the data is permanently destroyed.

The redaction *mechanism* is the one defined in
[layers.md §5](../primitive/layers.md); the disposition of
redacted originals is the
[retention archive](../primitive/retention-archive.md). This doc
adds the **user self-service authorization path** — parallel to
[moderation.md](moderation.md)'s community-driven authorization
for illegal content.

Future triggers — court order, next-of-kin under § 1922 BGB,
network-admin emergency action — reuse the same redaction scope
and archive mechanism. Each gets its own authorization rules; the
redaction action is shared.

## 1. Two redaction levels

**Identity-level (default).** Removes the user's identifying
information without touching their authored content bodies:

- The `username` layer on the graph User node is replaced with
  the [layers.md §5](../primitive/layers.md) redaction marker. The User node
  itself stays; edges and layer stacks stay; counts and authorship
  derivation continue to work.
- The Postgres `users` row is tombstoned — a new version row in
  which `display_name`, `bio`, `avatar_id`, and `website_url` are
  cleared (`NOT NULL` fields set to a redaction marker, nullable
  fields nulled), and `username` is replaced with the unique
  redacted form below. The original row is archived per §3.
- The user's avatar `media_attachments` row is tombstoned and
  archived.
- Private per-user state (preferences, bookmarks, hidden-actor
  lists, read state) is **deleted outright**. These tables hold
  no preservation value once the user is anonymized and carry no
  statutory retention obligation; archive bypass is appropriate.
  Forthcoming economic records (transactions, payouts) will
  instead be archived per §3 because they carry their own
  retention clocks.

**Content-level (opt-in).** A separate, explicit second step on
top of identity redaction. For each Post / Comment / ChatMessage
authored by the user:

- The Postgres body version row is tombstoned and the original
  body archived.
- Attached `media_attachments` rows are tombstoned and archived.
- The graph node, authoring edges, and layer stacks are
  untouched. The post still exists, still ranks, still resolves
  via authorship — only the body and its media become
  unavailable to public readers.

The default is identity-only because public bodies were
**publicly authored** — PII control happened at write time, and
mass-redacting bodies destroys other users' record of
conversations they participated in. Aggressive content redaction
is offered as an explicit second step for users who later regret
what they wrote.

### Username post-redaction

`users.username` is `UNIQUE` in Postgres
([data-model.md](../implementation/data-model.md)). The redacted
form must therefore be guaranteed-unique, not probabilistically
unique. The user's existing UUID PK is unique by construction:

```
users.username = "redacted-user-{user_id_uuid}"
```

This preserves the column invariant, never collides, and remains
traceable to the archive row via the embedded UUID. The
user-facing display value is rendered as `[redacted user]` (or
similar) at the API layer; the storage form satisfies the
uniqueness constraint.

The graph-side `User.username` layer carries the standard
[layers.md §5](../primitive/layers.md) redaction marker, not this string —
layer values have no uniqueness constraint, and the marker is
the auditable absence the layer history requires.

## 2. What is preserved

Account deletion never affects:

- **Graph nodes.** User, Post, Comment, ChatMessage, all authored
  content nodes stay.
- **Edges.** Actor edges (`:LIKES`, `:FOLLOWS`, `:CONTAINMENT`,
  `:REFERENCES`, etc.) stay. Outgoing edges from the redacted
  user keep their author cache pointed at the User node by UUID;
  the UUID does not change. Incoming edges from others are
  untouched.
- **Layer stacks.** Timestamps, layer numbers, and positions are
  preserved everywhere. Only specific layer *values* are replaced
  with markers.
- **Counts and ranking inputs.** Like-counts, member-counts,
  feed inputs continue to include the redacted user's
  contributions. Removing them would alter other users' record
  retroactively.
- **Authorship derivation.** Author = earliest incoming edge by
  timestamp ([authorship.md](../primitive/authorship.md)). Cached `author_id`
  on Posts / Comments / ChatMessages is the User's UUID, which
  does not change on redaction; no cache rebuild needed.

Mentions inside *other* users' posts are not edited — those posts
belong to their authors. The `@username` token in such posts now
resolves to the redaction marker on display. This is intentional:
editing other users' content to scrub a redacted user's name
would itself be a deletion of someone else's record.

## 3. Retention archive

Originals — the original `users` row, content body version rows
that were tombstoned, the prior values of redacted graph property
layers, and any tombstoned media attachments — are written to the
[retention archive](../primitive/retention-archive.md) with a
per-row legal hold appropriate to the data:

- **Ordinary profile PII** (display name, bio, avatar, website,
  layered username history) — typically a short or zero hold,
  expirable on user request per DSGVO storage minimization.
- **Content tied to financial transactions** (forthcoming with
  the economics primitive) — 10-year hold under German tax-record
  retention law.

Hold values are set at redaction time. The archive itself defines
the polymorphic schema, the per-row legal-hold-then-hard-delete
mechanism, and the `legal_admin` access path that makes archived
content available to legal authorities under compulsion. See
[retention-archive.md](../primitive/retention-archive.md) for the
mechanism.

## 4. The user self-service trigger

The user-initiated path is the only trigger spec'd here. Future
triggers reuse the same redaction scope (§1) and archive (§3);
only their authorization differs.

1. **Request.** The user invokes "delete my account" from the
   client. The API records the request, including whether the
   user opted into content-level redaction, and emails a
   confirmation link.
2. **Confirmation.** The user confirms via the emailed link. The
   API records the confirmed request with a 7-day deadline.
3. **Grace period.** For 7 days, the request is reversible — the
   user can cancel from any logged-in session, restoring full
   account state. Nothing on the graph or in Postgres is
   redacted yet; the request is a pending intent.
4. **Execution.** At deadline, the redaction action runs (§5).
   Identity-level redaction is automatic. Content-level redaction
   is included only if the user opted in during request or
   confirmation.
5. **Irreversibility.** After execution, the user's PII is in the
   archive and inaccessible to public surfaces. The archive's
   hold expiry will eventually destroy it. There is **no restore
   path** post-execution — the platform commits to the redaction
   once executed.

The grace period exists for the same reason GDPR confirmation
patterns exist: account deletion is destructive and easy to
trigger by mistake, by client bug, or by a compromised session.
The window is short enough that public surfaces clear quickly,
long enough that an affected user typically notices.

## 5. Write ordering across stores

Account deletion writes to three places: the retention archive
(Postgres), the graph (Memgraph), and the public Postgres display
tables. The order matters for crash safety:

1. **Archive first.** Write the original PII to the retention
   archive. Idempotent — the same request can be retried without
   producing duplicates (key on
   `(original_id, original_type, redacted_by)`).
2. **Graph redaction.** Apply the [layers.md §5](../primitive/layers.md)
   redaction markers to the relevant property layers on the User
   node (and, for content-level redaction, on any node-property
   layers that carry user-input strings).
3. **Postgres tombstone.** Write the tombstone version rows for
   the user profile and (if content-level was opted into) the
   content bodies and their media attachments.

Each step is retryable independently. A crash mid-flow leaves
the system in a safe state: PII is already preserved in the
archive; the graph and public Postgres state may be partially
or fully redacted, but never lose data. A reconciler re-runs any
incomplete redaction from the request record.

## 6. Interaction with moderation

[Moderation](moderation.md) and account deletion both invoke the
[layers.md §5](../primitive/layers.md) redaction mechanism but differ in
authorization, scope, and archive treatment:

|                | Moderation (illegal)                              | Account deletion                                  |
|----------------|---------------------------------------------------|---------------------------------------------------|
| Authorization  | Network governance + mod gate                     | User self-service (with grace)                    |
| Scope          | One specific content node                         | User profile + (opt-in) all authored content      |
| Archive hold   | Per case — evidence retention or immediate destroy | Per row — short for PII, 10y for financial data |
| Initiator      | Any active Network member                         | The account owner                                 |

The two paths run independently. A user under active moderation
can still request account deletion. Conversely, illegal-content
takedowns of a redacted user's content proceed normally — the
content body is in the retention archive, and a moderator acting
on a court order can request removal of the archive copy as
well, satisfying the destruction obligation that overrides
ordinary retention for illegal content specifically.

## What this doc is not

- **Not the redaction mechanism.** In-place layer-marker and
  Postgres-tombstone semantics live in
  [layers.md §5](../primitive/layers.md).
- **Not the moderation authorization.** Community-driven
  classification of illegal content lives in
  [moderation.md](moderation.md). This doc is a separate
  authorization path that happens to invoke the same mechanism.
- **Not the archive schema.** Concrete column types, indexes,
  migrations, the polymorphic JSONB shape, and the
  `legal_admin` role's auth model live in
  [data-model.md](../implementation/data-model.md).
- **Not the future triggers.** Court order, next-of-kin
  (§ 1922 BGB), and network-admin emergency action are listed
  here as planned reusers of the redaction scope; each warrants
  its own authorization spec when designed.
