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

1. **Token issuance model** — *substantially settled: decaying
   calendar mint, no fresh premine, peer-token percentage
   carry-forward, conservation equation and γ=5% bot-loss locked
   in. Outstanding: POL (protocol-owned liquidity) focused
   session — calendar mint into protocol's LP position.*

   *(i)/(W)/(X)/(Y)/(Z) all eliminated — see A.*
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

**Topic 1: Token issuance model — POL focused session next.**
Marketing-flow math locked. All non-POL candidates for non-burn
distribution eliminated; POL is the sole surviving supply
mechanism. See *A — Token shape* for the conservation equation,
worked examples, gaming-attack audit, and elimination reasons.

POL focused session: how calendar mint deposits into protocol-
owned LP — trigger cadence, one-sided vs. swap-pair, LP fee/
share disposition, chain coupling, IL posture, and mint schedule
sized against burn rate. Initial pre-check: POL appears self-
deal-immune by construction — no graph-visible extraction
surface.

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
  *distribution* mechanism is rejected. The spec's *supply curve*
  (fixed daily mint with annual decay) is the chosen issuance
  shape — see *A — Token shape*.

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
- **Issuance shape = decaying calendar mint, asymptotic fixed
  supply.** `[settled]` Peer-network supply curve (fixed daily
  mint with ~10%/year decay, ~18M lifetime). Exact parameters
  TBD at `token.md` authoring (possibly a milder variant with
  smaller starting daily mint and gentler decay).
- **No fresh premine; initial CGT carries forward proportionally
  from existing peer-token holdings.** `[settled]` Existing
  peer-token holders (company, founders, alpha users) keep their
  *percentage* of the prior token state, translated into CGT —
  not unit-for-unit. Bootstraps initial LP liquidity and respects
  pre-existing holder expectations without creating new
  concentration in designated parties.
- **Two flows: marketing flow (redistribution) + calendar-mint
  top-up (new supply).** `[settled, direction]` Marketing flow
  routes existing CGT from advertiser to contributors. Calendar-
  mint top-up is *a* new-supply path into the system.
- **Strict cap: `contributor_payout < deposit` always.** `[settled]`
  Non-strict version is gameable by advertiser+contributor self-
  deal coalitions (the 10k-campaign-target-friend-with-tiny-h-bump
  attack). Strict version makes such collusion strictly
  unprofitable.
- **Conservation equation with γ=5% bot-loss rate.** `[settled]`
  Per campaign: `contributor_payout = (1−γ)D`, `burn = (γ−0.02)D +
  mint_actual`, `treasury = 0.02D`, `mint_actual = min(α×D,
  pool_share)`. γ=5% gives 5% strict bot loss on self-deal; ~3%
  net long-run circulating-supply contraction per campaign
  (independent of mint pool state).
- **Net-deflationary regime.** `[settled]` Circulating supply
  decreases monotonically over campaign lifetime (treasury washes
  to long-run zero via eventual sale-back). Holding is
  structurally attractive — strong narrative.
- **Concurrency: irrelevant under the formula.** `[settled]`
  Contributor payout, bot loss, and net supply change are all
  invariant to mint pool state. Allocation rule across concurrent
  campaigns can be the simplest one (pro-rata at close, or FIFO).
- **Dry-spell mint stays in pool.** `[settled]` Accumulated mint
  during idle periods doesn't burn or escape — drains when
  campaigns return. The conservation equation handles emptiness
  automatically (burn drops with mint_actual; γ×D loss invariant).
- **Calendar mint = burn-buffer (under current formula).**
  `[settled finding]` Calendar mint adds zero to circulating
  supply under the strict cap — what enters circulation via mint
  is exactly what extra burn destroys. Mint plays an economic
  role *only* if a non-burn distribution channel is added. POL
  is the live candidate; see *A — Token shape*.
- **Structural cap on any new-mint-to-graph mechanism.**
  `[settled, derived]` Any mechanism that creates new CGT and
  routes it to graph-defined recipients hits the same self-deal
  cap: per binding period, distribution `< γD = 0.05D`, else
  self-deal becomes profitable. Maximum net circulating-supply
  growth per binding = `γD − burn = 0.02D`, less with any safety
  margin. Future "distribute to active users" proposals must
  clear this audit first.
