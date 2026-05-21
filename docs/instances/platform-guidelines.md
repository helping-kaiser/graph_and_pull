# Platform guidelines

The normative document the Network references when classifying
content via the [moderation](moderation.md) primitive. Three
buckets — `illegal`, `sensitive`, `normal` — backed by the same
Proposal mechanism that powers the rest of the platform: the
guidelines themselves are amendable by the Network at any time.

This doc is the **canonical text**. The graph stores only the
current version number and a content hash on the
[`:Network`](../implementation/graph-data-model.md) singleton —
the document itself lives here, in the repo.

## 1. The three buckets

The classification a Network member assigns when authoring or
voting on a moderation Proposal is one of `illegal`, `sensitive`,
or `normal`. The bucket *meanings* and the *behavioural
consequences* of each (redaction cascade, viewer filter,
reversibility) are pinned at the primitive level — see
[nodes.md "Universal: `moderation_status`"](../primitive/nodes.md#universal-moderation_status).
What follows here is the platform policy: which content the
Network puts in each bucket. The Network amends these lists via
§3 as the platform evolves.

### `illegal`

Starter list — adapted from the conventions of established
public platforms:

- **Child sexual abuse material (CSAM).** Always, everywhere.
  Reported to authorities and scheduled for hard-delete on
  archive entry — the legal hold is "report and destroy", not
  "retain for prosecution" (per
  [retention-archive.md](../primitive/retention-archive.md)).
- **Credible threats of violence** against a person, group, or
  identifiable target.
- **Incitement to violence** — calls to commit violent acts,
  glorification of mass-casualty events tied to designated
  terrorist organizations, recruitment for the same.
- **Non-consensual intimate imagery** (NCII / "revenge porn") —
  sexual or nude imagery shared without the subject's consent.
- **Doxxing** — unauthorized publication of private personal
  information (home address, phone, government ID, financial
  account numbers) outing an identifiable individual.
- **Trafficking** — content offering, soliciting, or coordinating
  trafficking in humans, controlled substances at scale,
  weapons-of-war, or trafficked wildlife.
- **Fraud and scams** — phishing, account-takeover marketplaces,
  fraudulent financial schemes, sale of stolen credentials.
- **Sale of strictly-controlled goods** — schedule-I narcotics,
  unregistered firearms, regulated chemicals outside legal
  channels. (Legal grey-zone goods — e.g. firearms in
  jurisdictions where private sale is permitted — are out of
  scope for `illegal` and may or may not be `sensitive`
  depending on context.)
- **Copyright infringement at scale** — wholesale republication
  of copyrighted works against an explicit removal request.

### `sensitive`

Starter list:

- **Graphic violence** — real-world injury, gore, accident
  footage, war footage with visible casualties.
- **Adult nudity and sexual content** that is consensual, lawful,
  and clearly intended for adult audiences.
- **Self-harm and suicide** — depictions, methods, in-progress
  imagery; non-supportive discussion.
- **Disturbing medical imagery** — surgery, severe injury,
  pathology.
- **Animal cruelty depictions** — including hunting and
  slaughter imagery that is not itself illegal.
- **Drug use depictions** — recreational use of legal or illegal
  substances depicted approvingly.
- **Strongly disturbing material** that doesn't fit a category
  above but a reasonable Network member would expect a viewing user
  filter to apply (e.g. detailed descriptions of torture).

### `normal`

Not a list. The default state for content that hasn't been
classified into either of the above buckets.

## 2. Jurisdiction

CoGra is one Network per instance. Each instance's Network sets
its own normative line via the amendment procedure (§3), and
will arrive at different rest-points depending on its
jurisdiction and community.

The list above is a starting point for the central instance. A
fork operating under a different legal regime is expected to
amend in either direction — adding categories required locally
(e.g. specific political speech restrictions) or removing
categories not applicable.

The `illegal` bucket is **not** a literal application of any
single jurisdiction's law. It is a community standard. A piece
of content can be lawful in some jurisdictions and still
classified `illegal` by the Network if the Network's normative
judgment lands there; conversely, content that is unlawful in
some jurisdictions may remain `normal` if the Network has not
classified it.

The legal-hold disposition for `illegal` content
([retention-archive.md](../primitive/retention-archive.md))
*does* track jurisdictional law — that is a per-row decision
made by the moderator and legal admin at redaction time, separate
from the classification decision.

## 3. Amendment procedure

The guidelines are amendable via the same Proposal primitive that
governs everything else on the platform.

**Subject.** Two `:Network` properties move together as the
canonical pointer to a guidelines version:

- `Network.guidelines_version` — monotonic integer, incremented
  by 1 on each amendment.
- `Network.guidelines_hash` — SHA-256 hex digest of the canonical
  document bytes at that version.

A guidelines amendment is a Proposal (or pair of Proposals) that
sets these two properties to the new version's values.

**Eligibility.** All active Network members
([network.md](../primitive/network.md)).

**Vote shape.** Shape B from the voter's User node, same as
moderation Proposals
([moderation.md §4](moderation.md#4-eligibility-weights-thresholds)).

**Threshold.**

| Action | Quorum property | Pass-threshold property | Mod gate |
|---|---|---|---|
| Amend guidelines | `Network.guidelines_change_quorum` (default 5%) | `Network.guidelines_change_threshold` (default ≥2/3) | ≥1 mod positive |

The defaults are slightly higher than `illegal` classification
(2% / ≥2/3) because guideline changes shift the normative frame
for *all future* moderation, not a single piece of content. The
`guidelines_change_*` thresholds themselves fall in the critical
bucket of [network.md §11](../primitive/network.md#11-amending-network-parameters) — amending
them needs the supermajority that protects platform-wide
governance.

**Mod gate.** Same as moderation classifications — at least one
moderator's positive vote is required. Same bot-defense
reasoning as
[governance.md §7](../primitive/governance.md#7-the-mod-gate).

**Drafting and discussion.** The Proposal carries the new version
number and hash. The actual text — the new version's diff against
the previous one — is published off-graph (e.g. the repo's pull
request) prior to the vote so members can review what they are
voting on. Voters who cast `+1` without reviewing the linked
text are operating on the same normative honor system as
moderators voting on encrypted ChatMessages
([moderation.md §5](moderation.md#5-scope)) — addressable through the
same Proposal mechanism applied to that user's role or
participation.

## 4. URL handling

The graph deliberately does **not** store a URL pointing at this
document. Different instances serve under different domains; the
graph stores only `guidelines_version` + `guidelines_hash`, and
each instance's frontend constructs the canonical URL from its
own domain configuration (`https://<instance-domain>/guidelines`
or whatever the instance's deployment chooses).

The hash is the integrity anchor: a client can verify the served
document matches the version the Network ratified, regardless of
how the URL is composed.

## What this doc is not

- **Not the moderation mechanism.** Reports, voting, the mod
  gate, the cascade — all in
  [moderation.md](moderation.md).
- **Not the legal-hold disposition.** Per-row legal hold for
  `illegal` originals is in
  [retention-archive.md](../primitive/retention-archive.md).
- **Not a substitute for jurisdictional legal review.** The
  Network's classification is a community standard; a legal
  admin still reviews legal-hold disposition per row.
- **Not exhaustive.** The starter lists in §1 are seed text. The
  Network amends them via §3 as the platform evolves.
