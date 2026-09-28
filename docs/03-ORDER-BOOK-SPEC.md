# Order Book Specification

The complete behavior of a Burbit market's book: how orders are expressed, how they rest, how they match, how each side settles, what makers and takers are, and how fees are charged. This is normative: the program and the reference implementation must both behave exactly as written here.

---

## 1. One book, four intents

Each market has **one order book quoted in YES prices**. A user can want four things, and each maps onto that single book:

| Intent | Placed on the book as | Escrow taken at placement |
| --- | --- | --- |
| **Buy YES** at price p | Bid at p | `p × size / 1000` USDC + maximum fee |
| **Sell YES** at price p | Ask at p | `size` YES shares |
| **Buy NO** at price q | Ask at (1000 − q) | `q × size / 1000` USDC + maximum fee |
| **Sell NO** at price q | Bid at (1000 − q) | `size` NO shares |

Prices are in mills (see `02-CORE-CONCEPTS.md` section 4). Buying NO at $0.75 enters the book as an ask at $0.25 in YES terms; the order records its true intent so settlement knows what the parties actually hold and owe. Because NO orders are folded into their YES equivalents, all liquidity concentrates in one place and there is exactly one market price.

## 2. The four settlement kinds

When a bid and an ask cross, what happens depends on the two intents behind them. This is the heart of the system:

| Bid intent | Ask intent | Kind | What settles | Pairs |
| --- | --- | --- | --- | --- |
| Buy YES | Sell YES | **TRANSFER_YES** | Buyer's USDC to seller; seller's YES shares to buyer | unchanged |
| Buy YES | Buy NO | **MINT** | Both buyers' USDC (p + (1000 − p) = $1.00 per share) locks as pair collateral; a new pair is created; YES to one buyer, NO to the other | +size |
| Sell NO | Sell YES | **MERGE** | The two sellers' shares form pairs, which are burned; each pair's $1.00 splits p to the YES seller and (1000 − p) to the NO seller | −size |
| Sell NO | Buy NO | **TRANSFER_NO** | Buyer's USDC to seller; seller's NO shares to buyer | unchanged |

Settlement math per fill (p = YES price in mills, q = size in micro-shares, all division floored in u128):

| Kind | YES-side seat | NO-side seat | Vault pair bucket |
| --- | --- | --- | --- |
| TRANSFER_YES | buyer: `usdc_locked −= p·q/1000`, `yes_free += q` | seller: `yes_locked −= q`, `usdc_free += p·q/1000` | n/a |
| MINT | buyer: `usdc_locked −= p·q/1000`, `yes_free += q` | buyer: `usdc_locked −= (1000−p)·q/1000`, `no_free += q` | `pairs += q` |
| MERGE | seller: `yes_locked −= q`, `usdc_free += p·q/1000` | seller: `no_locked −= q`, `usdc_free += (1000−p)·q/1000` | `pairs −= q` |
| TRANSFER_NO | seller: `no_locked −= q`, `usdc_free += (1000−p)·q/1000` | buyer: `usdc_locked −= (1000−p)·q/1000`, `no_free += q` | n/a |

Fees are applied on top of these moves (section 6). Note the deep property: **MINT is why no market maker is needed to start a market** (two opposite believers create shares between themselves), and **MERGE is why anyone can always cash out** (two opposite exiters dissolve shares back into dollars). Every share in existence has a real opposing holder; the platform never takes a side.

## 3. Maker, taker, and priority

- **Maker**: an order resting on the book at the moment a trade occurs. It added liquidity.
- **Taker**: the incoming order that crosses the book and executes against resting orders. A marketable limit order is the taker for the quantity it fills on arrival; any remainder that rests becomes a maker for later fills.
- **Price-time priority**: the best-priced resting order fills first; among equal prices, the earliest fills first. Enforced by the order key: `key = (price_mills << 64) | seq`, with `seq` bit-inverted for bids so that numeric ordering of keys equals price-time ordering on both trees.
- **Trades execute at the maker's price.** If a taker's limit is better than the maker's, the maker's price applies and the taker keeps the improvement; the taker's unused escrow is released immediately.
- **Self-trade prevention**: an incoming order skips any resting order owned by the same seat; the resting order stays untouched.

## 4. Order types and parameters