- **Asymptotic supply requires mint decoupled from burn
  activity.** `[settled, derived]` The peer-network curve has an
  asymptote because mint is *scheduled*. Any mechanism that ties
  mint amount to burn volume gives linear-in-volume supply →
  unbounded. POL (calendar mint into LP) preserves the
  asymptote; burn-coupled mint mechanisms do not.

---

## Open sub-questions

### A — Token shape

- **Chain choice.** Need cheap settlement, DEX composability,
  no single-operator risk. Candidates: ETH L2 (Base / Optimism /
  Arbitrum), Solana, custom appchain. `[proposal]` decide once the
  primitive is written; chain choice is implementation.
- **Token issuance model: decaying calendar mint, asymptotic fixed
  supply.** `[settled, direction]` Peer-network supply curve —
  fixed daily mint with ~10%/year decay, ~18M lifetime asymptote.
  Possibly a milder variant (smaller starting daily mint + gentler
  decay) to soften the early-vs-late dropoff; not BTC-steep but
  somewhere in that family. Exact parameters TBD when authoring
  `token.md`.
  - **Rejected: large premine** — concentrates CGT in designated
    parties before any economy exists. Wrong distribution.
  - **Rejected: burn-and-remint (campaign-driven mint with
    burned advertiser deposit)** — economically equivalent to
    just paying contributors from the deposit, with extra steps,
    *unless* mint > burn, in which case it's inflation-as-subsidy
    and collapses into the calendar-mint design anyway.
- **Initial allocation: proportional carry-forward from existing
  peer-token holdings.** `[settled]` No fresh premine to designated
  parties. Existing peer-token holders (company, founders, alpha
  users) keep their *percentage* of the prior token state,
  translated into CGT — not unit-for-unit. This seeds initial LP
  liquidity and respects pre-existing holder expectations without
  creating new concentration.
- **Marketing-flow conservation equation.** `[settled]` Per campaign:

  ```
  deposit + mint_actual = burn + treasury + contributor_payout
  ```

  With strict-cap design (`contributor_payout < deposit` always) and
  size-invariant `γ = 5%` percentage loss:

  ```
  contributor_payout = (1 − γ) × deposit             = 0.95 D
  treasury           = 0.02 × deposit                = 0.02 D
  mint_actual        = min(α × deposit, pool_share)
  burn               = (γ − 0.02) × deposit + mint_actual
                     = 0.03 D + mint_actual
  ```

  **Invariants:** contributor_payout, bot self-deal loss, and net
  circulating-supply change are all independent of mint_actual.
  Per-campaign net circulating-supply change = `−(γ − 0.02) × D
  = −0.03 D` long-run (after treasury sale-back). Strictly
  negative — circulating supply only decreases over time.

- **Worked examples** (D=10k, γ=5%, α=20%):

  | Scenario | mint_actual | burn | treasury | payout | bot loss | Δ supply |
  |---|---|---|---|---|---|---|
  | Pool full | 2,000 | 2,300 | 200 | 9,500 | 500 | −300 |
  | Pool empty | 0 | 300 | 200 | 9,500 | 500 | −300 |
  | Pool partial (500) | 500 | 800 | 200 | 9,500 | 500 | −300 |

  Identical outcomes for all participants and for system supply
  regardless of pool state.

- **Gaming-attack audit** (all safe under strict cap):

  | Attempt | Outcome |
  |---|---|
  | Self-deal (advertiser=contributor) | Loses γ×D always |
  | Tiny-h-gain self-deal | Same — payout is (1−γ)D regardless of h-gain |
  | Pool-depletion DoS | Cost-prohibitive (γ × Σdeposits); no leverage on legitimate users |
  | Sybil contributors | Shapley measures structural contribution from graph |
  | Timing for full/empty pool | Bot loss invariant — no exploit |
  | Coalition advertiser+contributors | Loses γ×D split among colluders |
  | Off-chain side payments to fake attribution | Attribution is graph-computed on-chain |
  | Cross-campaign coordination | Each campaign loses γ×D independently |

