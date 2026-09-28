# The Full Product Walkthrough

**Read this first.** This walks through Burbit the way you would explain it to someone building it or using it for the first time: what people are actually trading, where the money physically sits at every moment, how the YES and NO books relate, **who posts the very first ask when nobody owns anything**, why a share can never be worth more than $1, how market makers fit in, what rent costs on Solana and who pays it, and what happens to every participant when the event resolves.

Every other document is the formal specification of what is described here. The developer-facing build document is [`PRD.md`](PRD.md).

---

## 1. The product in one story

It is 2:14 PM. A token called $WOOF launched on a launchpad eleven minutes ago and it is 74% of the way to finishing. Thousands of people are watching it, asking one question: does this thing graduate or die?

Burbit turns that question into a market. The moment $WOOF crossed 70%, Burbit opened: **"Will $WOOF graduate within 15 minutes?"** On that market there are two claims you can buy:

- A **YES share**: pays exactly **$1.00** if $WOOF graduates before the deadline, $0 if it does not.
- A **NO share**: pays exactly **$1.00** if it does not graduate, $0 if it does.

Right now YES trades at **8 cents**. That price *is* the market's probability: the crowd prices graduation at 8%. Ade thinks the token is botted garbage, so she buys 20 NO shares at 92 cents: $18.40. If she is right, her shares redeem for $20.00. If she is wrong, they are worth nothing.

Nine minutes later the token stalls at 88% and the deadline passes. The market resolves NO. Ade taps Claim and $20.00 lands in her balance. The YES buyers get nothing; their money is what paid her.

**The launchpad is not part of Burbit.** It is the event source, the way a football match is the event source for a sports market. Burbit does not launch tokens, does not run a curve or a pool, does not price tokens. It reads a launchpad's public accounts, sees whether the outcome happened, and pays out. That is also why Burbit works with any launchpad: supporting a new one means reading one more kind of account.

---

## 2. What you are actually trading

### 2.1 Not the token. A claim.

You are never trading $WOOF itself. You are trading a **claim on $1.00** that depends on what $WOOF does. $WOOF could go to a $50M market cap or to zero; it changes nothing about how much a winning YES share pays. It pays $1.00.

### 2.2 A share is a ledger balance, not an SPL token

Every market account keeps a **seat** for each trader: one row with six numbers.

```
Seat: Ade
  usdc_free    18.40     <- withdrawable right now
  usdc_locked   0.00     <- committed to her resting buy orders
  yes_free      0
  yes_locked    0
  no_free      20        <- her 20 NO shares
  no_locked     0        <- would be >0 if she had them up for sale
```

Buying 20 NO shares means `no_free` goes from 0 to 20. That is the entire representation. Shares use 6 decimals like USDC, so one winning micro-share pays exactly one USDC base unit and redemption has zero rounding error.

**We do not create an SPL mint per market, and nothing is created per bid or per order.** Section 9 works through exactly why, with the rent arithmetic, because on Solana this is the decision that makes or breaks the economics.

---

## 3. The money: USDC on Solana, concretely

USDC on Solana is an ordinary SPL token: 6 decimals, so $1.00 = 1,000,000 base units. Mainnet mint `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`. Your wallet holds it in an associated token account (ATA).

**For development and testing we mint our own SPL token with 6 decimals and call it USDC.** The program never hardcodes the mint address; it reads it from its Config account. Switching from test USDC to real USDC is a config value, not a code change. Full setup in [`PRD.md`](PRD.md) section 14.

The entire monetary plumbing of one market:

1. **One vault per market.** When a market is created, the program creates a single USDC token account owned by a program-derived address (PDA). Only the Burbit program moves tokens out of it, and only in the ways listed below. Not Burbit the company. Not an admin key. The program.
2. **Money enters one way**: you sign a transaction transferring USDC from your ATA into the vault. This happens when you place a buy order (the cost rides along in the same transaction) or when you deposit explicitly.
3. **Inside, your money is a ledger number.** Trading moves numbers between seats. **No token transfers happen during trading at all**, which is why a fill costs a fraction of a cent and finishes in one instruction.
4. **Money exits two ways**: you sign `withdraw` (the vault PDA co-signs the transfer back to your ATA), or after the market ends the sweeper pushes your balance back to your wallet automatically so you never have to return for it.

