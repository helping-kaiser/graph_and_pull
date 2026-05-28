# Economics & token — working notes

WIP scratchpad for [Q20](open-questions.md#q20--economics-primitive-distribution-ledger-home-vocabulary-anchor).
**Nothing here is canonical** — the canonical landing pads will be
new files under `docs/primitive/` and `docs/implementation/` once
decisions stabilize. This doc gets deleted when those files exist.

Decisions marked `[settled]` are explicit user choices. `[proposal]`
items are sketches awaiting the user's call.

---

## How to use this file

This is the working doc for the Q20 economics design pass. It lives
on the long-lived branch `jakob/economics/design`. No PR into main
until the design is fully settled.

**Each session should:**

1. Re-read this file in full at the start. It replaces the previous
   session's context.
2. Pick the next item from "Discussion order" below. Don't jump
   ahead unless a prerequisite is naturally settled along the way.
   The user will flag if a topic needs an earlier resolution first.
3. Discuss with the user: lay out options + trade-offs; let the
   user decide. Per [CLAUDE.md](../CLAUDE.md), never make design
   decisions autonomously.
4. Update this file with the outcome — move resolved items into
   "Settled decisions", remove or supersede stale `[proposal]`
   sketches, add new sub-questions surfaced by the resolution.
5. Commit + push on this branch; end the session. Each session
   produces one commit on this branch.

**After the design is fully settled:**

- Open a PR merging this scratchpad into main.
- Then create separate branches for each canonical landing pad
  (see "Files this will eventually touch" at the bottom). Don't
  author canonical primitive / implementation docs from this
  design branch; this branch produces only the scratch.

## Discussion order

1. **Token issuance model** — pass-through vs mint-on-hit vs
   calendar-mint-with-decay. *In progress; next session resumes
   here.* Analysis under *A — Token shape* below.
2. Campaign expiry behavior (refund / pro-rata / mixed).
3. Goal-hit detection cadence (continuous / epoch-snapshot /
   claim-on-hit).
4. Attribution math concretization (Shapley specifics; conduit
   credit formula; cut-off enforcement).
5. Action gating specifics (which actions; quota shapes; CGT
   prices; how the soft-quota threshold gets set).
6. Wallet onboarding & claim-escrow policy.
7. Marketplace + infrastructure primitive scoping — in this design
   pass or deferred to a follow-up workstream?
8. Q19 stake-gated quorum reopen (now that a token exists).
9. Q16 `S(t)` input candidates (token-related or unrelated).
10. Authoring plan: which canonical docs in which order; what
    splits between `economics.md` / `token.md` / `ledger.md`.

## Next session pickup

**Topic 1: Token issuance model.** Three options on the table —
**pass-through** (fixed supply, no mint), **mint-on-hit** (campaign-
driven mint), and **calendar-mint with decay** (peer-network-style
supply curve, distributed via campaign attribution not per-activity).
Full analysis preserved under *A — Token shape* below. User has not
yet decided.

---

## Guiding principles surfaced in discussion

- **Fair > cheap.** Pick the cheapest only among equally-fair
  options.
- **Public auditability** of money flows is a design north star —
  vendors and buyers can't silently scam each other when contracts
  + payments are graph-visible.
- **Maximize free user actions; price only at the margins.**
  Gating exists to stop spam and fund infrastructure, not to
  extract from normal use.
- **Costs are explicit, not hidden.** "If it's free, you're the
  product" is a real observation — compute, storage, and bandwidth
  cost real resources; someone always pays. CoGra's answer is to
  make the payment relationship visible (host edges, transfer
  edges) instead of monetizing user data. Self-hosters pay nothing
  to the network; hosted users pay their host; net-negative users
  (consume more than they contribute) are sponsored explicitly by
  whoever values them, or pay themselves. Data is free for all;
  what gets paid for is *service delivery*, not data access.
- **Early-holder upside comes from demand growth, not from
  rewarding squatters.** Token price rises if advertiser demand
  outpaces fixed or slow-growing supply; "joined early and held"
  benefits from the rise without a mechanism that pays inactive
  early users on a calendar.
- **Per-action distribution is the anti-pattern, not calendar-mint
  per se.** The peer-network spec rewarded users per-activity
  (likes, posts, comments), which bots beat humans at. That
  mechanism is rejected. The spec's supply curve (fixed daily mint
  with annual decay) is a separate question and remains a candidate
  issuance model — see *A — Token shape* below.