- **Calendar mint as burn-buffer (current finding).** `[settled]`
  Mint_actual flows into circulation via contributors but is
  exactly offset by additional burn — net circulating change from
  mint = 0. The pool accumulates calendar mint; campaigns drain
  pool into burn. Calendar mint as currently wired does *zero
  economic work*: it's bookkeeping. The system's supply behavior
  is identical with or without calendar mint.

- **Open: non-burn distribution channel for mint.** Without one,
  calendar mint serves no purpose. (a) "cosmetic narrative"
  rejected (useless mechanisms get deleted). (b) "scrap calendar
  mint" rejected (would lock ~1500 peer-network holders as
  forever-owners of all CGT). (c) the path: find calendar mint a
  real job.

  **Eliminated candidates:**
  - **~~(i) Host / infrastructure with proof-of-resource.~~** Scrap.
    Big engineering overhead and off-ethos — distribution should
    flow to *relevant users*, not infrastructure providers, even
    if infra-providers can be proof-of-resource-verified.
  - **~~(W) Cap-relaxation for arms-length contributors.~~** Scrap.
    No un-gameable definition of "arms-length" exists — bots can
    span any graph distance with sybils, controlling both h(t)
    and hop count R between any two points in their fabricated
    sub-graph.
  - **~~(X) Target-supply commitment.~~** Not actionable. Best
    case the current mint shape is preserved — the open question
    is purely *where the mint goes*, not how much.
  - **~~(Y) Proof-of-personhood gated direct distribution.~~** Scrap.
    Off-ethos. Cogra's distinction is that bot/human resolution
    is a property of the *graph itself* (severance + topology), not
    of external differentiators (KYC, biometrics, IP scans, mouse
    tracking — all outdated and breakable).
  - **~~(Z) Importance-weighted distribution from burn activity.~~**
    Scrap. h(t) zero-jail handles free-riders structurally
    (severed/unconnected accounts contribute 0 to importance mass),
    but in-view self-deal still binds: bot funds campaign and is
    sole occupant of its own h-view neighborhood, capturing
    distribution back. Strict-cap reasoning extends to (Z) — per
    campaign distribution `< γD = 0.05D` or self-deal becomes
    profitable. At the cap, net circulating-supply growth =
    `γD − burn = 0.02D` per campaign, less with safety margin,
    and the growth lives in the campaign neighborhood — liquid
    market supply still contracts unless treasury continuously
    sells. Two shape problems on top of the small size: growth
    scales linearly with campaign volume → supply → ∞ (breaks
    the asymptotic curve), and supply direction depends on
    treasury policy rather than being structural. POL fills the
    same role without these issues.

  **Surviving candidate.**

  - **POL (Protocol-Owned Liquidity).** Calendar mint
    periodically deposits into the protocol's DEX LP position.
    Net: LP CGT depth grows, total supply grows, existing holders
    diluted proportionally, late users buy from deep protocol-LP
    at AMM market price rather than from coordinated early
    holders. Defensible narrative.

    Pre-check: POL appears self-deal-immune by construction —
    bot funds campaign, burns CGT, receives nothing from POL;
    would have to buy LP exposure at market price like any other
    actor. No graph-visible extraction surface.

    Open sub-questions for the POL focused session:
    - Trigger / cadence for pool → LP (per-epoch, threshold-
      triggered, continuous?).
    - One-sided deposit (AMM auto-balances; simpler; larger price
      impact) vs. swap-half-then-pair (capital-efficient; smaller
      price drop). User confirmed 50/50-by-value is the standard
      LP shape.
    - What's done with LP fees and LP shares:
      - (α) protocol holds forever — slow, no direct user benefit.
      - (β) fees fund treasury — composable with anything.
      - (γ) periodic release of LP shares to users — recursively
        anti-sybil-bound.
      - (δ) fees buyback-and-burn — further deflationary.
    - Chain / DEX choice (couples to "Chain choice" above).
    - Initial seeding: does peer-token carry-forward also seed
      initial LP, separately from POL's ongoing growth?
    - Impermanent-loss posture: protocol holds long-term, accept
      IL, or hedge?
    - Mint schedule vs. burn rate. Strict-cap burn is 0.03D per
      campaign; POL mint must offset this to set net supply
      direction. What ratio yields the shape we want?

  Treasury-only direction (mint accrues to treasury for
  discretionary use) rejected as poor distribution narrative.

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