One SPL transfer in, one out, pure arithmetic in between.

---

## 4. Why a share can never be worth more than $1

You asked: *if a winner claims $1, how does that work when a token might be worth more than $1 in the order book?*

**It cannot be.** A YES share never trades above $1, and three independent mechanisms prevent it. The third is the one that really matters and it is the deepest idea in the design.

**1. It would be irrational.** A YES share pays at most $1.00. Paying $1.20 for it is a guaranteed loss of at least 20 cents.

**2. The protocol refuses the order.** Prices are limited to the range $0.001 to $0.999. There is no way to express $1.05 in an order. It is not a policy, it is the data type.

**3. Supply is elastic. This is the real answer.**

A memecoin has fixed supply, so heavy demand has nowhere to go but price: that is how a token 100x's. **Burbit shares have no fixed supply.** Anyone, at any moment, can put up $1.00 and receive 1 YES + 1 NO. So suppose YES were somehow bid at 99.9¢:

> I put up $1.00 → I receive 1 YES + 1 NO → I sell the YES into that bid for 99.9¢ → I now hold a free NO share and I am down a tenth of a cent.

Free money. Everyone would do it, in size, instantly, until the bid fell back below $1. The supply of shares expands to meet any demand at any price approaching $1, so the price physically cannot break through it. The same argument on the other side keeps prices above $0: you can always merge 1 YES + 1 NO back into $1.00.

This is also why **YES price + NO price always equals $1.00**. If YES is 8¢ and NO is 85¢ (sum 93¢), anyone buys both for 93¢, holds a guaranteed $1.00, and pockets 7¢. Arbitrage closes it in seconds.

### So where is the upside?

**In the entry price, not the payout.** Ade bought NO at 92¢ and made 8.7%. Bola bought YES at 8¢; if $WOOF rips and YES trades at 80¢, he sells for a **10x**. If he holds and it graduates, he collects $1.00 for an 8 cent stake: **12.5x**. Longshots pay enormous multiples precisely because the payout is fixed and the entry is cheap.

| You bought YES at | If it wins you get | Return |
| --- | --- | --- |
| 2¢ | $1.00 | 50x |
| 8¢ | $1.00 | 12.5x |
| 25¢ | $1.00 | 4x |
| 50¢ | $1.00 | 2x |
| 92¢ | $1.00 | 1.09x |

---

## 5. Who creates the very first ask

This is the most important question anyone asks about this design, and your instinct about normal markets is exactly right.

### 5.1 Why it looks impossible

On a normal token order book, to post an ask on $WOOF you must **own** $WOOF. On a brand-new market nobody owns anything, so there can be no asks, so bids have nothing to hit, so no price exists. Normal exchanges solve this by paying a designated market maker to go first and carry inventory risk.

### 5.2 Why it is not a problem here

Prediction markets have a structural property normal markets lack: **shares do not need to exist before they are traded. The trade creates them.**

Here is the mechanism. There are four things a person can want, and each maps onto the single book, which is quoted in YES prices:

| What you want | You need | Where it lands on the YES book |
| --- | --- | --- |
| Buy YES at 8¢ | 8¢ of cash | **Bid** at 8¢ |
| Sell YES at 8¢ | 1 YES share | **Ask** at 8¢ |
| Buy NO at 92¢ | 92¢ of cash | **Ask** at 8¢ |
| Sell NO at 92¢ | 1 NO share | **Bid** at 8¢ |

Look at row three. **Buying NO puts an ask on the YES book, and it requires nothing but cash.**

That is the whole answer:

> **The first ask is created by the first person who wants to bet NO.**

