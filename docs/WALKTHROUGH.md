# The Full Product Walkthrough

**Read this first.** This document walks through Burbit the way you would explain it to someone building it or using it for the first time: what people are actually trading, where the money physically sits at every moment, how YES and NO relate to the order book, who creates the first orders when everyone starts with nothing, what happens to every participant when the event resolves, and when and how the book dies. Every other document in this set is the formal specification of what is described here in plain language. Nothing in here is hand-waving: every mechanism has a number attached and a place in the specs.

---

## 1. The product in one story

It is 2:14 PM. A token called $WOOF launched on a launchpad eleven minutes ago and its bonding curve is 74% full. Thousands of people are watching it, asking one question: does this thing graduate or die?

Burbit turns that question into a market. The moment $WOOF crossed 70%, Burbit automatically opened a market: **"Will $WOOF graduate within 15 minutes?"** On that market there are two kinds of claim you can buy:

- A **YES share**: pays exactly **$1.00** if $WOOF graduates before the deadline, $0 if it does not.
- A **NO share**: pays exactly **$1.00** if it does not graduate, $0 if it does.

Right now YES is trading at **8 cents**. That price IS the market's probability: the crowd thinks there's an 8% chance. Ade thinks the token is botted garbage and will die, so she pays 92 cents each for 20 NO shares: $18.40. If she's right, her 20 shares redeem for $20.00. If she's wrong, they redeem for nothing.

Nine minutes later the curve stalls at 88% and the deadline passes. The market resolves NO. Ade taps Claim and $20.00 lands in her balance. The people who bought YES get nothing; their money is what paid her.

That's the whole product. Everything below is how it actually works underneath.

**One thing to fix in your head before reading on.** The launchpad is not part of Burbit; it is the **event source**, the way a football match is the event source for a sports market. Burbit does not launch tokens, does not run a curve, a pool or an AMM, does not price tokens and does not model how they trade. It watches a launchpad's public accounts, sees whether the outcome happened, and pays out accordingly. The only thing Burbit builds is the market: the order book, the shares, the vault and the settlement. That is also why Burbit works with **any** launchpad: supporting a new one means teaching it to read one more kind of account, and nothing else changes.

## 2. What money is here: USDC on Solana, concretely

All trading, all collateral, and all payouts are in **USDC**, which on Solana is an ordinary SPL token (mint `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`, 6 decimals, so $1.00 = 1,000,000 base units). Your wallet holds USDC in your associated token account (ATA), the standard per-wallet USDC account every Solana wallet manages for you.

Here is the entire monetary plumbing of one Burbit market:

