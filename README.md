# CoGra

**CoGra** (Content Graph) is the **graph-architecture exploration**
for **Peer Network**'s next evolution — a social media platform
where the social graph and explicit user interactions drive feed
ranking, replacing AI content algorithms with a transparent, user-
controlled system.

The current Peer Network platform works like Instagram. This repo
branches off from main Peer Network development to design and
prototype the graph network that will succeed it; it is a multi-
year effort, not a short throwaway exploration.

## Mission

Decentralize the power of social media. The goal is not to become
the next Instagram/X/TikTok with a graph bolted on — it is to
shift power from social-media companies to users, where weight
and ranking are owned by users themselves. Every design decision
must resist re-centralization.

## What CoGra commits to

**No AI in the feed.** Ranking is computed from each viewing user's own
position in the graph and the weighted edges they create through
explicit interactions. There are no learned models and no
popularity amplifiers.

**Directional edges only.** What you see is shaped by your
outgoing edges, never by who points at you. Bot clusters and
unwanted attention can't insert themselves into your feed by
liking your content.

**[Append-only](docs/primitive/layers.md#append-only-vocabulary)
history.** Edges and node properties are layered, not
overwritten. The only carve-out is in-place redaction of illegal
content, and even that leaves a visible trace. Transparency and
auditability over convenience.

**Governance, not admin escape hatches.** Redactions and policy
changes run through community votes on the graph, with weights
and thresholds visible. There is no admin override —
[external demands enter as ordinary Proposals](docs/primitive/governance.md#external-demands-enter-as-proposals),
leaving an auditable trail rather than a silent edit.

**Fair economics.** Ad revenue distributes across the economic
landscape of the graph. Bot clusters earn nothing because real
users never point toward them. Pull marketing, not push.

**User comes first.** Users choose what they see, including ads.
No amount of money changes that. No one forces their way into
another user's feed.

**Community choices stay local.** What a community decides —
including severing ties — affects only the severing community's
own outbound paths, not the severed party's other neighbours.
Viewing users whose own paths pass through the severing community
see their feeds reshaped, and each chooses whether to cascade the
severance further. The spread is by choice, not by propagation.

**Fully public graph; no account needed to read.** Everything in
the graph is visible without signing in. Accounts gate
participation, not visibility — anyone can browse and evaluate
before committing to an identity.

**Privacy is per-content, not per-topology.** Chats and messages
can be end-to-end encrypted; the social fabric — who is connected
to whom — is intentionally public. CoGra protects what travels
along the graph, not the graph itself.

**Transparency, openness, freedom of the mind.** The system is a
visible, auditable graph. The codebase is fully open source —
forking, self-hosting, and running disconnected graphs are
architecturally supported. Nothing in the system rewards outrage,
runs dark patterns, or infers stances the user did not explicitly
take.

## Quick start

```bash
make run    # first-time: init + start DBs + migrate + start API
make dev    # returning: start DBs + migrate + start API
```

Full make-target list, environment variables, and the dev
workflow live in
[docs/implementation/development.md](docs/implementation/development.md).

## Where to go next

- **Design docs** — [docs/README.md](docs/README.md) (start here for the
  graph model, instance docs, and implementation specs).
- **Project rules and workflow** — [CONTRIBUTING.md](CONTRIBUTING.md).
- **AI-assistant guidance** — [CLAUDE.md](CLAUDE.md).
- **Build commands** — [Makefile](Makefile) and
  [docs/implementation/development.md](docs/implementation/development.md).