They do not think of themselves as a seller or a market maker. They tap "Buy NO at 92¢". The protocol records their intent and places it on the book as an ask at 8¢, because betting 92¢ that it fails is the identical economic position to selling a YES claim for 8¢.

### 5.3 But if both sides are buyers, who hands over the shares?

Nobody. **The vault manufactures them.** Watch the first trade in a market's life, where the book is empty and nobody owns anything:

```
Step 1  Bola: "buy 100 YES at 8¢"     -> $8.00 locked. Rests as a BID at 8¢.
Step 2  Ade:  "buy 100 NO at 92¢"     -> $92.00 locked. Arrives as an ASK at 8¢.
Step 3  They cross. 8¢ + 92¢ = $1.00 exactly.
```

The program takes both amounts, puts **$100.00 into the vault as collateral**, creates **100 brand-new pairs**, and credits:

```
Bola's seat:  yes_free += 100     (he paid $8.00)
Ade's seat:   no_free  += 100     (she paid $92.00)
Vault:        pairs = 100, holding $100.00
```

A market that had zero shares now has 100 YES and 100 NO in existence, fully backed, with **no seller, no market maker, and no protocol money**. That is a **mint**, and it is the engine of the whole system.

The reverse also exists. Once people hold shares, a YES holder selling and a NO holder selling can meet: their shares pair up, the pair is destroyed, and the $1.00 behind it comes out of the vault and splits between them at the trade price. That is a **merge**, and it is why anyone can always cash out even if nobody wants to buy their side.

### 5.4 So asks come from three places

1. **NO buyers** (needs only cash). The dominant source, especially early. Every person betting against the token is supplying the ask side for the people betting for it.
2. **YES holders selling** (needs shares). Real asks, available once mints have happened.
3. **Splitters and market makers** (needs $1 per pair). Anyone can call `split`: put up $1.00, receive 1 YES + 1 NO with no counterparty at all, then quote both sides of the book.

### 5.5 Does the protocol create orders? No.

You asked whether the protocol creates the asks and people match against it. **It does not, and it deliberately never will.** The moment the protocol posts an order it has taken a position, and it can lose. That is how a house works, not a market. Burbit holds no inventory, quotes no prices, and is never anyone's counterparty. Your counterparty is always another user.

What the protocol does instead is remove the *need* for a first seller:

- **The mint mechanism** (above) means opposite-side buyers are each other's counterparty.
- **The opening auction**: for the first 60 seconds no order matches at all. Orders just collect. Then everything crosses at once at a single clearing price. Nobody gains from being first, there is no bot race at the open, and the market's first shares are minted in one fair batch.
- **Maker rebates**: 20% of every taker fee is paid to the resting order that got hit, so quoting is paid work.
- **Progress guards**: a maker can attach "only valid while the token is between 60% and 85% done" to an order. If the token jumps outside that range the order voids itself before anyone can hit it. This is what makes quoting a fast-moving market survivable.

---

## 6. Market makers: who they are and how they earn

You are right that prediction markets have market makers. Here is exactly who they are and what they do.

They are **independent participants using their own money**. Not the protocol, not Burbit, not a privileged role. Anyone can do it, including you, and Burbit ships an open-source reference quoter bot so anyone can run one.

**The classic two-sided trade**, step by step:

```
1. Split $1.00        -> receive 1 YES + 1 NO      (cost: $1.00)
2. Post: sell YES at 12¢
3. Post: sell NO  at 90¢
4. If both fill:      -> received 12¢ + 90¢ = $1.02
   Profit: 2¢ per pair, and you hold nothing. Flat, done.
```

They captured the spread. Add the maker rebate (20% of the taker fees their orders generated) and that is the business.

**Their risk** is that only one side fills. If the YES sells at 12¢ and the NO does not, they are left holding a NO share and are now short the event, directionally exposed. Managing that is the job.

**What Burbit gives them to manage it:**

