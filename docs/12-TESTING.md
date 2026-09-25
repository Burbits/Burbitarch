# Testing

How correctness is established and kept: an executable reference implementation, differential testing of the program against it, invariant checking after every step, and adversarial scenario suites. The standard: the program must be bit-for-bit reconcilable with the reference on every balance, after any legal sequence of operations, including hostile ones.

---

## 1. The executable specification

The Python reference implementation (`curvepm/burbit_book.py`, extended for this architecture) is the normative model of the whole exchange: seats, escrow, the four intents, the four settlement kinds, price-time priority, ticks, fees and rebates, curve guards, expiries, the cap, the opening auction and its clearing-price rule, halt, resolution per family, void, redemption, sweeping and closing. Every rule in `03-ORDER-BOOK-SPEC.md` and `04-MARKET-LIFECYCLE.md` is implemented there in plain arithmetic (integers, floored division, same u128 semantics), and every documented example in these docs is a test case in it, including the full worked market in `03` section 8, reproduced to the base unit.

The reference also implements the curve model: reader parsing, progress, forcing cost, and the cap formula, cross-checked against the Rust `burbit-core` library to the lamport.

## 2. Layered test suites

### 2.1 Unit (reference and Rust core, mirrored)

- Settlement-kind math: all four kinds at boundary prices (1, 999 mills), boundary sizes (1 micro-share, cap-sized), fee rounding, rebate rounding, minimum-fee rule.
- Tick and notional validation; guard and expiry acceptance/voiding; self-trade skipping.
- Auction clearing price: maximal volume, imbalance tie-break, low-price tie-break, cap-binding auctions, empty and one-sided auctions, guard-voiding during uncross.
- Forcing cost across the progress table in `05`; cap conversion under moving and stale prices; staleness rejection.
- Reader fuzz: the parser over mutated real account bytes: truncations, appended fields, flipped discriminators, wrong owners: everything not exactly parseable must fail closed.

### 2.2 Differential (the core of the strategy)

Randomized instruction sequences are generated and executed twice: through the reference, and through the compiled program in an in-process Solana VM (LiteSVM). After **every instruction**, all balances (every seat's six balances, vault, pairs, fees, every resting order's remaining and escrow) must match exactly, and the invariant set below must hold in both.

Generator coverage requirements per corpus: all intents and order types; auction and continuous phases; deposits, withdrawals, splits, merges, transfers; cancels and cancel-alls; keeper calls interleaved at random legal moments; curve updates driving guards, cap changes, and completion mid-anything (mid-auction, mid-match, between escrow and rest); price-feed drift and staleness; multi-outcome sets and converts; the bond lifecycle including custody-withdrawal rug settlement; sequences ending in resolve/void, full redemption, sweep and close. Minimum corpus: 1,000 sequences × 300 operations in CI per commit; 100k-sequence soak nightly.

### 2.3 Invariants (asserted after every differential step)

1. `vault = Σ usdc_free + Σ usdc_locked + pairs × $1 + fees_accrued`
2. Pre-resolution: `Σ yes = Σ no = pairs` (free + locked)
3. Cap respected at every mint/split against the live curve and price
4. No negative balance anywhere, ever
5. Every resting order's escrow reconstructs exactly from its terms
6. Tree validity: red-black properties, key ordering, best-index caches equal true extremes, no dangling seat references, free list disjoint and complete
7. Post-close: vault empty, all rent refunded to recorded payers, sum of all refunds equals sum of all rent paid
8. Zero-sum: across any completed market, Σ user nets + treasury = 0

### 2.4 Adversarial scenarios (named, deterministic)

- **Forced graduation**: attacker buys out the curve; assert their maximum extractable win < their forcing spend under all book states at the cap.
- **Stale-quote sniper**: curve jumps beyond a maker's guard in the same slot as a taker's order; assert the maker order voids, never fills.
- **Completion race**: graduation lands between an order's submission and execution; assert halt, full escrow release, no fill.
- **Cap exhaustion under fire**: concurrent mint pressure at the cap; assert clipping, never exceeding.
- **Auction sniping**: identical order sets submitted in every permutation produce identical uncross results.
- **Self-trade laundering**: a wallet on both sides cannot extract rebates or move the price against itself.
- **Creator second wallet**: assert the open dump-market cap bounds extractable value to its fixed limit.
- **Dust and rounding attacks**: 1-unit orders, 1-mill prices, maximal fills; assert conservation to zero units lost or created.
- **Halt-depth grief**: a maximally deep book halts across `halt_continue` calls with exact total release.
- **Reader upgrade simulation**: appended-field and changed-layout account bytes mid-market; assert void path and $0.50 redemptions conserve the vault.

### 2.5 End-to-end (devnet)

A harness that mirrors real launchpad curve accounts onto devnet (replayed account states) and runs the full stack: keepers, indexer, API, quoter and app against live-shaped data, asserting the user-visible numbers against chain state continuously.

## 3. Tooling and CI gates

- Property-based testing (proptest/hypothesis) drives the generators; every CI failure minimizes and commits its reproducer to a regression corpus.
- Coverage gate on the program crate (line and branch) with the fund-path modules required at 100% branch coverage.
- `cargo fuzz` targets for readers and the block-pool/tree code.
- A CU benchmark suite replaying recorded order flow, gating regressions against the budgets in `06` section 5.
- No release tags without: green differential soak, audit-issue tracker at zero criticals, and the worked examples in these docs re-verified by the reference.
