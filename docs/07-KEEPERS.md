# Keepers

Every off-chain worker that advances Burbit markets, its exact algorithm, and why running it is worth it. All keepers are permissionless: the instructions they call check state, not identity, and pay fixed fees from each market's accrued fees. Burbit runs reference instances; anyone else can run the same open-source binaries and compete for the fees.

---

## 1. Roles at a glance

| Keeper | Calls | Trigger | Reward |
| --- | --- | --- | --- |
| Market creator | `create_market` | A tracked token crosses a listing milestone | Creation fee + full rent refund at close |
| Uncrosser | `uncross` | `now ≥ uncross_ts` on an Auction-state market | Uncross fee |
| Halter | `halt`, `halt_continue` | Curve `complete == true`, or `now ≥ close_ts` | Halt fee (completion halts pay more; racing to observe completion is the mechanism that makes `completion_slot` accurate) |
| Resolver | `resolve`, `record_creator_sale` | Market Halted and resolvable | Resolve fee |
| Pruner | `prune_expired` | Resting orders expired or guard-violated | Per-order prune fee |
| Sweeper | `sweep_positions` | Market Resolved with unsettled seats | Per-seat sweep fee |
| Closer | `close_market`, `sweep_fees` (treasury only for fees) | All seats settled | Residual rent routing |
| Proposal keeper | `finalize_outcome` | Ecosystem-market challenge window elapsed | Finalize fee |

## 2. The keeper loop (shared skeleton)

Each keeper is a small daemon over the same primitives:

1. **Watch**: subscribe to program accounts (websocket account subscription on the Market discriminator) and to the launchpad program's launchpad state accounts for tracked mints; maintain an in-memory table of `{market, state, times, curve_progress, complete}`.
2. **Decide**: on every account update or clock tick, emit the set of due instructions (pure function of the table; deterministic, so multiple keepers converge on the same work).
3. **Submit**: build transactions with a compute-budget instruction and a dynamic priority fee (p75 of recent fees, bumped on failure); land via standard RPC, with an optional bundle path for latency-critical completion halts.
4. **Reconcile**: treat "account already in target state" errors as success (someone else won the race); never retry an instruction whose precondition is gone.

Races between keepers are safe by construction: every keeper instruction is idempotent at the state-machine level, and the fee goes to whoever lands first.

## 3. Per-keeper specifics

### Market creator
- Follows the indexer's launch feed (or its own launchpad subscription). On milestone crossing, submits `create_market` with the launchpad state account and Pyth feed.
- Fronts ~0.053 SOL rent per market, returned at close; sizes its float as `rent × expected concurrent markets`.
- Applies listing policy locally (family, milestone, window) but the program independently re-verifies the milestone, so a rogue creator keeper can only create valid markets early or not at all.

### Halter
- The latency-sensitive one. Watches launchpad state accounts directly; on observing `real_token_reserves == 0` or `complete == true`, races `halt`. Because *any* trading instruction also halts on observation, the halter's job is to make sure a quiet market (no trades in flight) still records completion within a slot or two.
- Runs with the highest priority-fee ceiling of all keepers; the completion-halt fee is set to cover worst-case fee spikes.

### Resolver
- After halt: graduation and timing markets resolve immediately (completion recorded) or at deadline (final re-parse, then NO). Dump markets read the creator's token account. Maintains a deadline heap so nothing resolves late.

### Sweeper
- Batches up to n seats per transaction (bounded by the writable-account limit for destination ATAs, ~8 to 10 per transaction with lookup tables). Creates missing ATAs, charging the creation rent against the swept amount per the dust rules. Sweeps largest balances first so most value is returned fastest.

### Pruner
- Low urgency: matching voids stale orders lazily anyway. The pruner keeps quiet books clean so halts stay cheap.

## 4. Operational requirements

- A funded Solana keypair (SOL for fees and, for the creator keeper, rent float).
- A reliable RPC with websocket subscriptions; a second RPC as failover. For the halter, a low-latency or dedicated RPC is strongly recommended.
- Clock discipline: all time checks derive from chain clock (slot and cluster time), never wall clock.
- Metrics: every reference keeper exposes counters (instructions attempted, landed, lost races, fees earned, lag from trigger to landing) so operators can tune priority fees.

## 5. Economics

Keeper fees are set in Config so that, at conservative priority-fee assumptions, each role clears a margin on every action (target ≥ 3× the expected transaction cost). Because fees are paid from each market's own accrued trading fees, keeper work on a market is funded exactly by that market's activity; a market with zero trades accrues nothing and needs nothing beyond its (refundable) creation rent and its free-of-charge deadline resolution, whose fee falls back to a Config-funded minimum so even dead markets always close.

## 6. Degradation analysis

| Scenario | Consequence | Floor |
| --- | --- | --- |
| No creator keepers | No new markets | Anyone can start one; the program needs no permission |
| No halter | Completion recorded by the next trading instruction instead; quiet markets record late (bounded by close time) | Timing-bucket markets sensitive to seconds are not listed (policy), so late observation costs accuracy, not funds |
| No resolver | Redemption delayed | Any holder can call `resolve` themselves; the app exposes a "resolve now" button that submits it from the user's wallet |
| No sweeper | Winners must click redeem | Funds never expire; `redeem` is permissionless forever |
| No closer | Rent stays locked | Any party owed rent is motivated to close; the instruction is open to all |

Every degradation leaves user funds safe and reachable; keepers buy timeliness, never safety.