- **Progress guards**, so a quote dies automatically when the underlying token moves out of the range they priced for.
- **Expiry slots** on every order, a dead-man's switch so stale quotes expire on their own if their bot dies.
- **Post-only orders**, so they never accidentally pay taker fees.
- **Instant cancel**, one instruction, funds unlocked immediately.
- **Published base rates**: Burbit's collector measures real graduation rates by progress band, so quoters have a fair value to quote around instead of guessing.

---

## 7. How a price actually forms

The price you see is the **midpoint between the best bid and the best ask**, exactly as you said. So what happens when the book is thin or one-sided?

| Book state | What the UI shows |
| --- | --- |
| Auction phase (first 60s) | Orders visible but nothing matched. Prices labeled indicative until the opening cross |
| Bids and asks both present | **Mid = (best bid + best ask) / 2.** This is the headline probability |
| Spread wider than 10¢ | Falls back to the last traded price, because a wide mid is not meaningful |
| Only bids, no asks | No mid exists. Shows "best bid 8¢" and the last trade. A resting bid with nothing on the other side is a hope, not a price |
| Nothing at all | No price. The market shows the published base rate for that milestone as a reference, labeled as such |

**The first real price** is set by the opening auction: the program computes the single price that matches the most volume, and everything that crosses fills there, together. After that, ordinary continuous trading takes over and the price walks with every fill.

---

## 8. One trade, frame by frame

What happens in the half second after Ade taps "Buy 20 NO at 92¢":

1. Her **session key** signs the transaction. That is a limited key her wallet authorized once: it can place and cancel orders and **can never withdraw**. Burbit's fee payer service co-signs to cover the ~$0.001 network fee, so she sees no popup and pays no gas.
2. The program checks the market is live and **reads $WOOF's live launchpad account, passed inside this same transaction**. Had the token already graduated, this transaction would halt the market instead of trading. Progress guards are evaluated here too: any resting order whose range no longer matches reality is voided and refunded before it can be hit.
3. Her cost, 20 × $0.92 = **$18.40**, plus the maximum fee (2%, $0.368), moves from free to locked in her seat, pulled from her wallet in this same transaction if her free balance is short.
4. The matcher walks the opposing side, best price first, oldest first at equal prices. It finds Bola's resting bid at 8¢. Prices sum to $1.00, so this is a **mint** for 20 shares: the vault's collateral grows by $20.00, her `no_free` becomes 20, his `yes_free` grows by 20, his order's remaining size drops.
5. **She is the taker** (her order crossed on arrival). **He is the maker** (his order was resting). She pays 2% of $18.40 = **$0.368**. He pays nothing and is credited **$0.0736** (20% of her fee) instantly. The remaining $0.2944 accrues to the fee bucket.
6. All of that happened in **one atomic instruction**. There is no settlement queue, no pending state, no moment where a fill exists but the money has not moved. If any part failed, the trade never happened.
7. Any unfilled remainder rests on the book as a maker order, or is refunded immediately if she chose fill-and-kill.

---

## 9. Why shares are ledger entries and not SPL tokens

You asked directly: *do we mint every token to the vault for each order book pair and map holdings to addresses in the vault?*

What you are describing **is the ledger design**, with an extra step. If the SPL tokens sit in the vault and a table says who owns what, then the table is the truth and the tokens are redundant bookkeeping you are paying rent for. So we keep the table and skip the tokens.

The alternative, real SPL shares that live in users' wallets, is a legitimate design (it is what prediction markets on Ethereum do, where there is no rent). On Solana the arithmetic kills it:

| | SPL tokens per market | Ledger balances (chosen) |
| --- | --- | --- |
| Per market | 2 mints (YES + NO): **0.0029 SOL** | 0 extra |
| Per trader, per side held | 1 token account: **0.00204 SOL** | 1 seat block: **0.00078 SOL** (covers both sides and their cash) |
| A market with 50 traders | ~0.10 SOL, scattered across 50 wallets | **0.039 SOL**, all inside one account |
| Reclaiming it | Each user must close each token account manually. Almost nobody will | Automatic: the whole market account closes and refunds every payer |
| Per fill | Token program CPIs to move shares | Two numbers change |
| At 2,000 markets/day | Hundreds of SOL leaking into abandoned accounts | Recycled continuously |

