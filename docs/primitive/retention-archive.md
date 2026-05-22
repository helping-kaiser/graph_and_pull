# Retention archive

Some redactions destroy the original; others must preserve it for
legal purposes. The retention archive is the platform's universal
disposition for "redact from public view but retain for statutory
obligations" cases.

Both current authorization paths use it:

- **Illegal-content cascades** ([moderation](../instances/moderation.md))
  may need to retain the original as evidence for prosecution, or
  may be required to destroy it (e.g., content illegal to possess
  at all). The hold value is set per case at redaction time.
- **Account deletion** ([account-deletion](../instances/account-deletion.md))
  retains identity and (opt-in) content originals. The hold value
  reflects the applicable statutory retention period in the
  jurisdiction the instance operates under — examples: ~10 years
  for content tied to financial transactions in many tax regimes
  (e.g., § 147 AO in Germany, IRS record-retention requirements
  in the US, similar provisions elsewhere); often shorter for
  ordinary PII under data-protection storage-minimization rules
  (GDPR/DSGVO in the EU, comparable laws elsewhere). The
  specific retention period is jurisdiction-dependent, not
  pinned to any one country.

The archive's hard-delete-on-hold-expiry is the **only point in
the system where data is genuinely removed** (not the only
mechanism — the redaction itself is the mechanism, see
[layers.md §5](layers.md#5-deletion-policy); the archive entry's eventual
hard-delete is the post-redaction statutory end state).
[layers.md §5](layers.md#5-deletion-policy)'s "no silent deletion" rule still
holds: the redaction leaves an auditable mark on public surfaces
that does not change at hard-delete time; the archive entry's
existence and its destruction are both private and never visible
on public surfaces.

## 1. Polymorphic shape

One Postgres table, one row per redacted entity (user profile
snapshot, post body, comment body, chat-message body, media
attachment, etc.):

- `original_id` + `original_type` identify what was redacted.
- `original_data` is a JSONB blob of the original row's contents
  — schema-on-read, so the archive does not migrate when source
  schemas evolve.
- `redaction_reason` records the trigger
  (`'illegal-content-cascade'`, `'user-self-service'`,
  `'court-order'`, etc.).
- `redacted_by` is the User UUID of the actor who initiated the
  redaction.
- `redacted_at` is the timestamp.
- `legal_hold_until` is the per-row deadline.

Concrete column types, indexes, and migration mechanics belong in
[data-model.md](../implementation/data-model.md). This doc fixes
the shape: one polymorphic table; per-row hold; hard-delete on
expiry; access-controlled.

## 2. Per-row legal hold

Different content types and authorization paths set different
`legal_hold_until` values:

- **Illegal content.** All redacted originals from
  illegal-classification cascades land in the archive
  automatically at threshold-cross — the cascade does not block
  on a hold decision. `legal_admin` (see §4 — a member of the
  host's operations team, not a graph role) reviews each case
  asynchronously and sets `legal_hold_until` per the relevant
  law: some content is retained for prosecution (terror financing
  evidence, fraud records); other content is illegal to retain
  at all (e.g., CSAM) — `legal_admin` schedules immediate
  hard-delete (`legal_hold_until = now()`) and reports to
  authorities. Until reviewed, the row sits with a placeholder
  hold awaiting `legal_admin` action.
- **Account deletion.** Statutory retention for tax / economic
  records (often ~10 years for content tied to financial
  transactions, varies by jurisdiction); data-protection
  storage minimization for ordinary PII (often a short or zero
  hold, expirable on user request — GDPR/DSGVO and equivalents
  elsewhere).
- **Court orders.** As ordered by the court.

The archive table holds the original; whether and when it is
destroyed depends on the per-row deadline.

## 3. Statutory hard-delete

A scheduled job hard-deletes rows where
`legal_hold_until < now()` and no other statute extends the hold.

This is the **only path in the system where data is genuinely
removed**. [layers.md §5](layers.md#5-deletion-policy) declares "no silent deletion,
ever"; this is the explicit, statutorily required exception.

The exception is honest because:

- The redaction itself leaves a public mark (in-place layer
  marker, Postgres tombstone version row) that does not change at
  hard-delete time.
- The archive entry's existence is private — its destruction does
  not erase any public-facing history.
- DSGVO Art. 5(1)(e) (storage minimization) and similar
  provisions actively *require* destruction once the obligation
  expires; keeping retained PII indefinitely would itself be a
  violation.

The graph and public Postgres surfaces never see the deletion —
they have shown the redaction marker since the redaction itself.

## 4. Access path

The archive is **not** a graph-visible surface. It plays no role
in feed ranking, traversal, or any normal API path.

`legal_admin` is a **person on the host's operations team**, not
a graph role, not a `network_role` value, and not something the
graph or governance flow appoints. We use the name as shorthand
for the human(s) whose job is to act on cases the moderation flow
has already removed from the network: case review, setting per-row
hold values per the relevant law, reporting illegal content to
authorities, surfacing archive contents under compulsion (court
order, prosecutor request, tax-audit subpoena), and scheduling
statutory hard-delete.

The work is **post-redaction**. By the time `legal_admin`
touches a case, the redaction cascade has already removed the
content from the live graph and public Postgres surfaces;
`legal_admin` has no path back into the live graph and no part
in deciding what gets redacted in the first place.

The invariants are deliberately narrow:

- **No graph reach.** `legal_admin` cannot author Proposals,
  classify content, or otherwise act on the live graph.
- **No moderation authority.** The graph's "no admin override"
  rule still holds — illegal-content classification runs through
  the [moderation](../instances/moderation.md) flow before
  `legal_admin` ever sees the case.
- **No write access to the archive itself.** Archive rows are
  inserted by the redaction cascade; `legal_admin` only reads
  them (and triggers the per-row hard-delete on hold expiry).
- **No power to set arbitrary hold values.** Per-row holds are
  determined by the relevant law per case, not by `legal_admin`
  preference.

Widening this access would turn a compliance store into a
surveillance surface. Concrete access-control mechanics,
audit-logging, and the authentication shape under which an
operations-team member reaches the archive belong in
[data-model.md](../implementation/data-model.md) — whatever shape
that takes (Postgres role, off-instance tooling, a dedicated
admin app), it is a host-operations concern and the graph and
public Postgres surfaces never see it.

## What this doc is not

- **Not the redaction mechanism.** [layers.md §5](layers.md#5-deletion-policy)
  defines in-place layer markers and Postgres tombstone
  semantics. The archive is what happens to the original *after*
  redaction.
- **Not the authorization paths.** Who decides to redact — and
  what hold value to set — runs through the relevant instance
  docs ([moderation](../instances/moderation.md),
  [account-deletion](../instances/account-deletion.md)). Each
  maintains its own scope and hold-rule conventions.
- **Not the retention schedule.** "How long should X be held"
  comes from law, not from this doc; per-row hold values are
  determined at redaction time per case.
