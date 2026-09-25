# Markets and Settlement

What Burbit lets people predict about bonding-curve tokens, how each question settles from on-chain data, and the safety rules that make cheating cost more than it can win.

---

## 1. The bonding-curve lifecycle is the product

A bonding-curve token's life is short, violent and fully on-chain: launch, a race up the curve, then either graduation to an open pool or death. Tens of thousands launch per day; only a fraction of a percent graduate, the median graduation takes minutes, creators are insiders in nearly every launch, and most graduated tokens collapse shortly after. There is nowhere to express a view on any of this except buying the token itself, which means taking the insiders' risk. Burbit turns each measurable, irreversible event in that lifecycle into a market.

## 2. Launchpad integration is a reader, nothing more

Burbit operates no curve and no AMM; it integrates a launchpad by shipping a **reader**: read-only code that parses that launchpad's public per-token account and implements four functions (parse and validate, curve progress, forcing cost, creator address). Any bonding-curve launchpad can be integrated this way, permissionlessly, because public account data is readable by anyone. Every reader rejects anything it does not exactly recognize, which routes layout changes into the void path instead of wrong settlements.

The first reader targets the dominant Solana launchpad's public program. Facts the reader relies on, all from the launchpad's own published program documentation and IDL:

- Program id: `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P`.
- Per-token curve account: PDA `["bonding-curve", mint]`, owned by that program.
- Layout (after the 8-byte discriminator): `virtual_token_reserves: u64`, `virtual_sol_reserves: u64`, `real_token_reserves: u64`, `real_sol_reserves: u64`, `token_total_supply: u64`, `complete: bool`, `creator: Pubkey`, plus later-appended fields the reader ignores.
- `complete` starts false and is set true at the end of the buy that empties `real_token_reserves`; migration to the open pool follows permissionlessly. **`complete == true` is graduation**, and it is irreversible.
- Curve parameters used by the cost model: virtual reserves 30 SOL × 1.073B tokens, 793.1M tokens sellable on the curve, ~85 SOL real reserve at graduation, 1.25% fee per side.

Reader rules: verify the account owner is the registered program and the address equals the PDA for the mint; check the discriminator; parse the stable prefix by offset; tolerate appended trailing bytes; **reject anything else**. Rejection voids the market rather than risking a wrong settlement. Additional launchpads are added one reader at a time under the same contract (parse, progress, forcing cost, creator address).

## 3. Question families

This section is the summary; the exhaustive, normative enumeration of every question type, with per-question settlement grades, forcing analyses, offerability classes and the automated listing playbook, is `15-QUESTION-CATALOG.md`, grounded in the behavioral study `14-BONDING-CURVE-BEHAVIOR.md`.

| Family | Examples | Settles from | Who could force it | How offered |
| --- | --- | --- | --- | --- |
| **Graduation** | Graduates before 18:00? Within 10 minutes? | The curve account's `complete` flag | YES: buying out the curve (1 to 19.5 SOL depending on progress). NO: insiders dumping to stall | Open book; open interest capped by forcing cost; creator barred |
| **Graduation timing** | Time to graduate: under 5 min / 5 to 15 / 15 to 60 / never | The recorded `completion_slot` | As above | Multi-outcome market (section 6) |
| **Race** | Which of these 3 launches graduates first? | The competing curve accounts' completion slots | Buying out the cheapest curve in the group | Multi-outcome; capped by the cheapest forcing cost |
| **Creator dump (open)** | Creator sells ≥ 50% of their launch bag before 18:00? | The creator address from the curve account and that wallet's token account balance | The creator alone, trivially, from a second wallet | Open book with a small fixed cap (default $50 open interest), clearly labeled |
| **Creator-written rug** | Will the dev rug before expiry? | Custody withdrawal in the bond program itself | The creator, who is the only writer and the only payer | Section 7 |
| **Ecosystem-wide** | Graduations today over/under 80? | A count across launches | Requires forcing many graduations | Open book; settled by bonded proposal with a challenge window |
| **Not offered** | Price at time T, market cap targets, volume, holder counts | n/a | Anyone, cheaply, and profitably under any averaging rule | Never listed |

Price-based questions are excluded by design: simulation of snapshot and averaged settlements showed pump-based manipulation profitable in 42% to 85% of attempts depending on the rule. Only irreversible, binary, account-readable events are listed.

## 4. Forcing cost and the open-interest cap

The core anti-manipulation rule: **a market must never be worth more to cheat than the cheat costs.**

- `forcing_cost(curve)` = SOL to buy out the remainder of the curve on the launchpad (moving the price up it, paying its fees) minus SOL recovered by selling those tokens into the post-graduation pool. Computed in integer math by the reader from live reserves. Reference values for the first launchpad's parameters:

| Curve progress | Forcing cost (SOL) |
| --- | --- |
| 0% | 19.52 |
| 30% | 16.28 |
| 50% | 13.12 |
| 70% | 8.62 |
| 80% | 5.70 |
| 90% | 2.46 |
| 95% | 0.95 |

- The cap: `open_interest ≤ α × forcing_cost × sol_usd_floor`, α = 50%, where `sol_usd_floor` is the Pyth SOL/USD price minus its confidence interval, staleness-checked (≤ 30 slots). Enforced at market creation and re-enforced at **every mint and split** from the live curve, so the ceiling tightens as forcing gets cheaper. Transfers and merges are never capped.
- Consequence: an attacker who forces graduation to win the YES side can win at most the open interest, which is at most half what the forcing cost them, before even paying Burbit's fees or slippage. Simulated attacks (150 runs across crowd behaviors) never beat the on-chain formula.