Our markets live 5 to 60 minutes and get created thousands of times a day. Stranded rent and CPI overhead per fill are not acceptable at that cadence. **This is a case where the Ethereum design does not port to Solana, and rent is the reason.**

**What you keep with ledger shares:** you can sell them on the book, transfer them to another wallet (`transfer_shares`), merge them back to cash, and redeem them. They appear in the app's Positions page.
**What you give up:** they do not show in Phantom, and other protocols cannot compose with them. If that ever matters, a tokenize-on-demand instruction can wrap a position into a real SPL token later without changing anything else in the design.

---

## 10. Rent on Solana: who pays, who gets it back

You are right that PDAs cost rent. The key fact: **rent exemption is a refundable deposit, not a fee.** You lock lamports proportional to account size, and you get every lamport back when the account closes. The design is built around always closing.

| Account | Size | Deposit | Paid by | Refunded |
| --- | --- | --- | --- | --- |
| Market (initial allocation) | ~7.5 KB | **~0.053 SOL** | The keeper that created the market | Yes, at `close_market` |
| Market's USDC vault | 165 B | ~0.002 SOL | Same keeper | Yes, at close |
| One extra block (a seat or a resting order) | 112 B | **~0.00078 SOL** | Whoever's transaction needed it | Yes, at close, to that payer |
| Your wallet's USDC account | 165 B | ~0.002 SOL | You, once, ever | Yes, if you close it |

How the market account grows: it starts with a small pool of 112-byte blocks and **reallocates one block at a time** as traders and orders arrive. Blocks freed by cancelled orders are reused. Nobody pre-pays for capacity nobody uses.

**The keeper's economics**: it fronts ~0.055 SOL per market, earns a creation fee from that market's trading fees, and gets the full deposit back at close. Running the keeper is profitable, which is why it is permissionless: anyone can run one and compete for the fees. A market that never trades still closes and still refunds.

**What the user pays**: their seat block (~0.0008 SOL, refunded) and nothing else. Trading transaction fees are sponsored. If a market closes while they still hold value, the sweeper pushes it to their wallet before closing.

---

## 11. The end: when the book closes and what happens to everyone

Trading stops at the **first** of two events:

- **The event happens.** Any transaction that reads the launchpad and sees the token finished halts the market on the spot. The moment is recorded permanently.
- **The close time arrives.** Five minutes before the question's deadline a keeper calls `halt`. The gap exists so nobody can play games in the final seconds.

### At the halt, the book is destroyed. Every order's fate:

| What you had resting | What happens |
| --- | --- |
| A **bid** (cash locked) | Every cent, fee reserve included, returns to your free balance. You lost nothing; the bet never happened |
| An **ask** (shares locked) | Your shares return to your free share balance, still yours, awaiting resolution |
| Shares, no orders | Untouched, awaiting resolution |
| Free cash | Untouched, withdrawable throughout, before during and after |

No order can be filled against a known outcome, because the book is emptied *before* the outcome is declared.

### Then resolution

A keeper (or anyone; it is permissionless) calls `resolve`. The program reads the launchpad account: finished before the deadline → **YES**; deadline passed without it → **NO**. No jury, no vote, no data provider. If the launchpad's account has become unreadable, the market **voids** instead of guessing, and every share on both sides redeems for $0.50, returning each pair's dollar in full.

### Then payout. Say NO won, with 500 pairs and $500 in the vault.

**The winning side**: every NO share redeems for exactly **$1.00**. Ade's 20 → $20.00. She can withdraw immediately, or do nothing: the **sweeper** redeems and pushes every seat's balance to its owner's wallet automatically. There is **no claim deadline, ever**.

