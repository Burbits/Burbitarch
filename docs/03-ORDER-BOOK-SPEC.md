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

## 3. Maker and taker

### 3.1 The only rule that decides it

**The maker is the order that was already resting on the book when the trade happened. The taker is the order that arrived and crossed into it.** Nothing else is part of the definition.

This is worth stating flatly because on a Burbit book the usual shortcuts are all wrong:

- **"The seller is the maker"** is wrong. In a MINT both parties are buyers; in a MERGE both are sellers. Either one can be the maker.
- **"The NO side is the maker"** is wrong. NO orders fold onto both trees (a NO buyer rests on the asks, a NO seller rests on the bids, per section 1), so a NO trader is a maker or a taker exactly as often as a YES trader.
- **"The bid is the maker"** is wrong. Both trees hold resting orders, and a taker can arrive on either side.

Consequences that follow directly:

- **A taker is charged the 2% fee; a maker is charged nothing and receives 20% of the taker's fee as a rebate** (section 6). Which of two crossing traders pays therefore depends only on arrival order.
- **A partially-filled taker becomes a maker.** A LIMIT order that takes 60 of its 100 shares on arrival paid taker fees on those 60; the 40 that rest earn maker rebates when someone later crosses them. One order can be both, in that order, never the reverse.
- **Makers are credited in the same instruction as the fill.** The maker's seat lives in the same market account the taker's transaction is already writing to, so there is no pending state and no crank: the maker's shares, USDC and rebate all move before the transaction ends (`01-ARCHITECTURE.md` section 2).

### 3.2 Who is the maker in each settlement kind

Both parties to any cross can be either role. This table is exhaustive:

| Kind | Bid intent | Ask intent | If the **bid** was resting | If the **ask** was resting |
| --- | --- | --- | --- | --- |
| TRANSFER_YES | Buy YES | Sell YES | Maker = YES buyer; taker = YES seller, fee on proceeds | Maker = YES seller; taker = YES buyer, fee on cost |
| MINT | Buy YES | Buy NO | Maker = YES buyer; taker = NO buyer, fee on **the NO leg** | Maker = NO buyer; taker = YES buyer, fee on **the YES leg** |
| MERGE | Sell NO | Sell YES | Maker = NO seller; taker = YES seller, fee on **the YES leg** | Maker = YES seller; taker = NO seller, fee on **the NO leg** |
| TRANSFER_NO | Sell NO | Buy NO | Maker = NO seller; taker = NO buyer, fee on cost | Maker = NO buyer; taker = NO seller, fee on proceeds |

In every row the taker is charged on **their own side of the money**, per the settlement math in section 2: `p·q/1000` for a YES leg, `(1000−p)·q/1000` for a NO leg.

### 3.3 The direction asymmetry on MINT and MERGE

For a transfer the two legs are the same number, so it makes no difference to the fee which party arrives last. **For a MINT or a MERGE the two legs are `p` and `1000−p`, and the fee follows whoever arrived second.** At extreme prices that is a large difference on an identical trade.

150 shares minting at a YES price of $0.06. The trade, the pairs created and the $150 of collateral are identical in both directions; only the fee differs:

| Who arrived last (the taker) | Taker's own leg | Taker fee (2%) | Maker's rebate (20% of fee) |
| --- | --- | --- | --- |
| The YES buyer | 150 × $0.06 = **$9.00** | **$0.18** | $0.036 to the NO buyer |
| The NO buyer | 150 × $0.94 = **$141.00** | **$2.82** | $0.564 to the YES buyer |

This is intended, not an artifact: the fee prices the notional actually put at risk, and the NO buyer at 94¢ is risking $141 to win $9. But it has two design consequences that implementers and the app must respect:

1. **Longshot buyers should take; favourite buyers should post.** Buying the cheap side as a taker is close to free; buying the expensive side as a taker is the most expensive action on the venue. The app's Quick Bet on a high-priced outcome should surface the absolute fee in dollars, not just "2%", because 2% of a 94¢ share is 1.88¢ of a 6¢ maximum gain.
2. **A maker resting on the cheap side earns a large rebate.** Resting a YES bid at 6¢ and being minted into earns 20% of a fee charged on the *NO* leg. This is the strongest maker incentive in the system and it points liquidity at exactly the prices where curve-token markets trade.

### 3.4 Fees never come out of pair collateral

A buy order escrows `cost × 1.02` at placement (section 6). On a MINT, `cost` is what enters the vault's pair bucket and the 2% sits beside it:

```
pair bucket receives   p·q/1000 + (1000−p)·q/1000  =  exactly $1.00 per pair
taker fee              charged on top of the taker's cost, out of the taker's own escrow
maker rebate           credited to maker.usdc_free, a cash credit, not a discount
```

So `pairs × $1,000,000` is always fully funded no matter which side took, and invariant #1 in `02-CORE-CONCEPTS.md` holds without reference to fees. A share-selling taker instead has its fee deducted from proceeds, which is a debit against money leaving the pair bucket, never against money staying in it.

### 3.5 Priority

**Price-time priority.** The best-priced resting order fills first; among orders at the same price, the earliest-placed fills first.

This is enforced structurally rather than by comparison logic, through the order key (`06-ONCHAIN-PROGRAM-SPEC.md` section 1.2):

```
key = (yes_price_mills as u128) << 64 | seq        // asks
key = (yes_price_mills as u128) << 64 | !seq       // bids, seq bit-inverted
```

- **Asks** are matched from the tree **minimum**: lowest price first, and within a price the lowest `seq`, which is the earliest.
- **Bids** are matched from the tree **maximum**: highest price first, and within a price the *highest* key. Because `seq` is bit-inverted, the highest key at a given price is the lowest true `seq`, which is again the earliest.

One numeric ordering therefore yields price-time priority on both trees, and `bids_best` / `asks_best` are plain cached tree extremes. An implementation that stores bids without inverting `seq` will silently run last-in-first-out within a price level; this is the single easiest priority bug to ship and the differential suite tests it explicitly (`12-TESTING.md`).

**Trades execute at the maker's price.** If a taker's limit is better than the maker's, the maker's price applies and the taker keeps the improvement; the taker's unused escrow is released immediately in the same instruction.

**There is no `modify_order`.** Changing a resting order's price or size means `cancel_order` then `place_order`, which draws a fresh `seq` and goes to the back of its new price level. Priority cannot be retained across a change, and size cannot be reduced in place.

### 3.6 Self-trade prevention

An incoming order **skips** any resting order owned by the same seat. The resting order is not removed, not voided, and not filled; it is passed over and the match continues to the next order on the tree.

The design choice here is skip-and-continue, not cancel-newest or cancel-resting, because a maker running quotes on both sides must never have a quote destroyed by their own flow on the other side.

Its consequences are real and must be handled downstream:

- **A single seat can legally hold a crossed pair of its own orders.** A seat resting a bid at 60 that then places an ask at 55 skips itself, fills nothing (if it is alone at those prices), and rests. The book's `bids_best` now exceeds `asks_best`.
- **The indexer, the API and the UI must tolerate `best_bid ≥ best_ask`** and must not treat it as corrupt state or as an arbitrage signal. It is not crossed liquidity; it is one participant quoting both sides. Invariant #6 in `02-CORE-CONCEPTS.md` requires the best caches to equal the true tree extremes, and says nothing about the two trees being disjoint in price, deliberately.
- **Only the crossing side is affected.** Other traders can still hit either of those two orders normally, at which point the cross resolves itself.
- **A taker facing only its own orders** behaves per its order type: LIMIT rests, IOC releases its escrow, FOK reverts with `SelfTradeOnly`, and POST_ONLY **succeeds and rests**, because no part of it would actually have filled.

### 3.7 There is no maker or taker in the opening auction

During **Auction** nothing matches, so no order is ever resting-at-the-moment-of-a-trade. At `uncross` every crossing order fills at the one clearing price simultaneously, and the roles do not exist: the fee is 2% charged to the bid side with **no rebate to anyone** (section 6, `04-MARKET-LIFECYCLE.md` section 3).

Two things follow. Posting early in an auction buys no priority and earns no rebate, which is the point: there is no bot race at the open. And a trader who wants maker economics must wait for Continuous, where resting is rewarded.

## 4. Order types and parameters

### 4.1 Parameters of `place_order`

| Field | Values | Behavior |
| --- | --- | --- |
| `intent` | BUY_YES, SELL_YES, BUY_NO, SELL_NO | Section 1; determines the tree, the price folding and the escrow kind |
| `price` | mills, on-tick, in [1, 999] | Limit price in YES or NO terms per the intent. Never optional: there is no market order (section 4.5) |
| `size` | micro-shares | Total quantity; min $1.00 notional (`02-CORE-CONCEPTS.md` section 4) |
| `order_type` | LIMIT, IOC, FOK, POST_ONLY | Section 4.2 |
| `expiry_slot` | u64; 0 = no expiry | Section 4.4 |
| `guard_min`, `guard_max` | basis points of curve progress; `0, 10000` = no guard | Section 4.4 |
| `client_id` | u64 | Caller's correlation id, echoed in `OrderPosted`, `Fill` and `OrderDone` events |

