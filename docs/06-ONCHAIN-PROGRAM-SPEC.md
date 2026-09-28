# On-Chain Program Specification

The Burbit Solana program: accounts, data layouts, instructions, errors, events, compute and rent. Written to be implemented directly (Anchor with zero-copy accounts for the market body, or native with the same layouts). All integer math is unsigned, checked, computed in `u128` where products appear, floored on division.

---

## 1. Accounts

### 1.1 Config

- Seeds: `["config"]`. One global account. Size ~450 bytes.

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `admin` | Pubkey | deployer | Behind timelock for mutations |
| `pending_admin`, `pending_admin_ts` | Pubkey, i64 | n/a | Two-step, timelocked handover |
| `treasury` | Pubkey | n/a | Receives swept fees |
| `paused` | u8 | 0 | 1 = no new markets, no new orders; exits always allowed |
| `tiers[6]` | { `min_volume_30d`: u64, `taker_bps`: u16, `maker_bps`: i16 } | see below | The fee ladder. `maker_bps` > 0 charges that share of the maker's own notional; `maker_bps` < 0 pays that share of the **taker fee** as a rebate. Defaults: (0, 200, +100), ($250k, 180, +60), ($1M, 160, +25), ($5M, 145, 0), ($20M, 130, −1500), ($50M, 120, −3000). Rebate magnitude is bounded in-program at 3000 (30%) so the treasury always retains at least 70% of a taker fee |
| `auction_fee_numerator` | u16 | 5000 | Auction fills charge each participant this fraction of their own taker rate (50%) |
| `alpha_bps` | u16 | 1000 | Market size limit as a fraction of the amount the token still needs (10%) |
| `auction_length_s` | u32 | 60 | |
| `close_gap_s` | u32 | 300 | Trading stops this long before deadline |
| `min_order_usdc` | u64 | 1_000_000 | $1.00 |
| `create_fee`, `uncross_fee`, `halt_fee`, `resolve_fee`, `sweep_fee_per_seat`, `prune_fee_per_order` | u64 | tuned | Keeper fees, paid from market `fees_accrued` (creation fee deferred until fees exist) |
| `dust_threshold` | u64 | 10_000 | Sweep amounts below this ($0.01) are forfeited to fees |
| `usdc_mint` | Pubkey | the collateral mint | **Never hardcoded.** Mainnet uses real USDC; local and devnet use a self-minted 6-decimal test token (see `PRD.md` section 14). Promotion between environments is a config value, not a code change |
| `sol_usd_feed` | Pubkey | Pyth SOL/USD | |
| `max_price_staleness_slots` | u16 | 30 | |
| `readers[8]` | { `launchpad_program`: Pubkey, `reader_id`: u8, `kind`: u8 (curve or destination-pool), `enabled`: u8 } | reader 0 = first launchpad's curve; reader 1 = its destination AMM pool (for post-graduation markets) | |
| `timelock_s` | u32 | 86400 | Every admin mutation is proposed, then executable after this delay |
| `pending_update`, `pending_update_ts` | bytes, i64 | n/a | Staged parameter change |

Admin can **never**: move vault funds, mutate seats or books, set outcomes, or bypass the timelock. `pause` and `unpause` are the only immediate admin actions.

### 1.2 Market

- Seeds: `["market", source_account, question_tag, deadline_le_bytes]`. Zero-copy. Header 320 bytes + block pool.

**MarketHeader:**

| Field | Type | Notes |
| --- | --- | --- |
| `magic`, `version` | u32, u16 | Layout versioning |
| `state` | u8 | 0 Auction, 1 Continuous, 2 Halted, 3 Resolved, 4 ReadyToClose |
| `outcome` | u8 | 0 unset, 1 YES, 2 NO, 3 VOID, 10+n bucket n |
| `question_type` | u8 | 0 graduation, 1 timing, 2 race, 3 creator-dump, 4 creator-rug, 5 ecosystem |
| `question_params` | [u8; 32] | Family-specific (threshold bps, bucket bounds, group hash) |
| `mint`, `source_account` | Pubkey ×2 | The token and its launchpad curve PDA |
| `reader_id` | u8 | Which reader parses `source_account` |
| `token_creator` | Pubkey | From the launchpad state account; barred from open markets on this token |
| `open_ts`, `uncross_ts`, `close_ts`, `deadline_ts` | i64 ×4 | |
| `completion_slot` | u64 | 0 until graduation observed |
| `cap_usdc` | u64 | Last computed cap (informational; enforcement recomputes live) |
| `pairs` | u64 | Micro-share pairs outstanding |
| `fees_accrued` | u64 | Treasury bucket |
| `order_seq` | u64 | Monotonic per market |
| `bids_root`, `bids_best`, `asks_root`, `asks_best`, `seats_root` | u32 ×5 | Block indexes; `NIL = u32::MAX` |
| `free_head` | u32 | Free-list head |
| `blocks_allocated`, `blocks_free` | u32 ×2 | |
| `unsettled_seats` | u32 | Gate for close |
| `halt_cursor` | u32 | Resumable escrow-release walk |
| `rent_payer` | Pubkey | Creation keeper |
| `vault_bump`, `bump` | u8 ×2 | |

