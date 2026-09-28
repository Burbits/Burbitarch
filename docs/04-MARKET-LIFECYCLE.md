# Market Lifecycle

Every state a market can be in, every transition, who triggers it, and what is allowed in each. The lifecycle is designed so that no market can ever be stuck: every transition past Auction is permissionless and fee-incentivized, and every terminal path returns all funds and all rent.

---

## 1. State machine

```
                    create_market (keeper, at a listing milestone)
                                   |
                                   v
                               AUCTION  ── token graduates before uncross ──┐
                                   |                                        |
                        uncross (keeper, after                              |
                        the auction window)                                 |
                                   |                                        v
                                   v                                     HALTED
                             CONTINUOUS ── halt: graduation seen, ──────►  |
                                   |        or close time reached          |
                                   |                                       |
                                   └── reader fails to parse ──► VOID path │
                                                                           v
                                                        resolve (keeper: reads curve
                                                        or creator balance, or void)
                                                                           |
                                                                           v
                                                                       RESOLVED
                                                                           |
                                                     redeem (holders) / sweep (keeper)
                                                                           |
                                                                           v
                                                                        CLOSED
```

| State | Trading | Split/Merge | Cancels | Deposits/Withdrawals of free balance |
| --- | --- | --- | --- | --- |
| **Auction** | Orders rest, nothing matches | Yes | Yes | Yes |
| **Continuous** | Full matching | Yes | Yes | Yes |
| **Halted** | None; all resting escrow already released | Merge only | n/a (book empty) | Yes |
| **Resolved** | None | Merge only | n/a | Yes; plus redeem |
| **Closed** | Account gone | n/a | n/a | n/a (everything already swept) |

## 2. Creation

`create_market` is permissionless and normally called by the market-creator keeper the moment a token hits a listing milestone (default: 70% progress for graduation questions; see `05-MARKETS-AND-SETTLEMENT.md` for per-family milestones).

The program, in one instruction:

1. Verifies the passed launchpad state account is owned by a registered launchpad program, matches the expected address for the mint, parses under the registered reader, is not complete, and has reached the milestone.
2. Reads the amount the token still needs from its launchpad account, and (for SOL-denominated tokens) the SOL/USD price with a conservative floor, then sets the market size limit: `cap = α × amount_still_needed × price_floor`.
3. Derives and allocates the Market account (header plus initial block pool) and the market vault, recording the keeper as rent payer.
4. Records the token creator's address from the launchpad state account (barred from trading this token's open markets).
5. Sets times: `open = now`, `uncross_at = open + auction_length` (default 60 s), `close_at = deadline − close_gap` (default 300 s), and the question deadline.
6. Sets state = **Auction** and emits the MarketCreated event.

## 3. The opening auction

A brand-new market has an empty book. Opening straight into continuous trading would hand the first quote to whoever reads the curve fastest and make every early poster food for bots. So every market opens with a **single-price call auction**:

- For the auction window (default 60 s), `place_order` accepts and escrows orders but matches nothing. Cancels are allowed. Split and merge are allowed.
- Anyone may call `uncross` once `now ≥ uncross_at`. The instruction:
  1. Re-parses the launchpad state account. If the token has already graduated, the market halts instead (section 5) and every escrow is released.
  2. Voids any order whose progress guard excludes live progress or whose expiry has passed, releasing escrow.
  3. Computes the **clearing price**: the price that matches the most volume; ties broken by the smaller buy/sell imbalance, then by the lower price.
  4. Fills every bid at or above the clearing price and every ask at or below it, **all at the clearing price**, best price first, then earliest. Fills settle exactly as in continuous trading (mint, transfer or merge, per the intent table in `03-ORDER-BOOK-SPEC.md`), the fee (2%, no rebate) is charged to the bid side, and pairs minted respect the cap: if the cap binds, the latest-priority crossing orders are the ones left unfilled.
  5. Rests all unfilled remainders on the book and sets state = **Continuous**. If nothing crosses, the market simply opens with whatever rests.
- The keeper earns the uncross fee from market fee accrual.

Why it matters: arriving first in the auction earns nothing, so there is no bot race at the open; the first shares are minted between YES and NO buyers at one fair price with no market maker needed; and it concentrates the launch-moment attention, which for curve tokens is when the audience exists.

## 4. Continuous trading

Normal operation per `03-ORDER-BOOK-SPEC.md`. Every `place_order` transaction carries the token's live launchpad state account, so the market's view of the token is never staler than the current transaction:

- If the parse shows `complete == true`, the instruction halts the market on the spot instead of trading, and records the completion slot.
- Guard-violated and expired resting orders are voided as matching encounters them, and `prune_expired` cleans the rest.
- Mints re-check the cap against the launchpad's live state every time.