Every order type escrows in full **before** matching begins, from `usdc_free` or the relevant share-free balance first, with any shortfall pulled from the owner's USDC ATA in the same instruction (`06-ONCHAIN-PROGRAM-SPEC.md` section 2). Matching can then only move amounts already locked, which is invariant #4.

### 4.2 The four order types, normatively

**LIMIT.** Fills what it can against the book on arrival; the remainder rests. The resting remainder keeps its escrow locked and lives until one of: fully filled, cancelled by the owner or their session key, expired past `expiry_slot`, voided by its curve guard, or released by market halt. With `expiry_slot = 0` this is a good-till-cancelled order whose real horizon is the market's own halt, which for a graduation market is minutes away; with `expiry_slot` set it is good-till-date. Emits `OrderPosted` for the resting part.

**IOC** (immediate-or-cancel). Fills what it can against the book on arrival; the remainder's escrow is **released to free balances in the same instruction** and nothing rests. A partial fill is a success, not a failure: an IOC that fills 3 of 100 shares commits the 3, refunds the escrow behind the other 97, emits `OrderDone { reason: ioc-remainder }` and returns Ok. An IOC that fills nothing at all is also Ok, with every base unit of escrow returned. The app's Quick Bet is this type (section 4.5).

**FOK** (fill-or-kill). Fills its **entire** size on arrival or the whole transaction reverts. No partial fill of an FOK is ever committed and no escrow is taken on a revert, because the transaction rolls back. It reverts with:

| Cause | Error |
| --- | --- |
| Insufficient crossable depth at the limit price | `FokUnfillable` |
| The only crossable orders belong to the caller's own seat | `SelfTradeOnly` |
| Depth exists but is MINT depth and the open-interest cap cannot admit the full size | `FokUnfillable` |

That last row is a deliberate choice: from the taker's position the order is simply unfillable, and `CapExceeded` is reserved for instructions where the cap is the direct and only cause, namely `split` and `split_set`. The cap never errors a LIMIT, IOC or POST_ONLY; it silently bounds the mintable quantity and the order continues per its type (section 5).

**POST_ONLY.** Rejected outright if **any** part of it would fill on arrival; otherwise it rests in full. It is never repriced, never shaved, and never partially posted: the whole instruction reverts with `PostOnlyWouldFill`. This is the type for quoters who want the rebate and must not accidentally pay taker fees, and it is the only way to guarantee that.

Two edge behaviors, both normative:

- A POST_ONLY priced **exactly at** the opposite best crosses and is therefore rejected. Two orders at the same YES price on opposite trees do match (the loop in section 5 breaks only on a *strictly* worse maker price), so touching the best is crossing.
- A POST_ONLY whose only crossable counterparties are its own seat's orders **succeeds and rests**, per section 3.6, because self-trade prevention means no part of it would have filled.

### 4.3 Which types are legal in which state

| Order type | Auction | Continuous | Halted / Resolved / ReadyToClose |
| --- | --- | --- | --- |
| LIMIT | **Yes.** Escrows and rests; matches nothing until `uncross` | Yes | No; `WrongState` |
| POST_ONLY | **Yes.** Identical to LIMIT here, since nothing can fill | Yes | No; `WrongState` |
| IOC | **No**; `WrongState` | Yes | No; `WrongState` |
| FOK | **No**; `WrongState` | Yes | No; `WrongState` |

**Rationale for rejecting IOC and FOK in Auction.** During Auction nothing matches, so an IOC would match nothing, release all of its escrow and return success without doing anything: a transaction the user paid for, a confirmation the app must explain, and a no-op indistinguishable from a bug. An FOK could only ever revert. Both are better rejected at the state check with a clear error than accepted as guaranteed no-ops. This narrows the "state Auction or Continuous" check in `06-ONCHAIN-PROGRAM-SPEC.md` section 2 per order type; the program must apply the matrix above, and the app must present the Auction panel with limit pricing only, which `10-FRONTEND-SPEC.md` section 3.2 already does.