1. **The market vault.** When a market is created, the program creates **one USDC token account for that market**, whose authority is a program-derived address (PDA). Only the Burbit program can move tokens out of it, and the program's code only allows that in the specific situations listed in this walkthrough. Not Burbit the company, not an admin key: the program.
2. **Money goes in exactly one way**: you sign a transaction that transfers USDC from your ATA into the market vault. This happens when you place a buy order (the order's cost is transferred in as part of the same transaction) or when you explicitly deposit.
3. **Once inside, your money is a ledger entry.** The market account keeps a row for you (your "seat") that says how much USDC is yours: how much is **free** (withdrawable any time) and how much is **locked** behind your resting orders. Trading moves numbers between rows. No tokens move during trading at all, which is why matching is fast and cheap.
4. **Money comes out exactly two ways**: you sign a `withdraw` (the program's vault PDA co-signs the token transfer back to your ATA), or after the market ends, the payout sweeper pushes your balance back to your ATA automatically so you never have to come back for it.

So "trades happen in USDC on Solana" means: **one SPL token transfer in when you fund, one out when you exit, and pure ledger arithmetic for everything in between.** There is no wrapped token, no synthetic dollar, no conversion anywhere.

## 3. What people are trading: shares, and why they are not SPL tokens

A YES share is a claim on $1.00 from the market's vault if YES wins. Where does that claim live?

**It is a number in your seat row**, exactly like your USDC balance: your seat has six numbers: `usdc_free`, `usdc_locked`, `yes_free`, `yes_locked`, `no_free`, `no_locked`. Buying 20 NO shares means your `no_free` goes from 0 to 20,000,000 (shares also use 6 decimals, so 1 winning micro-share pays exactly 1 USDC base unit, and redemption math is exact with zero rounding).

**We do not create an SPL token mint per market, and we absolutely do not create anything per bid.** Answering the direct questions:

- **"Do we have to create tokens for each bid?"** No. A bid is a row in the order book inside the market account (price, size, owner, and the locked USDC behind it). Placing it creates nothing but that row; cancelling it deletes the row and unlocks the USDC. Zero mints, zero token accounts, zero NFTs.
- **"Is what I own an SPL token?"** No. It could have been: the alternative design mints two SPL tokens per market (a YES mint and a NO mint) and gives buyers real tokens in their wallets. We rejected it for this product because our markets live 5 to 60 minutes and are created thousands of times a day: two mints plus a token account per trader per side costs roughly 0.002 SOL of rent per user per outcome and leaves millions of dead accounts behind. Ledger shares cost one seat row (~0.0008 SOL, refunded when the market closes) and vanish cleanly.
- **What you give up** by not having SPL shares: they don't show in Phantom, and other protocols can't compose with them. **What you keep**: you can still send shares to another user (`transfer_shares` instruction), sell them on the book, merge them, and redeem them; the app's Positions page is where they show. If composability ever matters, a tokenize-on-request instruction can wrap a position into SPL form later without changing anything else.

## 4. The birth of shares: who creates the asks when everyone starts with nothing

This is the deepest question in the whole system, so here it is slowly.

At market creation the book is empty, the vault is empty, and **nobody owns a single share**. So who can possibly sell? The answer: **nobody needs to sell, because shares are manufactured by two BUYERS on opposite sides.**

The one rule that makes it work: **1 YES + 1 NO together always pay exactly $1.00**, whatever happens (one of them wins). So whenever a YES buyer and a NO buyer are willing to pay a combined $1.00, the program can take their money, lock it in the vault, and print a fresh pair: YES to one, NO to the other. That $1.00 sitting in the vault is precisely what the winning share will redeem later. This is called a **mint**, and it is not a favor or a subsidy: both buyers got exactly what they paid for.

Watch it happen with real numbers on the empty $WOOF book:

- Bola believes. He places: **buy 100 YES at 7¢**. His $7.00 (plus fee reserve) is locked. Nothing matches; his order rests.
- Ade doesn't. She places: **buy 100 NO at 93¢**. Her $93.00 is locked.
- 7¢ + 93¢ = $1.00. **The program locks their combined $100.00 into the vault as pair collateral, creates 100 brand-new pairs, gives 100 YES to Bola and 100 NO to Ade.** The market went from zero shares to 100 pairs with no seller, no market maker, and no house money.

And here is the key perspective shift: **from the YES book's point of view, Ade's "buy NO at 93¢" IS an ask at 7¢.** She is the "seller" Bola traded with, even though she never owned a share. That is who creates the asks at the start: the other side's buyers.

After mints have happened, ordinary selling exists too: Bola now owns 100 YES, and if the curve pumps he can post "sell 100 YES at 20¢", a true ask backed by escrowed shares. And there is a third source: **split**: anyone can pay $1.00 directly to the program and receive 1 YES + 1 NO (no counterparty needed), then quote both sides. That is exactly what the reference quoter bot does to seed thin markets: split $50 into 50 pairs, post YES asks and NO asks around fair value, earn the spread and rebates.

Finally, the **opening auction** solves the very first minute: for the market's first 60 seconds orders only collect (nothing matches), then everything crosses at once at a single clearing price. So the earliest believers and doubters all meet at one fair price instead of racing each other, and the first mints happen in one batch.

## 5. YES book, NO book: two views, one machine

You asked how "YES and NO independently trading their own order book" relates to the order book. Here is the full answer.

In the UI there APPEAR to be two books: a YES book with its own bids and asks, and a NO book with its own bids and asks. But look at what the orders mean:

- Buying NO at 93¢ = betting $0.93 to win $1.00 if NO = **exactly the same bet** as selling YES at 7¢ (receiving $0.07 now, or giving up a $1.00 claim if YES).
- In general: **any NO order at price q is the same economic object as the opposite YES order at price 1 − q.**

So the two books are mirrors of each other:

```
        YES book                          NO book (the same book, reflected)
   BIDS          ASKS                  BIDS            ASKS
  9¢ x 500     12¢ x 300     <=>    88¢ x 300      91¢ x 500
  8¢ x 200     15¢ x 800            85¢ x 800      92¢ x 200
```

Every bid on one side is an ask on the other at 1 − price, with identical size. If you built them as two genuinely independent books, two bad things happen: liquidity splits in half (a YES buyer can't meet a NO buyer), and prices desync until arbitrageurs eat your users (YES at 10¢ while NO at 85¢ means a riskless nickel per share for whoever notices). The fix for both is a rule linking the books: "when a YES bid and a NO bid sum to ≥ $1, mint; when a YES ask and a NO ask sum to ≤ $1, merge." But once that rule exists, the two books are mathematically one structure.

So Burbit **stores one physical book, denominated in YES prices**, and the app **renders two views** of it (the NO tab just displays every price as 1 − p). Your intent is preserved: when you tap "Buy NO at 93¢" the order is recorded as intent BUY_NO and entered into the single book as its YES-equivalent. When it trades, what settles depends on who it met, and there are exactly four cases:

| Your order met | What physically happens | Vault |
| --- | --- | --- |
| Buy YES meets Sell YES | Your USDC goes to the seller, their YES shares go to you | untouched |
| Buy YES meets Buy NO | **Mint**: both your monies lock in the vault, a new pair is created, each of you gets your side | grows by $1/share |
| Sell YES meets Sell NO | **Merge**: your shares pair up and are destroyed, the pair's $1 comes out of the vault and splits between you at the trade price | shrinks by $1/share |
| Buy NO meets Sell NO | Your USDC goes to the seller, their NO shares go to you | untouched |

One book, four possible settlements, and the price you see is always coherent: YES price + NO price = $1.00, enforced by arithmetic, not by hope.

## 6. A trade, frame by frame

What happens in the half-second after Ade taps "Buy 20 NO at 93¢":

1. Her session key (a limited key her wallet authorized once: it can place and cancel orders, it can never withdraw) signs a `place_order` transaction; Burbit's fee-payer service pays the ~$0.001 network fee, so she signs nothing visibly and pays no gas.
2. The program checks the market is live, and **parses $WOOF's live bonding-launchpad state account, which is passed inside the same transaction**. If the token had already graduated, this very transaction would halt the market instead of trading. This is also where progress guards fire: any resting order whose "only valid while curve is between X% and Y%" range no longer matches reality is voided and refunded before it can be hit.
3. Her cost, 20 × $0.93 = $18.60, plus the maximum fee (2%, $0.372), moves from her free balance (topped up from her wallet if needed, in the same transaction) into her locked balance.
4. The matcher walks the best-priced opposing orders, oldest first at each price. Say it finds Bola's resting YES bid at 7¢ for 100: prices sum to $1.00, so this is a **mint** for 20 shares. The vault's pair bucket grows by $20.00 (her $18.60 + his $1.40), her seat gains `no_free += 20`, his gains `yes_free += 20`, his order's remaining size drops to 80.
5. **She is the taker** (her order crossed and executed immediately); **he is the maker** (his order was resting). She pays the taker fee: 2% × $18.60 = $0.372. He pays nothing and receives 20% of her fee ($0.0744) as a rebate, credited to his `usdc_free` on the spot. The remaining 80% accrues to the market's fee bucket.
6. Everything above happened in **one atomic instruction**: match, money movement, maker credit, fee split. There is no pending state, no settlement queue, no moment where a fill exists but its money hasn't moved. If any step failed, the whole trade never happened.
7. If her order hadn't fully filled, the remainder would rest on the book as a new maker order (or be refunded instantly if she'd chosen fill-and-kill). Cancelling later returns the locked money to free in one instruction.

## 7. The end: when the book closes and what happens to every single person

A market stops trading at the FIRST of two events, and this is "when the order book closes":

- **The event happens**: any transaction touching the market observes `complete == true` on the curve (graduation). The market halts that instant; the moment is final and recorded.
- **The close time arrives**: 5 minutes before the question's deadline, a keeper calls `halt`. The 5-minute gap exists so nobody can play games in the final seconds.

**At the halt, the book dies, and here is what happens to each thing on it:**

| What you had at halt | What happens to it |
| --- | --- |
| A resting **bid** (locked USDC) | The full locked amount, fee reserve included, moves back to your free balance. You lost nothing; the bet simply never happened |
| A resting **ask** (locked shares) | Your shares move back to your free share balance, still yours, awaiting resolution like everyone else's |
| **Nothing resting, just shares** | Untouched; you now wait for resolution |
| Free USDC balance | Untouched; withdrawable at any time, before, during, and after all of this |

So no order "survives" resolution and nobody's resting order can be filled against a known outcome. The book empties, then the outcome gets decided.

**Resolution** happens seconds later: a keeper (or anyone; the instruction is permissionless) calls `resolve`, and the program reads the answer directly from the launchpad state account: graduated before the deadline → YES; deadline passed without it → NO. No jury, no vote, no data provider.

**Then the payout, and here is the part that makes the whole system make sense.** Suppose NO won and the vault holds $500 of pair collateral for 500 pairs.

- **The winning side**: every NO share redeems for exactly **$1.00** from the vault. Ade's 20 shares → $20.00 into her free balance. She can withdraw immediately, or do nothing: the **sweeper** crank redeems and pushes every seat's money back to its owner's wallet ATA automatically. There is no claim deadline, ever.
- **The losing side**: every YES share redeems for **$0**. And here is the crucial understanding: **the losers' money is not taken from them at resolution. It left them at mint time.** When Bola paid 7¢ and Ade paid 93¢, their combined $1.00 went into the vault right then. From that moment Bola owned a claim ticket worth "$1 if YES", and resolution merely announced that his ticket is the worthless one and hers is the $1.00 one. Nothing moves against the losers at the end; their shares just expire as paper. That is why the vault can always pay every winner in full with zero platform money: **it has held exactly $1.00 per pair since the moment each pair existed, and each pair has exactly one winner.**
- **People holding both sides**: 1 YES + 1 NO was always worth $1.00, so they could have merged back to cash at any time, even after the halt; at resolution their winning halves redeem for the same $1.00.
- **The zero-sum ledger**: winners' profits + Burbit's fees = losers' losses, to the cent (the fully worked example with fees is in `03-ORDER-BOOK-SPEC.md` section 8, and the test suite asserts this equality after every simulated market).
- **The void case**: if the launchpad's account becomes unreadable (they changed their data layout), the market refuses to guess: it voids, and **every share on both sides redeems for $0.50**, which hands every pair's $1.00 back out in full.

**Then the market disappears.** Once every seat is paid and fees are swept to the treasury, anyone calls `close_market`: the market account, its book, its seats and its vault are deallocated and every bit of rent flows back to whoever paid it. A market is born, lives 15 minutes, pays everyone, and leaves nothing behind but its event history.

## 8. Where every dollar is, at every moment

Every USDC unit in a market's vault is always in exactly one of four buckets, and the program enforces that they sum to the vault's actual token balance:

```
VAULT  =  Σ free balances   (yours, withdrawable now)
        + Σ locked balances (behind resting buy orders)
        + pairs × $1.00     (the collateral behind every share pair)
        + accrued fees      (the treasury's 80% share, pre-sweep)
```

Follow one dollar through Ade's whole journey: wallet ATA → (place order) her locked bucket → (mint) the pair-collateral bucket → (resolution) it becomes redeemable by whoever holds the winning share → (redeem/sweep) winner's free bucket → (withdraw) winner's wallet ATA. At no point does it pass through any account Burbit-the-company controls, and at no point is it anywhere the ledger can't account for.

## 9. Everything else you'd ask next

**Who is my counterparty?** Always another user: either someone selling you their shares, or the opposite-side buyer you minted with. Burbit never holds a position, never takes a side, and has no pool. If a market has believers and no doubters, no trade happens: the bids just sit (and are refunded at halt). A market is exactly as big as the disagreement in it.

**What if only YES buyers show up?** Then YES bids stack up below $1.00 with nothing to cross. No mint requires both sides. Prices you see on an uncrossed book are hopes, not trades; the app labels the last actual trade price separately from the best bid.

**Can I exit before resolution?** Always, two ways: sell your shares on the book (someone buys your position at the current odds), or if you hold both sides, merge for $1.00 per pair instantly with no counterparty needed. This is a core promise: no locked-till-the-end pots.

**Is there leverage or liquidation?** No, structurally. Every share is fully paid for at purchase. Maximum loss = what you paid. Nothing you do in one market touches another market.

**What are the fees, exactly?** Takers (the order that crosses) pay 2% of their USDC notional per fill; makers pay nothing and receive 20% of the taker's fee; the other 80% goes to the treasury and also funds the keeper bots. Deposits, withdrawals, cancels, splits, merges, transfers and redemptions are free.

**How does the market know the token graduated? Who do we trust for the result?** Nobody. The launchpad's own program keeps a public account per token whose `complete` flag flips true at graduation, irreversibly. Burbit's settlement instructions take that account as input and read it. The "oracle" is the launchpad's own state, which is the very thing being bet on.

**What stops someone from buying the graduation to win the YES market?** Market size limits. The program computes, live, what it would cost to force graduation on the launchpad (buy out the rest of the curve, net of resale) and caps the market's total collateral at half that cost. Cheating is guaranteed to cost more than it can win. The full manipulation analysis, including the questions we refuse to list because they can't be protected, is `05-MARKETS-AND-SETTLEMENT.md` and `15-QUESTION-CATALOG.md`.

**What can Burbit-the-company do to my money?** Nothing. The admin keys can pause new activity and register new launchpad readers behind a 24-hour timelock. There is no instruction in the program by which any admin, keeper, or operator moves a user's funds anywhere except to that user.

**What do I actually need in my wallet to start?** USDC and one signature. First trade: approve a session key (one wallet popup, scoped to place/cancel only, expires in 24h), and the order transaction itself carries your USDC into the market. After that, betting is one tap.

**What happens if the token graduates while my order is mid-flight?** The graduation check runs inside your order's own transaction, before matching. Your order becomes a no-op, the market halts, your money never leaves free balance. You cannot buy into a decided market even by one slot.

**When exactly do I stop being able to trade?** The instant `complete` is observed on-chain, or at deadline-minus-5-minutes, whichever is first. Both are checked by the program, not by a server.

---

If you understand this document, you understand the product. The specs behind each section: money and shares (`02-CORE-CONCEPTS.md`), the book and matching (`03-ORDER-BOOK-SPEC.md`), the timeline (`04-MARKET-LIFECYCLE.md`), the questions and their safety (`05`, `14`, `15`), the exact accounts and instructions (`06`), and the machines that keep it moving (`07`, `08`).