## 5. The complete safety rule set

| Rule | Mechanism | What it stops |
| --- | --- | --- |
| Only irreversible events | Graduation cannot be undone; listed families only | Pump-and-revert manipulation |
| No price questions | Not listed at all | Averaging and snapshot games |
| Open-interest cap | α × forcing cost, live, at every mint/split | Buying the outcome to collect the other side |
| Halt on completion | Any instruction seeing `complete == true` freezes the market | Trading on a decided result |
| Close gap | Trading stops 5 minutes before every deadline | Last-second deadline games |
| Curve guards | Orders self-void outside their progress range, checked at match | Makers picked off by faster curve readers |
| Creator barred | The creator address from the curve account cannot trade that token's open markets | The best-informed insider trading their own token |
| Short windows | Graduation markets run 5 to 15 minutes by default | Insiders stalling graduations profitably (blocking rises from ~1.7% at 5-minute windows to ~9.1% at 50-minute windows in simulation) |
| Creator-written rug markets only | The creator is the only possible writer of large rug markets | Anyone betting on a rug they can trigger |
| Small caps on open dump markets | Default $50 | Second-wallet self-dealing bounded to pocket change |
| Fail closed | Unparseable launchpad data voids the market at $0.50 per share | Wrong settlements after launchpad changes |
| Conservative price floor | Pyth price minus confidence for cap conversion | Cap inflation via oracle noise |

**Residual risk, disclosed rather than hidden:** bundled insider wallets cannot be identified on-chain. They can still block a few percent of would-be graduations and profit on NO. Burbit contains this with short windows, caps, and disclosure in the app, and prices it into the published base rates.

## 6. Multi-outcome markets

For bucketed questions (timing, races), Burbit uses **complete sets** over N mutually exclusive outcomes, exactly one of which pays $1.00:

- `split_set`: $1.00 mints one share of **every** outcome. `merge_set`: a full set burns back into $1.00. The vault invariant generalizes: collateral = sets outstanding × $1.00, and exactly one outcome per set pays.
- Each outcome has its own book in the market's block pool. A **mint** occurs when crossing buy orders across all N outcomes sum to ≥ $1.00; a **merge** when sell orders sum to ≤ $1.00; the matching engine assembles these multi-leg crossings the same way the binary engine assembles pairs.
- **Convert**: holding 1 NO-equivalent on outcome i (that is, being short i) is identical to holding 1 YES on every other outcome. The `convert` instruction lets a holder surrender shares of any k outcomes plus receive $ (k − 1) and shares of the rest, keeping the books arbitraged so all outcome prices sum to ~$1.00. The app's single NO button on a bucket buys the other buckets in one transaction.
- The cap for a race is set by the **cheapest** forcing cost in the group.

## 7. Creator-written rug markets

"Will the dev rug?" is the most-asked question and the one where a single person decides the answer. Burbit's design makes the creator the only writer and the only payer:

1. **Opt-in.** A creator places their launch bag (or the token authority) into Burbit's bond escrow. They can withdraw it at any time.
2. The creator posts a USDC bond B and splits it into B pairs. The NO side ("no rug") is locked to the creator's seat and cannot be transferred or sold.
3. The creator sells YES ("rug") shares on the market's book. Buyers hold them as insurance; the price is the market's live read of rug risk.
4. **Withdrawing custody before expiry is the rug event**, recorded in the same instruction: outcome YES, the bond pays YES holders $1.00 per share, the creator's locked NO is worthless.
5. At expiry with custody intact: outcome NO, the creator redeems the bond and keeps whatever the YES sales earned them.

The app shows a **coverage ratio** (bond ÷ current liquidation value of the escrowed bag) so buyers can judge how credible the promise is. Honesty earns the creator premium income; rugging costs the bond. No oracle, no judgment call, no counterparty but the creator's own money.

## 8. Ecosystem-wide markets and the bonded proposal

Questions spanning many launches ("graduations today over/under 80") have no single account to read, so they settle optimistically:

1. After the deadline, anyone proposes the outcome with a USDC bond (default $200).
2. A challenge window runs (default 2 hours). An unchallenged proposal becomes the outcome; the proposer recovers the bond plus a proposer fee.
3. A challenger posts an equal bond with the opposing outcome. The market enters a second window in which either side can be re-supported; the final unchallenged answer stands, the losing bond pays half to the winner and half to the treasury.
4. If no valid proposal survives by a long-stop time (24 hours), the market voids.

This family is a minority product line, sized so the bond exceeds plausible open interest, and is the only place in Burbit where a human assertion is part of settlement.

## 9. Listing policy and base rates

Markets are listed where interest is highest and manipulation cheapest to bound: graduation questions at 70% to 90% progress with 5-to-15-minute windows by default. Published milestone base rates (measured continuously by the collector service and replacing simulation-derived priors before mainnet) seed the reference quoter's fair values:

| Progress reached | Graduates within 5 min | 15 min | 60 min |
| --- | --- | --- | --- |
| 50% | 1.8% | 2.6% | 2.9% |
| 70% | 3.9% | 5.4% | 6.0% |
| 80% | 7.4% | 10.2% | 11.3% |
| 90% | 22.0% | 27.3% | 29.1% |

Creator-behavior rates (share of creators selling within 1/5/15 minutes) are collected the same way for dump-market pricing.
