# Fees and Economics

Every fee in the system, who pays it, where it goes, and the revenue model. Burbit earns trading fees only: it never stakes capital, never takes a side, and never earns from user losses.

---

## 1. The fee schedule

| Action | Fee | Paid by | Distribution |
| --- | --- | --- | --- |
| Taker fill (continuous trading) | **2.00% of the taker's USDC notional** on the filled quantity | Taker | **20% to the maker** on the other side, credited to their seat at fill time; **80% to the treasury**, accrued per market |
| Auction fill | 2.00% of the bid side's notional | Bid side | 100% treasury (a single-price cross has no resting maker to reward) |
| Maker fill | **0** | n/a | Makers earn, never pay |
| Split, merge, deposit, withdraw, cancel, transfer, redeem | **0** | n/a | Network fees only, sponsored in the app flow |
| Creator rug-market premium | 10% of the creator's YES-sale proceeds, charged as those sales settle | Creator | Treasury |

Formal definition (mills price p, micro-shares q):

```
taker_notional = p·q/1000            (share buyer)   or   proceeds        (share seller)
fee            = floor(taker_notional × 200 / 10_000), minimum 1 base unit on a nonzero fill
maker_rebate   = floor(fee × 2_000 / 10_000)
treasury       = fee − maker_rebate
```

Buy-side escrow locks `cost × 1.02` at placement so the fee can never make a balance negative; unused fee escrow releases with any unfilled remainder.

## 2. Why this exact structure

- **Flat 2% on the taker** is simple to reason about, matches what earlier prediction-market economics proved viable at launch scale, and prices the aggressor, who consumes liquidity and information, rather than the quoter who provides both.
- **The 20% maker rebate** replaces the liquidity-provider compensation of pool-based designs with its order-book equivalent: the maker whose resting quote you consumed is the liquidity provider, and they are paid from the fee you generated, per fill, instantly, with no program to apply to and no epoch to wait for.
- **Zero on exits and plumbing** keeps the fundamental promises clean: you can always cash out, merge, or redeem without friction, so the fully-collateralized model's guarantees are never taxed.

Effective cost intuition for a taker: at $0.50 the 2% fee is 1 cent on a 50-cent share, i.e. 2% of stake for a shot at 2×. At $0.05 it is 0.1 cents on a 5-cent share for a shot at 20×. The flat rate keeps longshot betting cheap in absolute terms, which is where curve-token action lives.

## 3. Worked examples

1. **Quick bet.** Eve market-buys 100 YES at $0.20: notional $20.00, fee $0.40, total $20.40. The maker (Alice) receives $20.00 + $0.08 rebate. Treasury +$0.32.
2. **Selling out early.** Later Eve sells 100 YES at $0.45 into a resting bid: proceeds $45.00, fee $0.90, Eve nets $44.10. The bid's owner (maker) gets $0.18 rebate. Treasury +$0.72.
3. **A mint.** Frank buys YES at $0.07 (taker) against Grace's resting NO-buy at $0.93: 200 shares mint. Frank's notional 200 × 0.07 = $14.00, fee $0.28 (rebate $0.056 to Grace). Grace pays $186.00 with no fee (she was the maker). $200.00 locks as pair collateral.
4. **The full market** in `03-ORDER-BOOK-SPEC.md` section 8 ends with treasury +$0.50 on $141 of gross losing stakes and $170+ of volume, and the zero-sum audit reconciles to the cent.

## 4. Where every fee lives on-chain

Rebates go straight to `seat.usdc_free` at fill time and are the maker's immediately. The treasury share accrues in `market.fees_accrued`, which also funds that market's keeper fees (creation, uncross, halt, resolve, prune, sweep, finalize); `sweep_fees` moves the remainder to the treasury before close. So each market's operations are paid by its own activity, and the treasury's take is net of the cost of running that market's lifecycle.

## 5. Revenue model

Volume, not open interest, drives fees: shares can change hands many times while the cap only bounds collateral locked at once, and mint/merge churn adds further taker notional. Illustrative scenarios (not forecasts), at 2% taker on one side of each trade and treasury share net of rebates ≈ 1.6% of taker volume:

| Scenario | Markets/day | Avg taker volume/market | Gross fee/day | Treasury/day |
| --- | --- | --- | --- | --- |
| Cautious | 500 | $150 | $1,500 | $1,200 |
| Base | 2,000 | $400 | $16,000 | $12,800 |
| Strong | 5,000 | $1,000 | $100,000 | $80,000 |

Keeper fee outflow at the base scenario is on the order of 1 to 2% of gross fees.

## 6. Costs borne by the platform

| Cost | Order of magnitude | Note |
| --- | --- | --- |
| Sponsored transaction fees | ~5k lamports per user action | At $200/SOL, $0.001; ~0.5% of the average fee per trade; capped per session key by the sponsor service |
| Rent float | 0.053 SOL per concurrent market via the creator keeper | Fully refunded at close; a float, not a cost |
| Indexer + API + RPC | Standard infra | The only real fixed cost |
| Reference keeper operations | Self-funding via keeper fees | |

## 7. Parameter governance

`taker_fee_bps`, `maker_rebate_bps`, keeper fees and dust threshold live in Config behind the timelock (default 24 h) and are bounded in-program: taker fee ≤ 500 bps, rebate between 10% and 50% of the fee. Fee changes apply to fills after the change lands; escrow locked under the old rate releases its surplus on fill. No fee parameter can ever touch redemption: $1.00 per winning share is a constant, not a parameter.