**The losing side**: every YES share redeems for **$0**. And here is the thing that makes the whole system make sense:

> **The losers' money was not taken at the end. It left them at mint time.**

When Bola paid 8¢ and Ade paid 92¢, their combined $1.00 went into the vault **right then**. From that instant, Bola held a ticket worth "$1 if YES" and Ade held "$1 if NO". Resolution merely announced which ticket is the live one. Nothing is seized from losers at the end; their shares simply expire.

That is why the vault can always pay every winner in full with zero platform money: **it has held exactly $1.00 per pair since the moment each pair existed, and every pair has exactly one winner.**

**Holding both sides**: 1 YES + 1 NO was always worth $1.00, so you could merge back to cash at any time, including after the halt.

**The ledger closes to zero**: winners' profits + Burbit's fees = losers' losses, to the cent. The test suite asserts this after every simulated market.

### Then the market disappears

Once every seat is paid and fees are swept, anyone calls `close_market`. The market account, its book, its seats and its vault are deallocated, and **every lamport of rent flows back to whoever paid it**. A market is born, lives fifteen minutes, pays everyone, and leaves nothing behind but its event history.

---

## 12. Where every dollar is, at every moment

Every USDC unit in a market's vault is in exactly one of four buckets, and the program enforces that they sum to the vault's real token balance:

```
VAULT  =  Σ free balances      (yours, withdrawable now)
        + Σ locked balances    (behind resting buy orders)
        + pairs × $1.00        (collateral behind every share pair)
        + accrued fees         (the treasury's share, pre-sweep)
```

One dollar's journey: wallet ATA → (place order) locked → (mint) pair collateral → (resolution) claimable by whoever holds the winning share → (redeem or sweep) winner's free balance → (withdraw) winner's wallet ATA. It never passes through any account Burbit the company controls, and never sits anywhere the ledger cannot account for.

---

## 13. Everything else you would ask next

**Who is my counterparty?** Always another user: someone selling you their shares, or the opposite-side buyer you minted with. Burbit never takes a position.

**What if only YES buyers show up?** Bids stack with nothing to cross. No mint happens without both sides. At halt every bid is refunded. A market is exactly as big as the disagreement inside it.

**Can I exit before resolution?** Always, two ways: sell on the book, or if you hold both sides, merge for $1.00 per pair instantly with no counterparty needed.

**Is there leverage or liquidation?** No. Every share is fully paid at purchase. Maximum loss is what you paid. Nothing in one market can touch another.

**What are the fees?** Takers pay 2% of their USDC notional per fill. Makers pay nothing and receive 20% of the taker's fee. Deposits, withdrawals, cancels, splits, merges, transfers and redemptions are free.

**How does the market know the token graduated?** The launchpad's own program keeps a public account per token whose completion flag flips irreversibly. Burbit's settlement instructions take that account as input and read it. The "oracle" is the thing being bet on.

**What stops someone buying the outcome to win the market?** Size limits. The program reads how much the token still needs to finish, and caps the market's total collateral at 10% of that figure, re-checked every time shares are created. Forcing the outcome always costs several times what winning the market pays.

**What can Burbit the company do to my money?** Nothing. Admin keys can pause new activity and register launchpad readers behind a 24-hour timelock. No instruction exists that moves a user's funds anywhere but to that user.

**What do I need to start?** USDC and one signature. First trade: approve a session key (one popup, place/cancel only, expires in 24 hours); the order transaction carries your USDC in. After that, betting is one tap.

**What if the token graduates while my order is in flight?** The check runs inside your own transaction before matching. Your order becomes a no-op, the market halts, your money never leaves free balance.

---

If you understand this document you understand the product. Build details: [`PRD.md`](PRD.md). Formal specs: money and shares (`02`), the book (`03`), the timeline (`04`), questions and safety (`05`, `14`, `15`), accounts and instructions (`06`), off-chain machinery (`07`, `08`).