---

## Settled decisions

- **Native CGT token, on-chain.** `[settled]` Advertisers buy CGT on
  a DEX and fund campaigns in CGT. Ledger is the chain.
- **Campaign success metric = `h_anchor(target)`.** `[settled]`
  Single anchor node. The anchor's h already aggregates her
  cluster's paths — raising it is reaching her cluster.
- **Treasury share = 2%.** `[settled]` Other 98% distributed to
  contributors.
- **Fairness over cheapness.** `[settled]` Pick the cheapest only
  among equally-fair options.
- **Ledger home (Q20.2) = the chain.** `[settled]` Postgres holds
  campaign metadata (target, anchor, goal, budget, window, status).
  Memgraph holds the graph including transfer edges. No CoGra-side
  balance store.
- **Transfer edges: recorded, not feed-traversable.** `[settled,
  with flex]` Leaning non-traversable for ranking; user noted some
  merit to making them traversable. Reopen if marketplace work
  creates pressure.
- **Action gating: reluctantly yes, scoped.** `[settled]` Used for
  (a) anti-spam on high-fanout actions and (b) compensating
  infrastructure providers for hosted users. Default posture:
  maximize free actions; price only at the margins.

---

## Open sub-questions

### A — Token shape

- **Chain choice.** Need cheap settlement, DEX composability,
  no single-operator risk. Candidates: ETH L2 (Base / Optimism /
  Arbitrum), Solana, custom appchain. `[proposal]` decide once the
  primitive is written; chain choice is implementation.
- **Token issuance model.** Three live options to evaluate against
  each other:
  - **(a) Pass-through, fixed supply** `[proposal]`. Total supply
    set at launch; never inflates. Advertisers buy CGT on a DEX,
    deposit to fund a campaign, contributors claim at goal-hit.
    Treasury's 2% is a slice of each deposit. Early-holder upside
    comes purely from demand growth against fixed supply.
    - **Pros**: one supply curve to design (initial only); no
      ongoing-mint governance question; cleanest ledger story
      (chain holds balances, CoGra signs Merkle roots, no
      mint-side state).
    - **Cons**: initial premine must cover lifetime liquidity
      (under-issuance → DEX slippage); no protocol-level subsidy
      lever; weaker deflationary narrative (price growth only,
      no supply story).
  - **(b) Mint-on-hit (campaign-driven)** `[proposal]`. New CGT
    minted on each campaign goal-hit, distributed to contributors;
    advertiser's deposit goes to a burn or sink. Mint rate tied to
    campaign activity, not calendar.
    - **Pros**: protocol can subsidize specific behaviors (hosts,
      conduits, etc.) without taking from advertiser budget;
      initial premine can be small.
    - **Cons**: two supply curves to balance (initial + ongoing
      mint rate); needs governance for the mint rate; more
      complex ledger story (chain must distinguish "transferred"
      from "minted" CGT for the claim contract).
  - **(c) Calendar-mint with decay** `[proposal — peer-network
    supply curve, anti-sybil distribution]`. Fixed daily mint
    (e.g. 5000 CGT/day) decaying 10%/year, totaling ~18M lifetime.
    Distribution mechanism is *not* per-activity (the rejected
    peer-network mechanism) but campaign-attribution: each day's
    mint accumulates in a pool drawn against campaign closes,
    or directly tops up campaign payouts.
    - **Pros**: clean deflationary supply narrative (the "joined
      early and got rich" story sells naturally without rewarding
      squatters); fixed lifetime supply with predictable curve;
      protocol-level reward pool independent of advertiser flow.
    - **Cons**: requires queueing/accumulation logic if a day has
      no qualifying campaigns; mismatch handling if campaign
      payouts exceed available mint; one extra knob (the decay
      rate) that needs justification.
  - **Trade-off summary**: (a) is simplest and leans on demand
    growth alone; (c) keeps the deflationary curve narrative
    without per-activity sybil risk; (b) is most flexible but
    adds two governance surfaces. None obviously dominates yet.