Exits are never state-gated in the other direction: `cancel_order`, `cancel_all` and `merge` remain available wherever `04-MARKET-LIFECYCLE.md` section 1 says they are, regardless of what any order type can do.

### 4.4 Expiry and the curve guard

**`expiry_slot`.** A dead-man switch, so an unrefreshed quote cannot be picked off by a faster curve reader.

- The order is valid **through** `expiry_slot` inclusive and dead from `expiry_slot + 1`; matching skips a resting order when `expiry_slot < current_slot`, removes it, and releases its escrow (section 5).
- `expiry_slot = 0` means no expiry.
- An order placed with a **non-zero `expiry_slot` already in the past is rejected**, not silently treated as an IOC. Silent downgrade would turn a mistyped expiry into an unintended market-taking order. This needs the error code `ExpiryInPast` added to `06-ONCHAIN-PROGRAM-SPEC.md` section 3, as the expiry counterpart of `GuardExcludesNow`.
- Expired orders are removed lazily by matching and permissionlessly by `prune_expired`, which pays a keeper fee (`07-KEEPERS.md`).

**Curve guard `[guard_min, guard_max]`.** The progress range, in basis points, that the order is valid for.

- `guard_min = 0, guard_max = 10000` is the no-guard sentinel.
- **Both bounds are live bounds.** Curve progress is not monotonic: selling back into the curve lowers real reserves and so lowers progress. A `guard_min` protects a NO quote from a retreating curve exactly as a `guard_max` protects a YES quote from a filling one.
- Checked at **placement**: if live progress is already outside the range the order is rejected with `GuardExcludesNow`, rather than posted and instantly voided.
- Checked at **every match**, against the launchpad account that every matching transaction carries. A guard-violating resting order is voided and its escrow released **before** the incoming taker can reach it, which is the protection: the taker cannot fill a quote the curve has already invalidated, even in the same slot (`12-TESTING.md`, stale-quote sniper case).
- Guard voiding is not an error for the taker. The taker's match loop removes the voided order and continues to the next one.

### 4.5 There is no market order, and what Quick Bet actually sends

`place_order` has no market type and no slippage parameter. A price is always required.

The app's **Quick Bet** (`10-FRONTEND-SPEC.md` section 3.2) is an **IOC whose limit price is the slippage bound**. The translation is the app's job and is normatively this:

1. Take the user's dollar amount and walk the live book from the best price on the relevant side, accumulating depth until the amount is covered.
2. Take the worst price touched by that walk.
3. Widen it by the slippage tolerance and round **away** from the user (up for a buy, down for a sell) to a legal tick for that price band.
4. Send `place_order` with `order_type = IOC` and `price` set to that value.

The user therefore gets: fills at or better than the bound, never worse; a partial fill if the book moved against them, with the unfilled escrow returned in the same transaction; and no resting remainder to manage. The estimate the app showed before the tap is an estimate of quantity, and the bound is a guarantee on price. Those are the two halves that need to be true, and this is the only construction that delivers both against a book that can move between quote and land.

### 4.6 Deliberate non-goals

Not in v1, and not omissions:

| Absent | Why |
| --- | --- |
| Market order | An unbounded-price order on a minutes-long book with a live cap is a way to lose money to depth you did not read. IOC with a computed bound (section 4.5) gives the same one-tap feel with a price guarantee |
| Stop, stop-limit, trailing stop | Require a trigger evaluated between transactions, which means a crank and an operator deciding when it fired. Nothing in Burbit has either |
| Reduce-only | Meaningful only against leverage. Positions here are fully-paid assets that cannot be liquidated (`02-CORE-CONCEPTS.md` section 8), so there is nothing to reduce-only against |
| Iceberg, hidden, reserve | The book lives in a public account. Anything "hidden" is readable by anyone with an RPC connection, so the feature would be a lie |
| Modify / amend | Cancel plus replace is the same two state changes with honest priority consequences (section 3.5) |
| OCO, brackets, cross-market orders | Composition belongs in the client, which can send several instructions in one transaction, not in the matching engine's hot path and compute budget |

## 5. The matching algorithm (normative)

