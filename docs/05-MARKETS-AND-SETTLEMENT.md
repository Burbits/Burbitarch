# Markets and Settlement

What Burbit lets people predict about freshly launched tokens, how each question settles from on-chain data, and the safety rules that make cheating cost more than it can win. The launchpad is an event source: Burbit reads outcomes from it and never participates in it.

---

## 1. The token's lifecycle is the event source

A freshly launched token's life is short, violent and fully on-chain: it launches, it either fills up and graduates to an open market or it dies, and its creator either holds or dumps. Tens of thousands launch per day; only a fraction of a percent graduate; the typical graduation takes minutes; creators are insiders in nearly every launch; most graduated tokens collapse shortly after.

Those are **events**, and events are all Burbit needs. Burbit does not launch tokens, does not run a curve or a pool, does not price tokens, and does not model how they trade. It watches for the outcomes and runs markets on whether they will happen. The relationship is exactly the relationship a sports market has to a match: we do not play, we settle on the score.

## 2. What Burbit reads from a launchpad

Every launchpad keeps a public on-chain account per token that records that token's state and, ultimately, its outcome. Integrating a launchpad means writing one **reader**: read-only code that parses that account and answers four questions.

| The reader answers | Used for |
| --- | --- |
| **Is this account genuine and readable?** | Validation; unreadable means the market voids instead of guessing |
| **Has the outcome happened yet?** (graduated or not) | Settling graduation, timing and race markets |
| **How far along is it, and how much is still needed to finish?** | Milestone markets, order guards, and the market size limit (section 4) |
| **Who is the creator?** | Barring them from their own token's markets; settling creator questions |

That is the whole integration surface. Any bonding-curve launchpad can be supported by adding a reader, permissionlessly, because the data is public and anyone may read it. Nothing is written to the launchpad, no permission is asked, and no other part of Burbit changes per venue.

The first reader targets the dominant Solana launchpad, whose program publishes its account layout and IDL. It locates the token's state account at the address that launchpad derives for the mint, confirms the account is owned by that launchpad's program, checks the type discriminator, and reads the fields it needs: the completion flag (the graduation outcome, which is irreversible once set), the amounts deposited and remaining, and the creator's address. Later fields appended by launchpad upgrades are ignored.

**Reader rules, strictly enforced:** verify owner and address, check the discriminator, parse only the fields whose meaning is stable, tolerate appended bytes, and **reject anything else**. A rejection voids the market and returns everyone's money rather than risking a wrong settlement.

## 3. Question families

This section is the summary; the exhaustive, normative enumeration of every question type, with per-question settlement grades, forcing analyses, offerability classes and the automated listing playbook, is `15-QUESTION-CATALOG.md`, grounded in the behavioral study `14-BONDING-CURVE-BEHAVIOR.md`.

| Family | Examples | Settles from | Who could force it | How offered |
| --- | --- | --- | --- | --- |
| **Graduation** | Graduates before 18:00? Within 10 minutes? | The launchpad's completion flag for the token | YES: paying to finish the token's launch yourself. NO: large holders dumping to stall it | Open book; size limited below the cost of forcing it; creator barred |
| **Graduation timing** | Time to graduate: under 5 min / 5 to 15 / 15 to 60 / never | The recorded completion slot | As above | Multi-outcome market (section 6) |
| **Race** | Which of these 3 launches graduates first? | The competing tokens' completion slots | Finishing the cheapest token in the group | Multi-outcome; limited by the cheapest one to force |
| **Creator dump (open)** | Creator sells ≥ 50% of their launch bag before 18:00? | The creator address read from the launchpad and that wallet's token balance | The creator alone, trivially, from a second wallet | Open book with a small fixed cap (default $50 open interest), clearly labeled |
| **Creator-written rug** | Will the dev rug before expiry? | Custody withdrawal in the bond program itself | The creator, who is the only writer and the only payer | Section 7 |
| **Ecosystem-wide** | Graduations today over/under 80? | A count across launches | Requires forcing many graduations | Open book; settled by bonded proposal with a challenge window |
| **Not offered** | Price at time T, market cap targets, volume, holder counts | n/a | Anyone, cheaply, and profitably under any averaging rule | Never listed |

Price-based questions are excluded by design: simulation of snapshot and averaged settlements showed pump-based manipulation profitable in 42% to 85% of attempts depending on the rule. Only irreversible, binary, account-readable events are listed.

## 4. The market size limit