- **Initial distribution / premine.** Open. Team allocation?
  Airdrop to alpha users? Liquidity-bootstrap auction?
- **Treasury accrual currency.** `[proposal]` Treasury takes CGT
  (campaigns are CGT-denominated, no conversion needed). Treasury
  free to market-sell at its discretion.

### B — Campaign primitive

- **Who can be an anchor?** `[proposal]` Any actor node. No consent
  required — severance is the implicit opt-out (anchor severs
  advertiser → paths through anchor go to 0 → campaign can't
  succeed via that anchor).
- **Forbidden configurations.** `[proposal]`
  - `anchor == target` (degenerate, `h(self)` undefined).
  - "Goal already met at campaign start" (immediate honey-pot).
  - Negative-h campaigns (paying to lower someone's `h(t)`) — would
    weaponize severance and corrupt the safety primitive. Campaigns
    are increase-only.
- **Campaign window.** `[proposal]` Expiry required; open-ended
  campaigns become honey-pots. On expiry with goal unmet: refund
  advertiser minus small treasury fee, OR pro-rata partial payout
  to contributors — **needs user call**.
- **Goal-hit detection.** `h(t)` is sort-time; "did we hit the goal"
  needs a defined evaluation point. `[proposal]` per-epoch snapshot
  (daily?). First snapshot that meets goal closes the campaign.
- **Concurrent campaigns.** `[proposal]` Linear composition: each
  campaign computes attribution independently against its own
  anchor / target / window. A single edge can contribute to many.

### C — Attribution math

- **Target shape**: Shapley-style marginal contribution. Each
  contributor's payout = the counterfactual drop in achieved
  `h_anchor(target)` if that contributor's edges (or the
  contributor's node, for conduits) were removed.
- **What counts as a "contribution".**
  - (a) New edges (actor or structural) added during the campaign
    window.
  - (b) New content nodes whose paths from anchor toward target
    raised `h`.
  - (c) Conduit nodes on the lifted paths — credited by their
    existing position even if they added nothing during the window.
- **Why Billie gets the largest share.** Conduit credit. Her
  counterfactual removal collapses many paths; the Shapley
  calculation captures this without any special rule.
- **Cut-off discipline.** Only edges/nodes appearing *after*
  `campaign_start_ts` count for the "new path" component. Conduit
  credit ignores cut-off (it's about pre-existing position).
- **Cost.** Exact Shapley on the relevant subgraph is expensive but
  bounded — the subgraph is "nodes/edges on paths from anchor to
  target that lifted `h`", not the whole graph. `[proposal]` Compute
  once at goal-hit, not per-impression.

### D — Ledger & on-chain mechanics

- Chain is the ledger; CoGra signs payout Merkle roots after each
  campaign closes; contributors claim from a payout contract.
- Postgres holds campaign metadata; Memgraph holds graph including
  transfer edges; chain holds balances and claim state.
- `[proposal]` Campaign object lives in Postgres as
  `(id, advertiser_id, target_node_id, anchor_node_id, goal_metric,
  budget_cgt, start_ts, end_ts, status, merkle_root_at_close)`.
- `[proposal]` Per-user wallet linkage is a `WalletAddress` system
  property on User / Collective nodes — not feed-traversable.

### E — Transfer edges & marketplace future

- **Edge type `:TRANSFERS`** (working name). Source = sender actor.
  Target = receiver actor. Tensor `(0, 0)` actor dims (no ranking
  contribution). System dimensions carry: amount, currency, on-chain
  tx hash.
- **System-dimension slot needs formalization.** CLAUDE.md mentions
  "2 dimensions + system dimensions" but
  [edges.md](primitive/edges.md) does not yet codify how system
  dimensions look. Need a small primitive addition to host
  `:TRANSFERS` cleanly.
- **Future expansion (out of scope for first economics PR).**
  - Marketplace: extend [items.md](instances/items.md) with price
    + listing semantics.
  - Contracts: graph-native escrow / multi-step agreements.
  - Proof-of-fulfillment edges (or junction nodes, user's hint)
    between contract and payment.
  - Public auditability — vendors and buyers can't silently scam
    each other when contract + payment edges are all visible.

### F — Action gating & infra payment

- **The pull-marketing spam attack.** Actor spams posts with
  `:REFERENCES` to Teufel during a Teufel campaign to harvest
  budget. Brakes:
  1. Existing severance — community severs spammer, paths collapse.
  2. Fanout-budget on `:REFERENCES` ([edges.md §2](primitive/edges.md)) —
     caps how many references one post can carry.
  3. `[proposal]` Soft per-day quota on `:REFERENCES` creation and
     post creation; CGT cost only above the quota. Free for normal
     users, expensive for spammers.
- **Infra payment.** Any node hosting its own data pays nothing.
  Hosted users pay their host. Host-of-record is a graph-recorded
  property.
  - `[proposal]` Hosts set their own prices; hosting marketplace
    lives downstream of the marketplace primitive above.
- **Posture.** User-free-actions are the default. CGT cost is *only*
  at the margins (exceeding soft limits, paying a chosen host). Keep
  the "you are the product" surface closed.

---

## Cross-cutting obstacles

- **No-AI rule applies.** Attribution math is graph-computed, not
  learned. Shapley on the graph is fine; ML "fair share" is not.
- **Edge tensor uniformity.** `:TRANSFERS` must fit the
  `(dim1, dim2) + system` shape. `(0, 0)` actor dims + a
  system-dimension transfer payload is the cleanest fit, but
  requires formalizing the system-dimension slot in `edges.md`.
- **Q19 (stake-gated quorum) reopen.** Now that a real token
  exists, stake gating is reachable again. Risks: contradicts
  "anyone can fork" if excessive; concentrates power in early
  holders. `[proposal]` Note in `governance.md` as a follow-up,
  don't bundle into the first economics PR.
- **Q16 (`S(t)` derivation).** Token balance as input to `S(t)` →
  reject candidate, gives wealthy users intrinsic ranking
  advantage and corrupts the graph-is-truth principle. Token
  *activity* (recent transfers, campaign participation) is a
  different question and probably also out. **User call needed.**
- **Wallet onboarding UX.** Every CoGra user needs a wallet to
  receive payouts. Not a primitive question, but flag early —
  payouts to users without wallets accumulate to a claim escrow
  with some expiry policy.

---

## Deliberately deferred

- Marketplace primitive (items + listings + contracts).
- Hosted-user infra marketplace.
- Stake-gated governance quorum (Q19 reopen).
- Specific chain choice and mint schedule (implementation).
- Wallet UX / claim escrow mechanism.

---

## Files this will eventually touch

- **New** `docs/primitive/economics.md` — pull-marketing definition,
  campaign object, h-based goal, attribution math, treasury split.
  The "pull marketing" vocabulary anchor.
- **New** `docs/primitive/token.md` — CGT semantics, on-chain model,
  mint schedule. May merge into `economics.md` if small.
- **New** `docs/implementation/ledger.md` — chain integration,
  Merkle-claim mechanics, Postgres campaign-metadata schema.
- **Update** [docs/primitive/edges.md](primitive/edges.md) —
  `:TRANSFERS` edge + formalize the system-dimension slot.
- **Update** [docs/primitive/authorship.md](primitive/authorship.md) —
  cross-link to economics.md.
- **Update** [docs/instances/collectives.md](instances/collectives.md) —
  advertiser role.
- **Update** [docs/primitive/governance.md](primitive/governance.md) —
  Q19 reopen note.
- **Update** README.md and CONTRIBUTING.md — point "pull marketing"
  language at the new primitive.
- **Update** [docs/open-questions.md](open-questions.md) — close
  Q20, follow-ups on Q16 and Q19.
