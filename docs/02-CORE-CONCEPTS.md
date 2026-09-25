# Core Concepts

The economic primitives everything else is built on: shares, pairs, collateral, prices, and the invariants that make the system solvent by construction.

---

## 1. Markets and shares

A **market** is one yes/no question about one token, with a deadline. Examples: "Will token X graduate before 18:00?", "Will X's creator sell half their bag within 30 minutes?"

Each market has two outcome shares, **YES** and **NO**:

- A winning share pays **$1.00** (1,000,000 USDC base units).
- A losing share pays **$0**.
- If a market is **voided**, every share on both sides pays **$0.50**, which returns each pair's collateral in full.

Shares are ledger balances inside the market account (see `01-ARCHITECTURE.md` section 5). They can be traded on the market's book, transferred to another user with `transfer_shares`, merged back into USDC, and redeemed after resolution.

## 2. The complete pair and full collateralization

1 YES + 1 NO always pays exactly $1.00, whichever way the market resolves. Therefore the program creates shares **only in pairs**, and each pair is backed by exactly $1.00 locked in the market vault:

- **Split:** lock $1.00, receive 1 YES + 1 NO. Available directly via the `split` instruction and implicitly whenever two buyers on opposite sides trade (a mint).
- **Merge:** return 1 YES + 1 NO, receive $1.00 back. Available directly via `merge` and implicitly whenever two sellers on opposite sides trade.
- **Redeem:** after resolution, each winning share pays $1.00 from the vault.

This is what makes Burbit solvent by construction. The vault always holds exactly $1.00 per pair outstanding, each pair has exactly one winning share, and so the vault can always pay every winner in full without any solvency assumption, insurance fund, margin engine or liquidation mechanism. Nothing about the platform's balance sheet is involved: winners are paid from the losers' locked collateral, peer to peer.

## 3. Price is probability

Prices are quoted as a fraction of the $1.00 payout, from $0.001 to $0.999.

- A YES share trading at $0.25 costs 25 cents and means the market prices graduation at 25%.
- Because a pair is worth exactly $1.00, arbitrage pins **YES price + NO price = $1.00**. Buying NO at $0.75 is the same economic act as selling YES at $0.25, and the order book treats it as exactly that (see `03-ORDER-BOOK-SPEC.md`).

The app displays prices in cents ("25¢") in trading contexts and as probability ("25% chance") in discovery contexts. They are the same number.

## 4. Units, precisely

| Quantity | Unit | Representation |
| --- | --- | --- |
| USDC amounts | base units, 6 decimals | `u64`; $1.00 = 1,000,000 |
| Share sizes | micro-shares | `u64`; 1 share = 1,000,000 units, so 1 winning micro-share pays exactly 1 USDC base unit |
| Prices | mills (thousandths of a dollar) | `u16` in [1, 999]; $0.57 = 570 |
| Curve progress | basis points | `u16` in [0, 10000] |

Cost of a trade in USDC base units: `cost = price_mills × size_microshares / 1000`, computed in `u128` and floored. Because micro-shares map one-to-one to payout base units, redemption is a plain move of `size` units from vault to seat with no rounding at all.

**Tick regime.** Orders must be priced on the current tick:

- Standard tick: **$0.01** (price divisible by 10 mills).
- Fine tick: **$0.001**, permitted for any order priced below $0.05 or above $0.95, so longshot odds stay expressible without letting mid-range queues fragment.

**Minimum order:** $1.00 notional (`cost ≥ 1,000,000` for buys, `size × payout ≥ 1,000,000` equivalent for sells).

## 5. Open interest and the cap

**Open interest** = pairs outstanding × $1.00 = the pair collateral locked in the vault. It is the maximum any participant set can win in aggregate, which makes it the quantity Burbit caps for safety:

```
open_interest ≤ α × forcing_cost_usd        (α = 50%)
```

`forcing_cost_usd` is what an attacker would spend to force the market's outcome on the launchpad (for graduation questions: buying out the rest of the curve, minus what they would recover selling into the post-graduation pool), computed on-chain from the token's live curve state by the market's reader, denominated in SOL, and converted to USD using the Pyth SOL/USD price with its confidence interval subtracted (a conservative floor). The cap is enforced at every mint and every split, and recomputed from live state each time, so it tightens automatically as the curve fills and forcing gets cheaper. Transfers and merges are never blocked by the cap. Full details and the manipulation analysis are in `05-MARKETS-AND-SETTLEMENT.md`.

## 6. Where money can be, exhaustively

Every USDC base unit inside a market's vault is in exactly one of four buckets, all tracked in program state:

1. **Seat free balance** (`usdc_free`): withdrawable by the owner at any time, in any market state.
2. **Seat locked balance** (`usdc_locked`): escrow behind the owner's resting buy orders (cost plus maximum fee). Released to free on cancel, void, halt, or the unfilled remainder of a fill.
3. **Pair collateral** (`pairs × $1.00`): backing outstanding pairs. Enters at mint/split, leaves at merge and redemption.
4. **Accrued fees** (`fees_accrued`): the treasury's 80% share of collected taker fees, swept by the treasury key. (Maker rebates are credited straight to maker seats at fill time and so live in bucket 1.)

Shares behave the same way: `yes_free / yes_locked / no_free / no_locked` per seat, with locked meaning "escrowed behind a resting sell order".

## 7. The invariants

These hold after every instruction. The program asserts the cheap ones on-chain; the differential test suite (see `12-TESTING.md`) asserts all of them after every step of randomized adversarial sequences.

1. **Vault conservation.** `vault.amount = Σ seats.usdc_free + Σ seats.usdc_locked + pairs × 1_000_000 + fees_accrued`.
2. **Share symmetry.** Before resolution, `Σ (yes_free + yes_locked) = Σ (no_free + no_locked) = pairs`.
3. **Cap safety.** At the moment of every mint and split, `pairs × 1_000_000 ≤ cap` computed from the live curve.
4. **No negative balances, ever.** Every buy escrows its maximum cost plus maximum fee at placement; every sell escrows its shares. Matching can only move amounts that are already locked.
5. **Bounded settlement.** After resolution and full redemption (or sweep), the vault holds exactly `fees_accrued`, and after `sweep_fees` and `close_market` it holds zero.
6. **Book consistency.** Every resting order's escrow is exactly reconstructible from its price, size and side; the trees contain no order referencing a freed seat; best-price caches equal the true tree extremes.
7. **Outcome permanence.** A market's outcome is set at most once, and only by `resolve` reading the accounts specified for its question type.

## 8. Positions are assets, not contracts

A consequence worth stating explicitly, because it defines the product: a Burbit position is a **fully-paid asset**. Once you own YES shares:

- nothing can liquidate you, at any price, ever;
- you owe nothing further; your maximum loss was paid at purchase;
- you can sell on the book, transfer to another wallet, merge against NO, or redeem at settlement;
- no funding rate, margin call, or oracle mark touches you.

There is no leverage anywhere in the system, no synthetic exposure, and no protocol-owned pool taking the other side. Every market is exactly as big as the opposing convictions of its participants, dollar for dollar.