| Field | Values | Behavior |
| --- | --- | --- |
| `intent` | BUY_YES, SELL_YES, BUY_NO, SELL_NO | Section 1 |
| `price` | mills, on-tick | Limit price in YES or NO terms per the intent |
| `size` | micro-shares | Total quantity |
| `order_type` | LIMIT | Fills what it can; the remainder rests until filled, cancelled, expired, voided by its guard, or the market halts |
| | IOC | Fills what it can immediately; the remainder's escrow is released. The app's Quick Bet is an IOC at the best price with a slippage bound |
| | FOK | Fills entirely immediately or not at all |
| | POST_ONLY | Rejected if any part would fill immediately; for makers who only want to rest and earn rebates |
| `expiry_slot` | 0 = none | The order is void once the chain passes this slot; a dead-man switch so unrefreshed quotes cannot be picked off. Expired orders are skipped and lazily removed during matching, and cancellable by anyone via `prune_expired` |
| `guard_min`, `guard_max` | basis points of progress; (0, 10000) = no guard | The **progress guard**: the order is valid only while the token's live progress is inside [min, max]. Checked against the launchpad account passed into every matching transaction; a guard-violating resting order is voided and its escrow released before anyone can hit it |
| `client_id` | u64 | Caller's correlation id, echoed in events |

## 5. The matching algorithm (normative)

Inside `place_order`, after state checks (market Continuous; token not graduated; caller not the barred creator) and escrow:

```
side       = BID if intent in {BUY_YES, SELL_NO} else ASK
yes_price  = price if intent in {BUY_YES, SELL_YES} else 1000 - price
remaining  = size

loop while remaining > 0:
    M = best resting order on the opposite tree, skipping with removal:
          - expired orders (expiry_slot < current slot)      -> void, release escrow
          - guard-violated orders (live progress outside)     -> void, release escrow
        and skipping without removal:
          - orders owned by the caller's seat
    if M is None: break
    if side == BID and M.yes_price > yes_price: break
    if side == ASK and M.yes_price < yes_price: break

    kind = derive from (bid intent, ask intent)              # section 2 table
    q    = min(remaining, M.remaining)
    if kind == MINT:
        q = min(q, cap_remaining_pairs())                    # cap from live curve + SOL/USD floor
        if q == 0: break
    settle(kind, M.yes_price, q)                             # section 2 math, maker seat credited in place
    charge_fee(taker, M, q, M.yes_price)                     # section 6
    emit Fill event (maker, taker, kind, price, q)
    remaining -= q
    if M.remaining == 0: remove M from tree, emit Done

if remaining > 0:
    LIMIT:      rest remainder on own tree (escrow already locked); emit Posted
    IOC:        release remainder's escrow
    FOK:        the whole instruction reverts unless remaining == 0 was reached
    POST_ONLY:  reverts if any fill occurred; else rests in full
```

Compute is bounded by the transaction's compute budget rather than a fixed fill limit; each fill is a handful of in-account mutations (no token transfers, no CPIs except the event log), so dozens of fills fit in one transaction. If the compute budget nears exhaustion mid-match, the program posts the remainder (LIMIT) or releases it (IOC) and succeeds, never reverts for depth.

During the **Auction** state the same escrow and validation rules apply but nothing matches; see `04-MARKET-LIFECYCLE.md` for the uncross algorithm.

## 6. Fees