The core anti-manipulation rule: **a market must never be worth more to win by cheating than cheating costs.**

There is only one way to cheat a graduation market: go to the launchpad and finish the token's launch yourself, so that YES becomes true. Doing that requires putting up the money the token still needs. **That amount is a single number Burbit reads from the launchpad's account**, the same way it reads the outcome flag: how much is still required for this token to finish.

So every market carries a size limit:

```
total collateral in the market  ≤  α × (amount still needed to finish)
```

with **α = 10%**, converted to USDC (for tokens denominated in SOL, using a conservative price floor: the feed price minus its confidence band, rejected if stale). The limit is set when the market opens and **re-checked every single time new shares are created**, so it tightens automatically as the token gets closer to finishing and cheating gets cheaper. Trading existing shares and cashing out are never limited: the cap only governs how much new collateral can enter.

**Why 10% is the right number.** An attacker who pushes a token to completion does not lose everything they spend; they end up holding the tokens they bought and can sell some of that back, so their true net cost is a fraction of what they put in. Measured across the whole range of a token's life, that net cost never falls below roughly a quarter of the gross amount deposited. Capping at 10% of the gross therefore keeps every market below **half** of the attacker's true net cost, at every point, with margin. The attacker's best case is to spend a dollar to win less than fifty cents, before Burbit's fees and before the price impact of their own buying. The parameter lives in Config, is bounded in-program, moves only behind a timelock, and is re-derived as the collector service accumulates live measurements.

The same principle sets every other family's limits, and where it cannot be satisfied, the question is not offered at all. `15-QUESTION-CATALOG.md` states the limit and the reasoning for each question type individually.

## 5. The complete safety rule set

| Rule | Mechanism | What it stops |
| --- | --- | --- |
| Only irreversible events | Graduation cannot be undone; listed families only | Pump-and-revert manipulation |
| No price questions | Not listed at all | Averaging and snapshot games |
| Market size limit | 10% of the amount the token still needs, re-checked live whenever shares are created | Paying to force the outcome and collecting more than it cost |
| Halt on completion | Any instruction that sees the outcome has happened freezes the market | Trading on a decided result |
| Close gap | Trading stops 5 minutes before every deadline | Last-second deadline games |
| Progress guards | Orders self-void outside the progress range their owner set, checked at match | Makers picked off when the token's state jumps |
| Creator barred | The creator address read from the launchpad cannot trade that token's open markets | The best-informed insider trading their own token |
| Short windows | Graduation markets run 5 to 15 minutes by default | Insiders stalling graduations profitably (blocking rises from ~1.7% at 5-minute windows to ~9.1% at 50-minute windows in simulation) |
| Creator-written rug markets only | The creator is the only possible writer of large rug markets | Anyone betting on a rug they can trigger |
| Small caps on open dump markets | Default $50 | Second-wallet self-dealing bounded to pocket change |
| Fail closed | Unparseable launchpad data voids the market at $0.50 per share | Wrong settlements after launchpad changes |
| Conservative price floor | Feed price minus its confidence band when converting the size limit to USDC | Size limits inflated by price-feed noise |

**Residual risk, disclosed rather than hidden:** bundled insider wallets cannot be identified on-chain. They can still block a few percent of would-be graduations and profit on NO. Burbit contains this with short windows, caps, and disclosure in the app, and prices it into the published base rates.

## 6. Multi-outcome markets

For bucketed questions (timing, races), Burbit uses **complete sets** over N mutually exclusive outcomes, exactly one of which pays $1.00:

- `split_set`: $1.00 mints one share of **every** outcome. `merge_set`: a full set burns back into $1.00. The vault invariant generalizes: collateral = sets outstanding × $1.00, and exactly one outcome per set pays.
- Each outcome has its own book in the market's block pool. A **mint** occurs when crossing buy orders across all N outcomes sum to ≥ $1.00; a **merge** when sell orders sum to ≤ $1.00; the matching engine assembles these multi-leg crossings the same way the binary engine assembles pairs.
- **Convert**: holding 1 NO-equivalent on outcome i (that is, being short i) is identical to holding 1 YES on every other outcome. The `convert` instruction lets a holder surrender shares of any k outcomes plus receive $ (k − 1) and shares of the rest, keeping the books arbitraged so all outcome prices sum to ~$1.00. The app's single NO button on a bucket buys the other buckets in one transaction.
- The size limit for a race is set by the **cheapest** token in the group to force.

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