**Block pool:** `blocks_allocated` × **112-byte** blocks following the header. The account is reallocated in +112-byte steps by any instruction that needs a block and finds the free list empty; that transaction's fee payer funds the added rent and is recorded in the block for refund at close. Each block is one of:

**OrderBlock** (in the bids or asks tree; tree key = `(yes_price_mills as u128) << 64 | seq`, seq bit-inverted for bids):

| Field | Type |
| --- | --- |
| `key` | u128 |
| `parent`, `left`, `right` | u32 ×3 (red-black links; color bit packed in `flags`) |
| `owner_seat` | u32 |
| `remaining` | u64 (micro-shares) |
| `escrow_remaining` | u64 (USDC or shares, per intent) |
| `intent` | u8 |
| `order_type` | u8 |
| `flags` | u8 (color, escrowed-kind) |
| `guard_min`, `guard_max` | u16 ×2 |
| `expiry_slot` | u64 |
| `client_id` | u64 |
| `rent_payer_idx` | u32 (index into a payer table block, so block rent refunds route correctly) |

**SeatBlock** (in the seats tree; key = owner pubkey):

| Field | Type |
| --- | --- |
| `owner` | Pubkey |
| `parent`, `left`, `right` | u32 ×3 |
| `usdc_free`, `usdc_locked` | u64 ×2 |
| `yes_free`, `yes_locked`, `no_free`, `no_locked` | u64 ×4 |
| `open_orders` | u16 |
| `tier` | u8 (fee tier stamped from `TraderStats` at seat creation; fixed for this market's life) |
| `volume_traded` | u64 (this seat's filled notional; rolled into `TraderStats` at sweep) |
| `flags` | u8 (color, settled, creator-locked-NO) |

Multi-outcome markets extend the seat with a bucket-balance table in an overflow block chained from the seat.

### 1.3 Vault

- SPL token account, USDC, seeds `["vault", market]`, authority = PDA `["vault-auth", market]`. All transfers in or out are performed by the program via `invoke_signed`; the exhaustive list of instructions that touch it is in section 2.

### 1.4 SessionKeys

- Seeds: `["session", owner]`. Up to 4 entries of `{ key: Pubkey, expiry_ts: i64, scope: u8 }`. Scope 0 = place/cancel only. A session key may sign `place_order`, `cancel_order`, `cancel_all` for its owner's seat and **nothing else**; `withdraw`, `transfer_shares`, `register_session_key` require the owner wallet.

### 1.4a TraderStats (fee tier)

- Seeds: `["stats", owner]`. One per trader, global across all markets. ~300 bytes, ~0.003 SOL rent, refundable if closed.
- Fields: `buckets[30]: u64` (rolling daily traded notional), `last_bucket_day: u32`, `volume_30d: u64` (cached sum), `cached_tier: u8`, `bump`.
- **Written** only when a market is swept or closed, rolling each seat's accumulated volume into the owner's buckets and recomputing the tier. Buckets older than 30 days are zeroed lazily on write.
- **Read** only when a seat is created, to stamp `seat.tier`. It is therefore never required on the settlement path, which is what makes cross-market volume tiering possible without loading cross-market accounts into a fill.
- Absent account means tier 0. Creating it is optional and permissionless; the app creates it on a user's first deposit.

### 1.5 Bond (creator-written rug markets)

- Seeds: `["bond", mint, creator]`: `{ creator, custody_token_account, custody_kind, bond_usdc, expiry_ts, market, state }`. Custody withdrawal before expiry executes rug settlement atomically (section 2, `withdraw_custody`).

### 1.6 Proposal (ecosystem-wide markets)

- Seeds: `["proposal", market]`: `{ proposer, outcome, bond, proposed_ts, challenger, challenge_outcome, challenged_ts, round }`.

## 2. Instructions

Conventions: every instruction that can trade, halt or resolve takes the market's `source_account` and validates it (owner = registered launchpad program for `reader_id`; address = expected PDA for `mint`; parse per reader; on parse failure → the void path, never an error that leaves the market live). Signer column shows the required authority; "session" means owner or registered session key. All keeper fees are paid from `fees_accrued`, capped at what has accrued.

| Instruction | Signer | Writable accounts | Checks and effects |
| --- | --- | --- | --- |
| `initialize_config(params)` | deployer | Config | Once |
| `propose_config_update(diff)` / `apply_config_update` | admin | Config | Apply only after `timelock_s` |
| `pause` / `unpause` | admin | Config | Immediate |
| `create_market(question, deadline, params)` | anyone (keeper) | Market, Vault, Config | Curve valid, not complete, milestone met; Pyth fresh; cap computed; creator recorded; state → Auction; MarketCreated event |
| `deposit(amount)` | owner | Market, Vault, owner USDC ATA | Token transfer in; `usdc_free += amount`; claims a seat if new |
| `withdraw(amount, dest)` | owner | Market, Vault, dest token account | `usdc_free −= amount`; transfer out via vault authority; dest must be owned by owner |
| `place_order(intent, price, size, order_type, expiry_slot, guard, client_id)` | session | Market, Vault (only if shortfall deposit), owner USDC ATA (same), launchpad state account, Pyth feed | Not paused; state Auction or Continuous; caller ≠ `token_creator` (open markets); tick and min-notional checks; escrow from `usdc_free`/share free first, shortfall pulled from the owner's ATA in the same instruction (requires owner signer; session-key flow pre-deposits instead); if curve complete → halt path; match per `03-ORDER-BOOK-SPEC.md`; events |
| `cancel_order(order_id)` / `cancel_all` | session | Market | Release escrow to free; free block |
| `prune_expired(n)` | anyone | Market, launchpad state account | Void up to n expired/guard-violated resting orders; keeper fee |
| `uncross` | anyone | Market, launchpad state account, Pyth | `now ≥ uncross_ts`; auction algorithm per `04-MARKET-LIFECYCLE.md`; state → Continuous; fee |
| `split(n)` / `merge(n)` | owner | Market, Vault, owner ATA (split in / merge out optional) | Split: state ≤ Continuous, cap check, lock $1×n, credit both sides. Merge: any state, burn pair, release $1×n to free |
| `transfer_shares(side, amount, to)` | owner | Market | Move free shares between seats; creator-locked NO untransferable |
| `halt` / `halt_continue` | anyone | Market, launchpad state account | On completion (record slot) or `now ≥ close_ts`; release all resting escrow (resumable via `halt_cursor`); state → Halted; fee on completion-halt |
| `record_witness(event_kind)` | anyone | Market, witnessed account(s) | The generalized ever-witness crank for grade-B questions (`15-QUESTION-CATALOG.md`): verifies the market's witness condition against the passed account(s) at the current slot and, if satisfied pre-deadline, records the slot permanently. Kinds: creator balance below threshold (dump markets, reads the creator's token account), progress ≥ X, real SOL ≥ X, SOL market cap ≥ threshold (King-of-the-Hill proxy), pool reserve below X (post-graduation crash, reads the destination AMM pool via the pool reader), creator fee accrual ≥ X. Pays a small keeper fee on a successful first witness |
| `resolve` | anyone | Market, launchpad state account or creator token account or destination pool account | Per family table in `04-MARKET-LIFECYCLE.md` and the catalog (`15-QUESTION-CATALOG.md`): grade-A flags read live; grade-B questions resolve YES if a witness slot was recorded before the deadline (with one final live check), else NO; grade-C snapshots read the named account at the named slot boundary; sets `outcome` once; state → Resolved; fee |
| `redeem` | owner | Market, Vault | Winning shares × $1 (VOID: all × $0.50) to `usdc_free`; optionally chain `withdraw` |
| `sweep_positions(n)` | anyone | Market, Vault, up to n owner ATAs | Redeem + push each seat's free balance to its owner's USDC ATA; below `dust_threshold` → fees; mark settled, free blocks, `unsettled_seats −= 1` each; per-seat fee |
| `sweep_fees` | treasury | Market, Vault, treasury ATA | Transfer `fees_accrued` |
| `close_market` | anyone | Market, Vault, rent payer accounts | `unsettled_seats == 0`, fees swept, book empty; close accounts; refund rent per payer records |
| `flag_void` | admin (timelocked) | Market | Only pre-resolution; enters void path |
| `register_session_key(key, expiry)` / `revoke_session_key(key)` | owner | SessionKeys | |
| `create_bond(bond_usdc, expiry)` / `deposit_custody` / `write_rug_market` | creator | Bond, Market, Vault, custody accounts | Bond split into pairs; NO locked to creator seat |
| `withdraw_custody` | creator | Bond, Market, custody accounts | Before expiry: custody out **and** outcome → YES in the same instruction. After expiry: custody out, bond redeemable, outcome NO |
| `propose_outcome(outcome)` / `challenge_outcome(outcome)` / `finalize_outcome` | anyone (bonded) | Proposal, Market, Vault | Ecosystem markets per `05-MARKETS-AND-SETTLEMENT.md` section 8 |
| Multi-outcome: `create_multi_market`, `split_set`, `merge_set`, `convert(mask, amount)` | as above | | Complete-set semantics per `05` section 6 |

## 3. Errors (non-exhaustive, stable codes)

`Paused`, `WrongState`, `MarketExpired`, `BadTick`, `BelowMinOrder`, `PriceOutOfRange`, `InsufficientFree`, `CapExceeded`, `SelfTradeOnly` (FOK found only self orders), `PostOnlyWouldFill`, `FokUnfillable`, `CreatorBarred`, `GuardExcludesNow` (placement rejected when the guard already excludes live progress), `CurveAccountMismatch`, `CurveOwnerMismatch`, `CurveParseFailed` (routes to void where applicable), `StalePrice`, `NotOwner`, `SessionScope`, `SessionExpired`, `OutcomeAlreadySet`, `NothingToRedeem`, `SeatNotSettled`, `TimelockPending`, `BondActive`.

## 4. Events

Emitted via the program's own log instruction (a self-CPI carrying borsh-encoded event data as instruction data, immune to log truncation). Every event carries `{ market, seq, slot, ts }` plus:

| Event | Fields |
| --- | --- |
| `MarketCreated` | mint, curve, question_type, deadline, cap |
| `OrderPosted` | seat owner, order_id, intent, price, size, client_id |
| `Fill` | maker owner, taker owner, kind, yes_price, size, taker_fee, maker_rebate, maker_order_id, taker_client_id |
| `OrderDone` | order_id, reason (filled/cancelled/expired/guard/halt/ioc-remainder) |
| `Uncrossed` | clearing_price, matched_size |
| `PairsChanged` | pairs, cap |
| `Halted` | reason, completion_slot |
| `Resolved` | outcome |
| `Redeemed` / `Swept` | owner, amount |
| `Voided` | reason |
| `Deposited` / `Withdrawn` | owner, amount |

The indexer consumes these to maintain books, trades, candles and positions with no polling.

## 5. Compute, size and rent budgets

| Item | Budget |
| --- | --- |
| `place_order`, no fill | ≤ 25k CU |
| `place_order`, per fill | ≤ 6k CU marginal (in-account mutations + event bytes only) |
| `uncross`, per filled order | ≤ 8k CU; deep auctions resume via multiple transactions |
| Curve parse + Pyth read | ≤ 10k CU combined |
| Market creation rent | header 320 B + 64 initial blocks ≈ 7.5 KB ≈ **0.053 SOL**, keeper-paid, fully refunded at close |
| Marginal block | 112 B ≈ **0.00078 SOL**, payer-refunded at close |
| Seat cost | 1 block (+1 overflow for multi-outcome) |
| Transaction fee per user action | ~5k lamports, sponsored by the fee-payer service in the app flow |

The single-account market means one writable hot account per market; Burbit's per-market throughput ceiling is Solana's per-account write lock, far above the realistic order flow of a minutes-long market.

## 6. On-chain assertions

Cheap invariants asserted in-program every instruction: pairs and fee arithmetic never overflow or underflow; escrow released equals escrow recorded; `bids_best`/`asks_best` equal tree extremes after every mutation (checked in debug builds, spot-checked in release); vault balance is reconciled against the ledger sum in `sweep_fees` and `close_market` (full check, instruction fails loudly on mismatch, which can only mean a program bug and freezes only that market). The full invariant suite runs off-chain in the differential tests (`12-TESTING.md`).

## 7. The v2 signed-order gateway (reserved, not in v1)

One added instruction, `settle_signed_order`, will accept an ed25519-signed order message (order fields + market + salt + expiry slot, signed by the seat owner or session key), verified by requiring the native ed25519-verify instruction immediately before it in the transaction and introspecting it through the instructions sysvar with every offset and index re-derived and pinned. The order then enters the exact matching path of `place_order`. Replay protection: a per-seat ring of `{salt, max_slot}` entries pruned past expiry. This adds gasless, signatureless-feeling order placement relayed by anyone, with no change to custody, matching, settlement or any account layout above; the design is reserved now so v1 layouts leave room (seat flags and header versioning) and nothing needs migration.
