# Roadmap and Build Plan

The order of construction, what exists already, phase gates, and the planned extensions. Time estimates assume a small focused team; the sequencing is the contract, the dates are not.

---

## 1. What already exists (research repo)

| Asset | Status |
| --- | --- |
| `curvepm/burbit_book.py` | Reference order book: intents, settlement kinds, priority, fees, guards, cap, auction, halt, resolve, redeem; randomized invariant tests. To be extended to seats/blocks, USDC units, sweeping and closing per these docs |
| `rust/burbit-core` | Curve parsing, forcing cost, caps, settlement checks in no_std integer math, cross-checked against the Python model |
| `curvepm/pump.py`, `lifecycle.py`, `manip.py`, `analysis.py` | Calibrated curve simulation, manipulation experiments, base rates |
| `curvepm/collector.py` | Live base-rate collector over RPC |

## 2. Phase 1: Core protocol (the hackathon build)

Goal: a devnet demo trading real launches end to end.

1. **Reference upgrade**: extend `burbit_book.py` to the full model in these docs (USDC units, seats, blocks, sweep, close, multi-outcome stubs); port the worked examples as tests.
2. **Program skeleton**: Config, Market header + block pool + trees (the hardest single component; build it as a standalone `hypertree` crate with its own property tests before wiring), vault, seats, deposit/withdraw.
3. **Trading**: place/cancel/matching with all four kinds, fees, guards, expiries, cap; events via self-CPI log.
4. **Lifecycle**: auction + uncross, halt (+continue), resolve for graduation family, redeem, sweep, close, void path; reader 0 for the first launchpad; Pyth cap conversion.
5. **Differential harness**: LiteSVM runner, generator, invariants; CI from the first instruction onward, not after the fact.
6. **Keepers**: creator, uncross/halt/resolve, sweep/close in one binary with role flags.
7. **Indexer + API**: events consumer, launches tracker, the REST/WS subset the app needs (launches, markets, book, trades, account).
8. **App**: launch feed, market screen, quick bet, set odds, positions, claim; session keys + sponsor service.
9. **Devnet launch** with mirrored curve accounts; demo: live launches, real trades, automatic settlement seconds after graduation.

Exit gate: differential soak green; a full market lifecycle (create → auction → trade → graduate → resolve → sweep → close) reproducible on devnet in under 15 minutes with rent conserved to zero.

## 3. Phase 2: Private beta (mainnet, capped)

- Security: both audits, bug bounty live, upgrade-authority multisig + timelock policy in force.
- Mainnet with conservative Config: low per-market caps, graduation family only, listing policy 70%+ progress, invite-gated app writes (the program itself stays permissionless).
- Reference quoter live on collector-measured base rates; maker rebate flowing.
- Timing-bucket and race markets (complete sets, convert); creator-written rug markets and the creator console.
- Data: public stats endpoint, leaderboard, profiles.

Exit gate: 30 days incident-free, reconciliation zero-drift, keeper economics self-sustaining at real priority fees.

## 4. Phase 3: Scale

- Additional launchpad readers (one at a time, each with fuzz corpus and fail-closed proof) and multi-launchpad race markets.
- Open dump markets (small caps), ecosystem-wide daily markets with bonded proposals.
- Cap raises with TVL and audit maturity; fee-parameter review with published data.
- Public keeper program: docs, binaries, fee tuning so third-party keepers dominate operations.

## 5. Phase 4: The signed-order gateway (v2)

The reserved extension in `06` section 7, shipped when order flow justifies it: ed25519-signed order messages verified by sysvar introspection, relayed by anyone, entering the identical matching path. Delivers CEX-feel placement (no transaction from the user at all) while changing nothing about custody or settlement. Includes: relay service and public relay protocol, per-seat replay rings, maker streaming interface, and the same differential coverage extended to signed flow. Explicit non-goals remain non-goals: no off-chain matching authority, no operator sequencing rights, no custody changes.

## 6. Standing non-goals

- No AMM or house liquidity, ever: the auction + rebates + quoter is the liquidity story.
- No price-level or oracle-priced questions.
- No leverage, margin, or liquidations.
- No custodial balances or embedded-wallet custody.
- No admin capability that can touch user funds, in any phase.
