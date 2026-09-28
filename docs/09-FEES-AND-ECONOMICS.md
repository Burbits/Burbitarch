# Fees and Economics

Every fee in the system, who pays it, where it goes, and the revenue model. Burbit earns trading fees only: it never stakes capital, never takes a side, and never earns from user losses.

---

## 1. The fee schedule

Both sides of a trade pay, and both rates **improve with the trader's own trailing 30-day volume**. A maker starts by paying half the taker rate, falls to free, and at the top of the ladder is paid a rebate instead.

### 1.1 The tier ladder

| Tier | Trailing 30-day volume | **Taker** pays, on their own fill value | **Maker** |
| --- | --- | --- | --- |
| 0 | under $250k | **2.00%** | pays **1.00%** of their own fill value (half the base taker rate) |
| 1 | $250k | 1.80% | pays 0.60% |
| 2 | $1M | 1.60% | pays 0.25% |
| 3 | $5M | 1.45% | **free** |
| 4 | $20M | 1.30% | **earns 15%** of the taker fee on that fill |
| 5 | **$50M+** | **1.20%** | **earns 30%** of the taker fee on that fill |

Each side is charged on **its own tier**, independently: a tier-0 taker hitting a tier-5 maker pays 2.00% while that maker earns 30% of it.

| Other actions | Fee |
| --- | --- |
| **Auction fill** | Each filled participant pays **half their own taker rate** (1.00% at tier 0). No rebates: nobody was resting, so nobody supplied liquidity |
| Split, merge, deposit, withdraw, cancel, transfer, redeem | **Zero** |
| Creator rug-market premium | 10% of the creator's YES-sale proceeds |

### 1.2 Formal definition

```
taker_fee   = floor(taker_notional × taker_rate(taker_tier))      # on the taker's own fill value
maker_fee   = floor(maker_notional × maker_rate(maker_tier))      # on the maker's own fill value, tiers 0 to 2
maker_rebate= floor(taker_fee × rebate_share(maker_tier))         # tiers 4 to 5, paid out of the taker fee
treasury    = taker_fee + maker_fee − maker_rebate
```

A maker is **never both charged and paid** on the same fill: tiers 0 to 2 charge, tier 3 is neutral, tiers 4 to 5 pay.

**Two different bases, deliberately.** Fees are a percentage of **your own** fill value, so each side pays in proportion to the capital it actually committed. The rebate is a percentage of the **taker fee collected on that fill**, never of the maker's notional. This is a solvency requirement, not a style choice: on a mint the maker's notional can be eleven times the taker's, so a rebate priced off the maker's notional could exceed the fee collected and the protocol would pay out more than it took in. `treasury` above is positive by construction at every tier, because the maximum rebate share is 30%.

Buy-side escrow locks `cost × 1.02` at placement so a fee can never make a balance negative; unused escrow releases with any unfilled remainder.

### 1.3 How a tier is determined, and why it is frozen per market

A trader's tier is read from their global `TraderStats` account and **stamped into their seat when that seat is created**, inside a transaction they sign themselves. Every subsequent fill reads the tier from the seat, which already lives inside the market account, so **tiering adds no accounts and no cost to the settlement path**. Volume accrues to the seat on each fill and rolls up into `TraderStats` when the market is swept and closed.

The consequence is that a tier is fixed for the life of one market. Since markets live minutes, this is immaterial, and it is what allows a cross-market statistic to price a fill that cannot load cross-market accounts. Full mechanics: `06-ONCHAIN-PROGRAM-SPEC.md`.

**Climbing tiers by wash trading is unprofitable by construction.** Manufacturing volume costs the taker rate plus the maker rate on every faked trade, and returns at most a 30% rebate share, so the exercise always loses money. No anti-wash rule is required; the arithmetic is the defense.

## 2. Why this structure

- **Both sides pay, because both sides use the venue.** A ladder that starts makers at half the taker rate and ends with them earning is a deliberate gradient: it prices casual flow, and it pays the participants who carry the book once they have proven volume.
- **Rates improve with volume, not with status.** There is no application, no market-maker agreement and no discretionary tier. The only input is measured volume, and the measurement is on-chain.
- **The rebate is capped at 30%**, so the treasury retains at least 70% of every taker fee no matter who is trading. The fee system can never run at a loss.
- **Zero on exits and plumbing** keeps the core promises untaxed: you can always cash out, merge or redeem without friction.

### 2.1 Launch configuration: the tier-0 maker rate starts at zero

At launch **every participant is tier 0 by definition**, because no one has any trailing volume yet. So the tier-0 maker rate is not one rung of a ladder at that point, it is the rate every maker in the venue pays. Charging 1.00% there roughly doubles the minimum viable spread for a two-sided quoter (section 3, example 5) at exactly the moment the book has no quoters at all.

Burbit therefore ships with two documented settings of the same ladder:

| Tier | Trailing 30-day volume | Taker | Maker **at launch** | Maker **at steady state** |
| --- | --- | --- | --- | --- |
| 0 | under $250k | 2.00% | **free (waived)** | pays 1.00% |
| 1 | $250k | 1.80% | pays 0.60% | pays 0.60% |
| 2 | $1M | 1.60% | pays 0.25% | pays 0.25% |
| 3 | $5M | 1.45% | free | free |
| 4 | $20M | 1.30% | earns 15% of the taker fee | earns 15% |
| 5 | $50M+ | 1.20% | earns 30% of the taker fee | earns 30% |