Inside `place_order`, after state checks (state legal for the order type per section 4.3; token not graduated; caller not the barred creator) and escrow:

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
- **Auction fills** charge the bid side 2% with no rebate (there is no meaningful maker in a single-price cross).
- **Free of Burbit fees**: split, merge, deposit, withdraw, cancel, transfer_shares, redeem. (Network fees, fractions of a cent, may apply and are sponsored in the app's default flow.)

Worked fee examples: `09-FEES-AND-ECONOMICS.md`.

## 7. Cancellation and exits

- `cancel_order(order_id)`: owner or their session key; removes the order and releases its escrow to the seat's free balance instantly.
- `cancel_all(market)`: same, for every order the seat owns.
- `prune_expired(market, n)`: permissionless; removes up to n expired or guard-violated resting orders, releasing their escrow. Keepers run this; matching also does it lazily.
- On **halt** (graduation, close time, or void), every resting order's escrow is released in-place; nothing remains at risk on the book. Because orders are escrow-backed ledger entries, release is a pure in-account operation and cannot fail.
- A holder can always exit a position before settlement by selling on the book, or by holding both sides and calling `merge` (always allowed, in every state, including after halt).

## 8. Worked example, end to end

Market: "Will token X graduate within 15 minutes?", created at 70% curve progress. Forcing cost 8.62 SOL; with SOL floored at $150, cap = 50% × $1,293 = $646 of open interest.

**Auction (60 s).** Orders collected:

| Trader | Intent | Price | Size (shares) | In YES terms |
| --- | --- | --- | --- | --- |
| Alice | Buy YES | $0.07 | 150 | Bid 70 |
| Carol | Buy YES | $0.05 | 100 | Bid 50 |
| Bob | Buy NO | $0.94 | 50 | Ask 60 |
| Dan | Buy NO | $0.96 | 100 | Ask 40 |

**Uncross.** Volume matchable at 40 and 50 mills is 100; at 60 and 70 mills it is 150 (both sides balanced at 150). The tie between 60 and 70 goes to the lower price: **clearing price $0.06**.

- Alice's 150 fill against Dan (100, cheapest ask first) then Bob (50). Both fills are **MINTs** at $0.06.
- Alice pays 150 × 0.06 = **$9.00**, plus the auction fee 2% × 9.00 = **$0.18**.
- Dan pays 100 × 0.94 = **$94.00**. Bob pays 50 × 0.94 = **$47.00**.
- 150 pairs now exist; the vault holds **$150.00** of pair collateral (under the $646 cap) plus $0.18 fees.
- Carol's bid at $0.05 did not cross and rests.

**Continuous.** The curve climbs to 85%. Alice posts an ask: sell 100 YES at $0.20 (maker). Eve takes it (buy YES 100 at $0.20): a **TRANSFER_YES**.

- Eve pays $20.00 + taker fee 2% × 20.00 = $0.40, total **$20.40**.
- Alice receives $20.00 plus the maker rebate 20% × 0.40 = **$0.08**.
- Treasury accrues $0.32. Total accrued fees: $0.50.

**Graduation** at minute 9: the next instruction touching the market halts it; Carol's resting escrow is released in full.

**Resolve and redeem.** YES wins. The vault holds $150.00 of collateral for 150 pairs:

| Trader | Paid | Received | Net |
| --- | --- | --- | --- |
| Eve | 20.40 | 100.00 (100 YES redeemed) | **+79.60** |
| Alice | 9.18 | 20.00 + 0.08 + 50.00 (50 YES redeemed) | **+60.90** |
| Dan | 94.00 | 0 | **−94.00** |
| Bob | 47.00 | 0 | **−47.00** |
| Carol | 0 | 0 (escrow released) | 0 |
| Burbit treasury | n/a | fees 0.58 − 0.08 rebate | **+0.50** |

Zero-sum check: 79.60 + 60.90 + 0.50 = 141.00 = 94.00 + 47.00. The vault ends holding exactly the accrued fees, which `sweep_fees` collects, after which `close_market` returns all rent. Burbit never took a side; it earned only fees.

## 9. What happens "when each side is on", stated plainly

- A **bid resting** means: USDC (cost plus max fee) is locked in that seat, claimable by any ask that crosses it, refundable the instant it is cancelled, expired, guard-voided, or the market halts.
- An **ask resting** means: shares are locked in that seat under the same rules. An "ask" from a NO buyer locks USDC instead, because its settlement (MINT) consumes dollars, not shares; the escrow table in section 1 is definitive.
- A **cross** means: the settlement-kind table executes atomically, the maker seat is credited in the same instruction, the taker fee is split 20/80 between maker and treasury, and events are emitted for the indexer. There is no pending state, no deferred settlement, and no moment at which a fill exists but its money has not moved.
