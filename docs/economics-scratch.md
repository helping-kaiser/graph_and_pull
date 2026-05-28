# Economics & token — working notes

WIP scratchpad for [Q20](open-questions.md#q20--economics-primitive-distribution-ledger-home-vocabulary-anchor).
**Nothing here is canonical** — the canonical landing pads will be
new files under `docs/primitive/` and `docs/implementation/` once
decisions stabilize. This doc gets deleted when those files exist.

Decisions marked `[settled]` are explicit user choices. `[proposal]`
items are sketches awaiting the user's call.

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
- **Mint schedule.**
  - The user flagged the tension: "joined early and got rich" sells
    early adopters; sustained-activity-only rewards weaken that.
  - `[proposal]` Rewards minted on goal-hit from a sustained-issuance
    pool, rate tied to advertiser spend not calendar time. Means
    early graph-important users accrue CGT during the early period
    and may hold — captures the early-adopter narrative without
    rewarding pure squatters.
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