Only tier 0 differs, and only tier 0 needs to: the higher rungs are unreachable until real volume exists, and by the time anyone reaches them the waiver has served its purpose.

**Recommended trigger for lifting the waiver:** when at least **10 distinct accounts have reached tier 1** on their own 30-day volume. That is the point at which professional makers demonstrably exist, the ladder is doing real work, and the venue no longer needs to subsidise the act of quoting. Lifting it is a `tiers[0].maker_bps` change from 0 to +100 in Config, behind the standard 24-hour timelock, with **no code change and no migration**.

Everything else in this document describes the steady-state schedule.

## 3. Worked examples

All at tier 0 unless stated.

1. **Quick bet.** Eve market-buys 100 YES at $0.20. Taker notional $20.00, taker fee 2.00% = **$0.40**, she pays $20.40. The maker, Alice, sold at $20.00 and pays her own maker fee 1.00% = **$0.20**, netting $19.80. Treasury takes **$0.60**.
2. **The same trade if Alice were tier 5.** Eve still pays $0.40. Alice pays nothing and **earns 30% of Eve's fee = $0.12**, netting $20.12. Treasury takes $0.28.
3. **Selling out early.** Eve sells 100 YES at $0.45 into a resting bid. Proceeds $45.00, taker fee 2.00% = $0.90, she nets $44.10. The resting bidder pays a maker fee of 1.00% on their own $45.00 = $0.45. Treasury $1.35.
4. **A mint.** Frank buys 200 YES at $0.07 (taker) against Grace's resting NO buy at $0.93. Frank's notional $14.00, fee **$0.28**. Grace's notional $186.00, maker fee 1.00% = **$1.86**. $200.00 locks as pair collateral; treasury takes $2.14. Note that Grace pays more than Frank because she committed thirteen times the capital, which is the point of charging each side on its own fill value.
5. **Two-sided quoting, tier by tier.** A maker splits $1.00, sells YES at 12¢ and NO at 90¢, and collects $1.02 for a gross spread of 2.00¢:

| Maker tier | Maker fee on the two fills | Net spread kept | Minimum viable spread |
| --- | --- | --- | --- |
| 0, launch waiver | none | **2.00¢** | ~0 |
| 0, steady state | 1.00% of 12¢ + 1.00% of 90¢ = 1.02¢ | 0.98¢ | ~1.01¢ |
| 2 | 0.25% of $1.02 = 0.26¢ | 1.74¢ | ~0.26¢ |
| 3 | 0 | 2.00¢ | ~0 |
| 5 | 0, plus 30% of both takers' fees | **above 2.00¢** | negative: quoting pays before the spread |

This is the ladder's intended shape: casual quoting is priced, professional quoting is subsidized.

## 4. Where every fee lives on-chain

Rebates go straight to `seat.usdc_free` at fill time and are the maker's immediately; maker fees are deducted from the maker's proceeds in the same instruction. Neither is accrued, queued or claimed later. The treasury share accrues in `market.fees_accrued`, which also funds that market's keeper fees (creation, uncross, halt, resolve, prune, sweep, finalize); `sweep_fees` moves the remainder to the treasury before close. So each market's operations are paid by its own activity, and the treasury's take is net of the cost of running that market's lifecycle.

## 5. Revenue model

Volume, not open interest, drives fees: shares can change hands many times while the size limit only bounds collateral locked at once, and mint and merge churn adds further notional.

Effective take per dollar of traded notional depends on the mix of tiers on each side:

| Both sides at | Taker pays | Maker pays or earns | **Protocol keeps** |
| --- | --- | --- | --- |
| Tier 0, launch waiver in force | 2.00% | free | **2.00%** |
| Tier 0, steady state | 2.00% | pays 1.00% | **3.00%** |
| Tier 0 taker, tier 3 maker | 2.00% | free | **2.00%** |
| Tier 5 both | 1.20% | earns 0.36% (30% of the taker fee) | **0.84%** |

Illustrative scenarios (not forecasts), at a blended 2.0% of traded notional:

| Scenario | Markets/day | Avg volume/market | Gross fee/day | Treasury/day after rebates |
| --- | --- | --- | --- | --- |
| Cautious | 500 | $150 | $1,500 | ~$1,400 |
| Base | 2,000 | $400 | $16,000 | ~$14,500 |
| Strong | 5,000 | $1,000 | $100,000 | ~$88,000 |

Keeper fee outflow at the base scenario is on the order of 1 to 2% of gross fees. Because the ladder lowers rates as volume grows, the blended take declines as the venue matures, which is the intended trade of rate for volume.

## 6. Costs borne by the platform

| Cost | Order of magnitude | Note |
| --- | --- | --- |
| Sponsored transaction fees | ~5k lamports per user action | At $200/SOL, $0.001; ~0.5% of the average fee per trade; capped per session key by the sponsor service |
| Rent float | 0.053 SOL per concurrent market via the creator keeper | Fully refunded at close; a float, not a cost |
| Indexer + API + RPC | Standard infra | The only real fixed cost |
| Reference keeper operations | Self-funding via keeper fees | |

## 7. Parameter governance

`taker_fee_bps`, `maker_rebate_bps`, keeper fees and dust threshold live in Config behind the timelock (default 24 h) and are bounded in-program: taker fee ≤ 500 bps, rebate between 10% and 50% of the fee. Fee changes apply to fills after the change lands; escrow locked under the old rate releases its surplus on fill. No fee parameter can ever touch redemption: $1.00 per winning share is a constant, not a parameter.
