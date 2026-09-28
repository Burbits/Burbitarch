# Security

The threat model, the defenses, and the rules the codebase must never break. Read with `05-MARKETS-AND-SETTLEMENT.md` (market-manipulation rules) and `12-TESTING.md` (how these claims are checked).

---

## 1. Assets at risk and trust boundaries

| Asset | Where | Who can move it |
| --- | --- | --- |
| User USDC | Per-market vaults | The program alone, and only per the instruction table in `06-ONCHAIN-PROGRAM-SPEC.md`: to the seat owner (withdraw, sweep), to another seat on a fill, to pair collateral, or to fees |
| Positions | Seat ledger | Owner (trade, transfer, merge, redeem); matching within escrowed bounds |
| Fees | `fees_accrued` | Keeper fees per fixed schedule; treasury sweep |
| Rent | Market + blocks | Recorded payers at close |

Trust boundaries: **users trust the program** (audited, tested, invariant-checked) and **Solana itself**. They do not trust keepers (liveness only), the indexer (convenience only), the fee-payer service (spends only its own SOL), the app, or Burbit the company. The single external data dependency is the Pyth SOL/USD feed, used only to floor the open-interest cap, consumed with staleness and confidence checks, and incapable of moving funds: a corrupted price could at worst set a cap wrong, bounded further by the in-program cap ceiling per market.

## 2. Program-level defenses

1. **Checked math everywhere.** All balance arithmetic in u128 intermediates with explicit floors; any overflow aborts the instruction. No signed arithmetic in fund paths.
2. **Escrow-first design.** Nothing can be owed: every order locks its worst case (cost + max fee, or shares) before it can rest; matching only moves locked amounts; releases are exact.
3. **Account validation on every entry point.** PDA re-derivation for market, vault, seat context; owner checks on every external account; the launchpad state account validated by owner, address and reader parse on every use; reject-by-default readers.
4. **No lamport or token path bypasses.** The vault authority PDA signs only inside `withdraw`, `sweep_positions`, `sweep_fees`, `deposit` (inbound), `split`/`merge`, `redeem` and bond flows; grep-level enumerability of every `invoke_signed` is a review requirement.
5. **Re-entrancy and CPI surface.** The program CPIs only to the SPL token program and to itself (event log). No arbitrary CPI, no callback surface, no delegate patterns.
6. **State machine enforcement.** Every instruction checks `state` first; transitions are one-way; `outcome` writes exactly once.
7. **Freeze-on-inconsistency.** The full vault reconciliation in `sweep_fees`/`close_market` failing halts only that market (funds stay; manual review path via void), never the protocol.

## 3. The admin plane (the part that kills protocols)

History's lesson is that exchanges die from admin keys, not order books. Burbit's admin surface is minimized and hard-limited:

- **Admin can**: pause new markets and new orders (exits always work), stage parameter changes and reader registrations behind a **24-hour timelock**, and flag an unresolved malformed market for void (also timelocked).
- **Admin cannot**: touch any vault, seat, book, outcome, or fee bucket; upgrade parameters outside in-program bounds; or shorten the timelock (the timelock parameter itself is timelocked).
- **Program upgrade authority** is the largest residual power. Policy: multisig with hardware-isolated signers, a public upgrade timelock via a governance/deploy buffer, no pre-signed transactions of any kind, durable-nonce transactions forbidden for authority keys, and monitoring that alerts on any authority-related instruction. Path to renouncing or freezing upgrades once the program matures is a roadmap commitment, not a vague intention.
- **Treasury key** can only receive.

## 4. Market-integrity threats (summary; full analysis in `05`)

| Threat | Defense |
| --- | --- |
| Force the outcome to win a market | Market size capped at α × the amount the token still needs, read live and priced with a floored SOL/USD |
| Trade on a decided outcome | Halt-on-completion inside every trading instruction |
| Deadline games | Close gap; irreversible events only |
| Pick off stale makers | Progress guards checked atomically at match; per-order expiry slots |
| Creator insider trading | Creator barred from open markets; rug markets creator-written only; open dump markets capped to pocket change |
| Launchpad layout change | Fail-closed void at $0.50/share |
| Oracle manipulation of the cap | Confidence-floored, staleness-checked price; per-market cap ceiling |
| Bundled insiders stalling graduations | Short windows, caps, disclosure; accepted residual risk |

## 5. Session keys and the sponsor service

- A session key is scope-limited in-program (place/cancel only), time-limited, revocable, and per-wallet. Compromise of a session key lets an attacker place and cancel orders with the victim's already-deposited balances; it cannot withdraw, transfer shares, or extend itself. Mitigations: default 24 h expiry, origin-scoped storage, revoke-on-logout, and a per-key notional velocity limit enforced by the sponsor service (an attacker with the key but not sponsorship must pay their own fees, which also fingerprints them).
- The sponsor endpoint validates a submitted transaction byte-for-byte: exactly one Burbit instruction of the allowed set, signed by a registered session key, fee payer position occupied by the service. It can lose at most its own SOL float and rate-limits per key and per IP.

## 6. Keeper-related risks

Keepers can only call permissionless instructions whose effects are fully determined by on-chain state; a malicious keeper can at worst do valid things early (create a legitimate market, halt at the correct moment) or waste its own fees. Ordering games are bounded: uncross results are independent of who calls; halts race toward accuracy by design; sweeps are push-payments to rightful owners only. The one keeper-latency-sensitive value, `completion_slot`, degrades toward the close time bound if all keepers vanish, and market families sensitive to per-second timing are not listed, by policy, for exactly this reason.

## 7. Denial-of-service considerations

- **Book stuffing**: every resting order costs escrow (min $1 notional) plus a rent-funded block; spam is self-funding for the market and reclaimable. Matching skips-and-removes voided orders at bounded cost; `prune_expired` cleans quiescent books for a fee.
- **Seat stuffing**: a seat requires a deposit or an order; empty seats can be swept and freed.
- **Hot-account contention**: one writable market account bounds each market's throughput at Solana's per-account lock rate, far above realistic flow for minutes-long markets; the protocol-wide surface is horizontally sharded by market.
- **Halt gas-griefing**: halts are resumable (`halt_continue`) so no book depth can make halting impossible.

## 8. Privacy

Everything is public by design: positions, orders, PnL. The app says so. The only privacy gate is the WebSocket `account` channel's ownership proof, which protects convenience aggregation, not the underlying public data.

## 9. Audit and disclosure plan

- Two independent audits of the program before mainnet funds: one full-scope (accounts, matching, settlement, resolution), one focused on the block-pool/tree implementation and the reader parsers.
- The differential test suite and invariant checks (`12-TESTING.md`) run in CI on every commit; fuzzing of readers against corpus-mutated launchpad account bytes.
- A public bug-bounty with a published safe-harbor policy from devnet onward; contact and PGP key in the repository; severity tiers topping at a bounty meaningful relative to TVL.
- Incident policy: pause, void-and-refund affected markets (the designed unwind), publish a post-mortem. Because vaults are per-market and fully collateralized, the blast radius of any defect is enumerable in advance.