- **Takers pay 2% of their USDC notional** on each fill: `fee = 2 × cost / 100` where `cost` is the taker's side of the settlement math (for share-selling takers, the fee is deducted from their USDC proceeds; for share-buying takers it is taken from escrow, which was locked as `cost × 1.02` at placement). Fees round down; the minimum charged is 1 base unit on any nonzero fill.
- **Makers pay nothing.** At each fill, **20% of the taker fee is credited immediately to the maker's seat** as a rebate, and the remaining 80% accrues to `fees_accrued` for the treasury.
- **Auction fills** charge **each filled participant 1% of their own notional**, with no rebate. Rationale: in a single-price cross neither party was resting, so neither provided liquidity and neither earns a maker rebate; the single 2% trade fee is split evenly between them. Charging one side only would be arbitrary, because in a mint both parties are buyers paying cash.
- **Free of Burbit fees**: split, merge, deposit, withdraw, cancel, transfer_shares, redeem. (Network fees, fractions of a cent, may apply and are sponsored in the app's default flow.)

Worked fee examples: `09-FEES-AND-ECONOMICS.md`.

## 7. Cancellation and exits

- `cancel_order(order_id)`: owner or their session key; removes the order and releases its escrow to the seat's free balance instantly.
- `cancel_all(market)`: same, for every order the seat owns.
- `prune_expired(market, n)`: permissionless; removes up to n expired or guard-violated resting orders, releasing their escrow. Keepers run this; matching also does it lazily.
- On **halt** (graduation, close time, or void), every resting order's escrow is released in-place; nothing remains at risk on the book. Because orders are escrow-backed ledger entries, release is a pure in-account operation and cannot fail.
- A holder can always exit a position before settlement by selling on the book, or by holding both sides and calling `merge` (always allowed, in every state, including after halt).

## 8. Worked example, end to end

Market: "Will token X graduate within 15 minutes?", created at 70% progress. Forcing cost 8.62 SOL; with SOL floored at $150, cap = 50% × $1,293 = $646 of open interest.

**Auction (60 s).** Orders collected:

| Trader | Intent | Price | Size (shares) | In YES terms |
| --- | --- | --- | --- | --- |
| Alice | Buy YES | $0.07 | 150 | Bid 70 |
| Carol | Buy YES | $0.05 | 100 | Bid 50 |
| Bob | Buy NO | $0.94 | 50 | Ask 60 |
| Dan | Buy NO | $0.96 | 100 | Ask 40 |

**Uncross.** Volume matchable at 40 and 50 mills is 100; at 60 and 70 mills it is 150 (both sides balanced at 150). The tie between 60 and 70 goes to the lower price: **clearing price $0.06**.

- Alice's 150 fill against Dan (100, cheapest ask first) then Bob (50). Both fills are **MINTs** at $0.06.
- Alice pays 150 × 0.06 = **$9.00**, plus her auction fee 1% = **$0.09**. Dan pays **$94.00** + **$0.94**; Bob pays **$47.00** + **$0.47**. Total auction fees $1.50, which is 1% of the $150 minted.
- Dan pays 100 × 0.94 = **$94.00**. Bob pays 50 × 0.94 = **$47.00**.
- 150 pairs now exist; the vault holds **$150.00** of pair collateral (under the $646 cap) plus $0.18 fees.
- Carol's bid at $0.05 did not cross and rests.

**Continuous.** The curve climbs to 85%. Alice posts an ask: sell 100 YES at $0.20 (maker). Eve takes it (buy YES 100 at $0.20): a **TRANSFER_YES**.

- Eve pays $20.00 + taker fee 2% × 20.00 = $0.40, total **$20.40**.
- Alice receives $20.00 plus the maker rebate 20% × 0.40 = **$0.08**.
- Treasury accrues $0.32. Total accrued fees: auction $1.50 + continuous $0.40 − rebate $0.08 = **$1.82**.

**Graduation** at minute 9: the next instruction touching the market halts it; Carol's resting escrow is released in full.

**Resolve and redeem.** YES wins. The vault holds $150.00 of collateral for 150 pairs:

| Trader | Paid | Received | Net |
| --- | --- | --- | --- |
| Eve | 20.40 | 100.00 (100 YES redeemed) | **+79.60** |
| Alice | 9.09 | 20.00 + 0.08 rebate + 50.00 (50 YES redeemed) | **+60.99** |
| Dan | 94.94 | 0 | **−94.94** |
| Bob | 47.47 | 0 | **−47.47** |
| Carol | 0 | 0 (escrow released) | 0 |
| Burbit treasury | n/a | fees 1.90 − 0.08 rebate | **+1.82** |

Zero-sum check: 79.60 + 60.99 + 1.82 = 142.41 = 94.94 + 47.47. The vault ends holding exactly the accrued fees, which `sweep_fees` collects, after which `close_market` returns all rent. Burbit never took a side; it earned only fees.

## 9. What happens "when each side is on", stated plainly

- A **bid resting** means: USDC (cost plus max fee) is locked in that seat, claimable by any ask that crosses it, refundable the instant it is cancelled, expired, guard-voided, or the market halts.
- An **ask resting** means: shares are locked in that seat under the same rules. An "ask" from a NO buyer locks USDC instead, because its settlement (MINT) consumes dollars, not shares; the escrow table in section 1 is definitive.
- A **cross** means: the settlement-kind table executes atomically, the maker seat is credited in the same instruction, the taker fee is split 20/80 between maker and treasury, and events are emitted for the indexer. There is no pending state, no deferred settlement, and no moment at which a fill exists but its money has not moved.