## 5. Halt

Trading freezes on the first of:

1. **Graduation observed**: any instruction that parses the curve and sees `complete == true`, or an explicit `halt` call by a keeper. The observing slot is recorded as `completion_slot` (used by timing-bucket markets). YES-side holders are motivated to see this recorded fast, and the keeper fee pays for the race, so the recorded slot sits within seconds of the true completion.
2. **Close time**: `now ≥ close_at` (five minutes before the deadline), via `halt`. The gap kills last-second games around the deadline.
3. **Reader failure**: the launchpad state account no longer parses (layout changed, account closed unexpectedly mid-life in a way the reader does not recognize). The market halts and is flagged for **void** resolution.

On halt, every resting order's escrow is released to seat free balances in-place (a bounded walk of both trees, resumable across transactions via `halt_continue` if a book is unusually deep). After halt: no trading, no splits; merges remain allowed so paired holders can exit to cash immediately without waiting for resolution.

## 6. Resolution

`resolve` is permissionless once resolvable and pays a keeper fee:

| Question family | Resolves | When |
| --- | --- | --- |
| Graduation | YES if `completion_slot` was recorded before the deadline; NO once the deadline passes without completion (the instruction re-parses the curve one final time to catch an unobserved completion) | Seconds after the fact |
| Timing buckets | The bucket containing `completion_slot`; "never" if none by deadline | Same |
| Race | The token in the group whose completion slot is earliest; "none" if no completion by deadline | Same |
| Creator dump (open) | YES if the creator's token balance fell below the threshold before the deadline (observed via `record_creator_sale`, or read at resolve) | At deadline or on observation |
| Creator-written rug | YES the instant the creator withdraws custody before expiry (same instruction sets the outcome); NO at expiry otherwise | Instant |
| Ecosystem-wide | Bonded proposal: proposer posts a bond with the answer; unchallenged after the challenge window it stands; a successful challenge takes the bond and re-opens proposals | Challenge window (default 2 h) |
| Any family, reader failure | **VOID** | Immediately on halt-for-failure |

Outcomes are one of: YES, NO, VOID (each share pays $0.50), or a bucket index for multi-outcome markets. Once set, the outcome is permanent.

## 7. Redemption and sweeping

- `redeem`: any holder, any time after resolution, no deadline. Pays `$1.00 × winning shares` (or `$0.50 × all shares` on VOID) from the vault into the seat's free balance, then the seat owner withdraws, or in the same transaction if they ask.
- `sweep_positions(n)`: permissionless push-payment crank. For up to n seats: redeems any winning shares, then transfers the seat's entire free balance to the owner's USDC associated token account (creating it if needed, rent charged against the swept amount only if above a dust threshold; dust below the threshold is forfeited to fees rather than stranded). Marks the seat settled and frees its block. The keeper earns a per-seat sweep fee. Nobody is required to come back to collect winnings.
- `sweep_fees`: the treasury key collects `fees_accrued`.

## 8. Closing

When every seat is settled and freed (`unsettled_seats == 0`) and fees are swept, anyone may call `close_market`: the market account and vault close, and all rent, including every incrementally-allocated block, is refunded to the recorded payers (the creation keeper for the base account, each block's payer for blocks). A market's full lifecycle is rent-neutral for everyone involved.

## 9. Void, as a first-class path

Void is not an error state; it is a designed unwind that can trigger at any point pre-resolution:

- Reader failure at any moment (the dominant cause: a launchpad changing its account layout).
- Admin `flag_void` behind the timelock, for a market discovered to be malformed (wrong deadline, wrong account), only while unresolved.

On void: halt semantics (all escrow released), outcome = VOID, every share redeems $0.50, sweep and close as normal. Each pair's $1.00 goes back out in full; the only sums not returned are fees already charged on executed trades.

## 10. Timeline of a typical graduation market

| t | Event |
| --- | --- |
| 0:00 | Token hits 70% progress; keeper creates the market; Auction opens; size limit set from live state |
| 0:00 to 1:00 | Orders collect; cancels allowed; nothing matches |
| 1:00 | Keeper uncrosses at the single clearing price; Continuous begins |
| 1:00 to ~9:00 | Trading tracks the token; guards void stale quotes as progress moves; the size limit tightens as forcing gets cheaper |
| ~9:00 | Curve completes on the launchpad; the next Burbit instruction halts the market and records the slot |
| ~9:01 | Keeper resolves YES |
| ~9:01 onward | Holders redeem; sweep keeper pays out every remaining seat |
| ~9:05 | Fees swept; market closed; all rent refunded |

A market that never graduates instead halts at `close_at`, resolves NO at the deadline, and follows the same payout path.
