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
3. **Splitters and independent market makers** (needs $1 per pair, **their own**). Anyone can call `split`: put up $1.00, receive 1 YES + 1 NO with no counterparty at all, then quote both sides of the book. Burbit is never one of these participants; the reference quoter is open-source software that third parties run with their own funds.

### 5.5 Does the protocol create orders? No.

You asked whether the protocol creates the asks and people match against it. **It does not, and it deliberately never will.** The moment the protocol posts an order it has taken a position, and it can lose. That is how a house works, not a market. Burbit holds no inventory, quotes no prices, and is never anyone's counterparty. Your counterparty is always another user.

What the protocol does instead is remove the *need* for a first seller:

- **The mint mechanism** (above) means opposite-side buyers are each other's counterparty.
- **The opening auction**: for the first 60 seconds no order matches at all. Orders just collect. Then everything crosses at once at a single clearing price. Nobody gains from being first, there is no bot race at the open, and the market's first shares are minted in one fair batch.
- **Maker rebates**: 20% of every taker fee is paid to the resting order that got hit, so quoting is paid work.
- **Progress guards**: a maker can attach "only valid while the token is between 60% and 85% done" to an order. If the token jumps outside that range the order voids itself before anyone can hit it. This is what makes quoting a fast-moving market survivable.

---

## 5A. Opening a market: what the first prices are and what people actually pay

### 5A.1 There is no starting bid or ask. There cannot be.

A new market opens with an **empty book and an empty vault**. No bid, no ask, no price, and **no money from Burbit**. That is not a gap in the design, it is the design: **the price is the thing the market exists to find**. Posting a starting price would mean guessing, and then standing behind that guess with real money against traders who know more than you do.

**Nobody has to go first with capital.** Section 5 explains the mechanism: the first YES buyer and the first NO buyer are each other's counterparty, and the vault manufactures their shares out of their own combined dollar.

What a user sees on a brand-new market is therefore:

```
  Will $WOOF graduate within 15 minutes?
  ------------------------------------------------
  YES   no bids   no asks        NO   no bids   no asks
  No market price yet.
  Reference: tokens at this stage have historically
  graduated in this window about 5% of the time.
  ------------------------------------------------
  Auction ends in 0:47.  Name your price.
```

The 5% is the **published base rate** from the collector service, shown as context. It is a starting hint for humans, not a price in the book, and the app labels it that way.

### 5A.2 Your bid is automatically the other side's offer

The moment one person names a price, **both sides of the market exist**, because of the complementary relationship you identified: the two prices must sum to $1.00.

Bola bids 8¢ for 100 YES. Nothing has matched, but look at what the book now shows:

| | YES view | NO view |
| --- | --- | --- |
| Best bid | **8¢** (Bola wants to buy YES) | none |
| Best ask | none | **92¢** (anyone can buy NO at 92¢) |

Bola posted one order and it became the **entire NO offer**. A NO believer arriving one second later sees "Buy NO at 92¢, 100 available" and can hit it instantly. They are not buying from Burbit and not buying from an inventory; hitting it mints a fresh pair between the two of them.

So the answer to "what will the starting bid and ask be": **whatever the first person says**, and the opposite side is set automatically at $1.00 minus that price.

### 5A.3 The opening auction discovers the real first price

Because a single first order is one person's opinion, markets open with a 60-second auction where **nothing matches until the end**, and then everything crosses at **one single price**. Nobody gains by being first, so there is no race, and the first price is a collective result rather than one person's guess.

Here is a complete opening, with exactly what everyone pays.

**Orders submitted during the 60 seconds:**

| Person | Wants | Their limit | Size | Cash locked (cost + 2% fee reserve) |
| --- | --- | --- | --- | --- |
| Eve | Buy YES | 12¢ | 50 | $6.12 |
| Bola | Buy YES | 8¢ | 100 | $8.16 |
| Carol | Buy YES | 5¢ | 200 | $10.20 |
| Ade | Buy NO | 92¢ | 100 | $93.84 |
| Dan | Buy NO | 95¢ | 50 | $48.45 |

**As one book, in YES prices** (remember: buying NO at 92¢ is an ask at 8¢):

```
   BIDS (want YES)              ASKS (want NO)
   12¢  x 50   Eve              5¢  x 50   Dan   (buying NO at 95¢)
    8¢  x 100  Bola             8¢  x 100  Ade   (buying NO at 92¢)
    5¢  x 200  Carol
```

**The uncross** finds the price that matches the most shares:

| Candidate price | Buyers willing to pay it or more | Sellers willing to accept it or less | Matched |
| --- | --- | --- | --- |
| 5¢ | 350 | 50 | 50 |
| **8¢** | 150 | 150 | **150** |
| 12¢ | 50 | 150 | 50 |

**Clearing price: 8¢.** Everyone who crosses fills at 8¢, whatever their own limit was.

**What each person actually pays:**

| Person | Gets | At | Pays | Fee (1%) | Total | Refunded |
| --- | --- | --- | --- | --- | --- | --- |
| Eve | 50 YES | 8¢ | $4.00 | $0.04 | **$4.04** | $2.08 (she offered 12¢, paid 8¢) |
| Bola | 100 YES | 8¢ | $8.00 | $0.08 | **$8.08** | $0.08 |
| Ade | 100 NO | 92¢ | $92.00 | $0.92 | **$92.92** | $0.92 |
| Dan | 50 NO | 92¢ | $46.00 | $0.46 | **$46.46** | $1.99 (she offered 95¢, paid 92¢) |
| Carol | nothing | n/a | $0 | $0 | **$0** | order rests at 5¢, money stays locked and is hers |

**Reconciliation:** cash in from all four = $4.00 + $8.00 + $92.00 + $46.00 = **$150.00**, which is exactly the 150 pairs created × $1.00. YES buyers took 150 shares, NO buyers took 150 shares. The vault holds $150.00 and owes exactly $150.00. Fees collected: $1.50, precisely 1% of the minted value.

**After the auction the market has its first price:**

| | YES view | NO view |
| --- | --- | --- |
| Best bid | 5¢ (Carol, 200) | none |
| Best ask | none | 95¢ (200) |
| Last trade | **8¢** | **92¢** |

The book is one-sided again until someone posts the other side. The instant Bola decides to take profit and posts "sell 50 YES at 15¢", it becomes two-sided:

| | YES view | NO view |
| --- | --- | --- |
| Best bid | 5¢ | 85¢ |
| Best ask | 15¢ | 95¢ |
| Mid | **10¢** | **90¢** |

Note the mids sum to $1.00, as they always must.

### 5A.4 What you pay, in plain terms

| Question | Answer |
| --- | --- |
| What do I pay to place an order? | Nothing upfront beyond locking the money the order would cost. Cancel it and the lock releases instantly |
| What if my order never fills? | You pay **nothing**. Every cent comes back at the halt. Unfilled orders are free |
| What does it cost to buy 100 YES at 8¢? | **$8.00**, plus at most 2% fee. If it wins, you receive $100.00 |
| What does it cost to buy 100 NO at 92¢? | **$92.00**, plus at most 2% fee. If it wins, you receive $100.00 |
| Why is the NO side so much more expensive? | Because it is much more likely to be right. You risk $92 to make $8; the YES buyer risks $8 to make $92. Both are paying the market's odds |
| What is the smallest bet? | $1.00 of notional |
| Do I have to think in shares? | No. The app takes a **dollar amount** and shows the shares and the payout: "$20 on NO at 92¢ → 21.74 shares → to win **$21.74**" |

### 5A.5 What if only one side shows up?

Then nothing crosses. There is no price, no trade, and no loss: the orders rest, and at the halt every cent of escrow is returned. A market with only believers and no doubters simply never trades. This is a feature: **Burbit never manufactures a counterparty**, so nobody is ever filled against a price nobody was willing to take.

## 5B. Who funds what: Burbit's capital at risk is zero

Every dollar that exists anywhere in Burbit has a named source. Here is the complete list, with Burbit's contribution to each.

| What needs funding | Who funds it | Burbit's capital at risk |
| --- | --- | --- |
| **The $1.00 backing every share pair** | The two traders who minted it, in the exact proportion of the price they agreed | **$0** |
| **Escrow behind every resting order** | The trader who placed it. Refunded in full if it never fills | **$0** |
| **Every winner's payout** | The losing side's collateral, locked since the moment the pair was created | **$0** |
| **Liquidity and quotes on the book** | Traders and independent market makers, using their own money | **$0.** Burbit posts no orders |
| **Market maker inventory** | That market maker, who buys it by calling `split` with their own funds | **$0** |
| **A market's account rent** (~0.055 SOL) | Whichever keeper created that market. **Refunded in full when it closes**, plus a creation fee | **$0 at risk.** If Burbit runs a keeper it fronts refundable rent as working capital, and third parties can run keepers instead |
| **Network fees on user trades** (~$0.001 each) | The fee sponsor service, paid out of fee revenue | An **operating expense**, like paying for servers. Optional: users can pay their own. Nothing at risk in any market |

Read the first four rows again: **the money in a market is entirely traders' money, and it is always exactly enough.** The vault holds $1.00 per pair from the instant that pair is created until it is redeemed or merged. It is not a treasury, a float, a liquidity pool or an insurance fund, and Burbit cannot add to it or take from it. There is no scenario where Burbit needs capital to open a market, to keep one liquid, or to pay a winner.

**What Burbit earns**: trading fees from both sides, on a ladder that falls as a trader's own volume rises, with the highest-volume makers paid a share of the taker fee rather than charged. That is the entire business model. No spread capture, no position taking, no yield on user funds, no loss when a market goes against anyone.

---

## 6. Maker, taker, market maker: who is who, and who gets paid

### 6.1 The rule, in one question

Maker and taker are decided **per fill**, by one question:

> **Was your order already sitting on the book when the trade happened?**

| | **Maker** | **Taker** |
| --- | --- | --- |
| Your order | Was **resting** on the book, waiting | **Arrived and crossed** immediately |
| What you did | Supplied liquidity (someone could trade because you were there) | Consumed liquidity (you took what was there) |
| Fee | **Free at launch.** Later: 1.00% of your own fill value at the base tier, falling to free at tier 3 | **2.00%** of your own fill value at the base tier, falling to 1.20% |
| Rebate | At the top tiers, **15% to 30% of the taker's fee**, credited instantly, instead of paying a fee | None |
| Execution price | **Your price** is the trade price | You get the maker's price, keeping any improvement |
| Who you are in the UI | The person who used "Set your odds" | The person who tapped "Quick bet" |

It is a property of the **order at that moment**, not of the person. You are not registered as one or the other and you do not choose. Whether you are the maker or the taker on a given fill is simply whether you got there first.

### 6.2 One order can be both

This matters and it surprises people. Say the best ask is 8¢ for 50 shares, and you place a limit buy for 100 at 12¢:

```
  50 shares fill instantly at 8¢    -> on these you are the TAKER (you pay 2%)
  50 shares rest on the book at 12¢ -> from now on you are a MAKER
                                        if someone sells into you later,
                                        you pay nothing and collect a rebate
```

One order, one transaction, both roles. The program accounts for it fill by fill.

### 6.3 Who is a "market maker"?

**It is a behavior, not a status.** There is no registration, no application, no designated-market-maker role, no special account type, no privileged fee tier, and no obligation to quote. The program does not contain the concept. A market maker is simply **anyone who habitually rests orders on both sides to earn the spread and the rebates**, and that can be a bot, a fund, or you on your phone.

Burbit will never have a privileged market maker, because that party would need inventory and a reason to take risk, and the only reason that works at scale is the protocol subsidizing them. Burbit has no capital to subsidize with, by design. Instead it **removes the need** for one: opposite-side buyers mint against each other (section 5), the opening auction batches the first cross, rebates pay quoters, and anyone can `split` $1 into inventory out of thin air.

### 6.4 Who gets paid a rebate, exactly

You are paid a rebate on a fill **if and only if all four are true**:

1. Your order was **resting on the book** when the fill happened, and
2. An **incoming taker order matched against it**, and
3. That fill **generated a fee** (every continuous-trading fill does), and
4. **Your own trailing 30-day volume puts you at tier 4 or 5.** Below that you pay a maker fee instead (0.60% and 0.25% of your own fill value at tiers 1 and 2) or trade free (tier 3, and tier 0 while the launch waiver is in force).

The payment is **15% or 30% of that fill's taker fee**, credited **to your free balance in the same instruction as the fill**. It is not accrued, not claimed later, not a points programme, and not discretionary. A maker is never both charged and paid on the same fill.

**Worked example.** Alice rests "sell 100 YES at 20¢". Eve arrives and takes it.

```
  Eve (taker, tier 0):  notional $20.00,  fee 2.00%  = $0.40  ->  pays $20.40

  If Alice is tier 0:   maker fee 1.00% of $20.00    = $0.20  ->  receives $19.80
  If Alice is tier 3:   free                          = $0.00  ->  receives $20.00
  If Alice is tier 5:   rebate 30% of Eve's $0.40     = $0.12  ->  receives $20.12
```

Same trade, three different outcomes for the maker, decided by nothing but her own measured 30-day volume.

**You do not get a rebate:**

| Situation | Why not |
| --- | --- |
| **You are below tier 4** | You pay a maker fee, or trade free at tier 3 and at tier 0 during the launch waiver. The rebate is the top of the ladder, not the default |
| Fills in the **opening auction** | Nobody was resting; everyone submitted into the same sealed batch. Each side pays half their own taker rate instead, and nobody earns a rebate |
| Fills where **you were the taker** | You consumed liquidity, you pay the taker fee |
| `split`, `merge`, `transfer`, `redeem`, deposit, withdraw, cancel | No fee is charged, so there is nothing to rebate |

**Guaranteeing maker status**: place a **post-only** order. If it would cross and execute immediately, the program rejects it instead of filling it. Quoters use this so a fast-moving book can never accidentally turn their quote into a taker order.

---

## 6B. The market maker's trade, in numbers

They are **independent participants using their own money**. Burbit ships an open-source reference quoter bot so anyone can run one.

**The classic two-sided trade**, step by step:

```
1. Split $1.00        -> receive 1 YES + 1 NO      (cost: $1.00)
2. Post: sell YES at 12¢
3. Post: sell NO  at 90¢
4. If both fill:      -> received 12¢ + 90¢ = $1.02
   Profit: 2¢ per pair, and you hold nothing. Flat, done.
```

They captured the spread. What it costs or earns them depends on their volume tier: at the base tier they keep the full 2¢ while the launch waiver holds and 0.98¢ once it is lifted, at tier 3 they pay nothing and keep the full 2¢, and at tier 5 they also collect 30% of both takers' fees on top. Climbing the ladder is the business.

**Their risk** is that only one side fills. If the YES sells at 12¢ and the NO does not, they are left holding a NO share and are now short the event, directionally exposed. Managing that is the job.

**What Burbit gives them to manage it:**

- **Progress guards**, so a quote dies automatically when the underlying token moves out of the range they priced for.
- **Expiry slots** on every order, a dead-man's switch so stale quotes expire on their own if their bot dies.
- **Post-only orders**, so they never accidentally pay taker fees.
- **Instant cancel**, one instruction, funds unlocked immediately.
- **Published base rates**: Burbit's collector measures real graduation rates by progress band, so quoters have a fair value to quote around instead of guessing.

---

## 7. What price a market starts at, what moves it, and what happens to your value

### 7.1 What price does a market start at?

**None. A market has no price until its first trade, and Burbit puts in no money to create one.**

To be completely unambiguous, because this is the single most important economic property of the design: **Burbit never funds a market, never posts an order, never holds a position, and never acts as anyone's counterparty. Not at launch, not to bootstrap, not ever. The capital required to open a market is zero.** A market is created by writing a question and an empty book into an account. Nothing is deposited into it. The first dollar to enter any market belongs to a trader.

This is not a cost-saving measure, it is a safety property. Any party that posts the opening price is taking a position they can lose. If someone opened every market at 50¢ / 50¢, the first informed trader would buy NO at 50¢ on a market whose honest base rate is nearer 5%, and take roughly 45¢ per share straight out of that party's pocket. Doing that across thousands of markets a day is not a bootstrapping strategy, it is a subscription to being picked off by better-informed traders. **Burbit refuses to be that party, and the design removes the need for one entirely** (section 5: opposite-side buyers mint against each other).

So instead of a posted opening price:

| Stage | What exists |
| --- | --- |
| Market opens | No price. The app shows the **published base rate** for this milestone ("historically ~5%") clearly labeled as a reference, not a market price |
| During the 60-second auction | Orders accumulate, nothing matches, everything is indicative |
| **At the uncross** | The program computes the **single price that matches the most shares**. This is the market's **first price**, discovered collectively |
| Continuous trading | Price is the **midpoint of the best bid and best ask**, moving with every order |

### 7.2 What the price is at any moment

| Book state | Displayed price |
| --- | --- |
| Bids and asks both present | **Mid = (best bid + best ask) / 2.** The headline probability |
| Spread wider than 10¢ | Falls back to the **last traded price**; a wide mid is not meaningful |
| Only bids, no asks | No mid. Shows "best bid 8¢" and the last trade. A bid with nothing opposite is a hope, not a price |
| Nothing at all | No price; the base rate is shown as reference |

And always, in every state: **YES + NO = $1.00**. The two sides are one number displayed two ways.

### 7.3 What actually makes the price move

There is **no formula**. This is the fundamental difference from a pool: in an AMM, price is a function of reserves, so every trade mechanically recomputes it. In an order book, **the price is just a description of the best orders currently resting**. It moves when the set of resting orders changes, which happens in exactly three ways:

1. **Orders get consumed.** A taker eats the best ask. That ask is gone. The next-best ask, at a worse price, becomes the top of the book. The price "moved up" because the cheap offers were bought.
2. **New orders arrive inside the spread.** Someone posts a bid above the old best bid, and the mid moves toward them.
3. **Orders are cancelled or void.** A maker pulls a quote, or a progress guard or expiry kills it. The next level becomes the top of the book.

What *causes* people to do those things is information about the token: progress jumping, a whale buying, the creator selling, the deadline approaching. **The market translates information into orders, and orders into price.** The program does not have an opinion.

### 7.4 What happens to your value as the price moves

This is the part worth being exact about. Follow Bola, who bought 100 YES at 8¢ for $8.00.

| Moment | What happened to the token | YES | NO | Bola's 100 YES marks at | His unrealized P&L |
| --- | --- | --- | --- | --- | --- |
| Open | Listed at 74% | 8¢ | 92¢ | $8.00 | $0 |
| Minute 2 | Jumps to 82% | 13¢ | 87¢ | $13.00 | +$5.00 |
| Minute 5 | A whale buys; 88% | 25¢ | 75¢ | $25.00 | +$17.00 |
| Minute 8 | Creator sells; back to 84% | 18¢ | 82¢ | $18.00 | +$10.00 |
| Minute 9 | **It graduates** | **$1.00** | **$0** | **$100.00** | **+$92.00, now real** |

Three things to take from this:

**1. Your share count never changes.** Bola always holds 100 YES. Price movement does not mint, burn, dilute or multiply anything you own. What changes is what somebody else would pay you for it today.

**2. Gains are unrealized until you act.** At 25¢ Bola's position is *marked* at $25.00, but that is a quote, not money. It becomes real only when he **sells** (a buyer pays him $25.00 out of their own balance) or when the market **resolves** and he redeems for $100.00.

**3. The vault does not move. At all.** This is the crucial one. Throughout that entire table, the vault held exactly $1.00 per pair, unchanged. **Price movement never creates or destroys collateral; it only re-splits the same $1.00 between the two sides.** Look at the columns: when YES went from 8¢ to 25¢, NO went from 92¢ to 75¢. Bola's $17 paper gain is exactly Ade's $17 paper loss. Every cent one side gains, the other side loses, at every instant, because their prices must sum to $1.00.

That is the whole economics in one sentence: **the pair is always worth $1.00; trading only decides how that $1.00 is divided, until the event decides it absolutely.**

### 7.5 "What price starts each market for the winner to go with $1?"

**Any price. The $1 payout is a constant, not a function of the price.**

The starting price, and every price after it, determines only **what you paid**, which is to say **your return multiple**. The winner receives $1.00 per share whether the market opened at 2¢ or at 90¢:

| You bought the winning side at | You paid per share | You receive | Your return |
| --- | --- | --- | --- |
| 2¢ | $0.02 | $1.00 | **50x** |
| 8¢ | $0.08 | $1.00 | **12.5x** |
| 30¢ | $0.30 | $1.00 | **3.3x** |
| 50¢ | $0.50 | $1.00 | **2x** |
| 90¢ | $0.90 | $1.00 | **1.11x** |

And the funding side of it always balances, which is why the $1.00 is always there to pay: **at the moment a pair is created, the two buyers together contribute exactly $1.00.** If YES was bought at 8¢, the NO buyer necessarily paid 92¢. The price decides *who funds how much of the dollar*, never *how big the dollar is*.

```
  YES buyer pays  8¢  ┐
                      ├──>  $1.00 locked in the vault  ──>  paid to whichever side wins
  NO  buyer pays 92¢  ┘
```

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

**What are the fees?** Rates are set by your own trailing 30-day volume. Takers pay 2.00% of their own fill value at the base tier, falling to 1.20% at the top. **Makers trade free at launch**; once the venue has real volume the base maker rate becomes 1.00%, falling to free again at tier 3, and the top two tiers are paid 15% or 30% of the taker's fee instead. Deposits, withdrawals, cancels, splits, merges, transfers and redemptions are always free. Full ladder: `09-FEES-AND-ECONOMICS.md`.

**How does the market know the token graduated?** The launchpad's own program keeps a public account per token whose completion flag flips irreversibly. Burbit's settlement instructions take that account as input and read it. The "oracle" is the thing being bet on.

**What stops someone buying the outcome to win the market?** Size limits. The program reads how much the token still needs to finish, and caps the market's total collateral at 10% of that figure, re-checked every time shares are created. Forcing the outcome always costs several times what winning the market pays.

**What can Burbit the company do to my money?** Nothing. Admin keys can pause new activity and register launchpad readers behind a 24-hour timelock. No instruction exists that moves a user's funds anywhere but to that user.

**What do I need to start?** USDC and one signature. First trade: approve a session key (one popup, place/cancel only, expires in 24 hours); the order transaction carries your USDC in. After that, betting is one tap.

**What if the token graduates while my order is in flight?** The check runs inside your own transaction before matching. Your order becomes a no-op, the market halts, your money never leaves free balance.

---

If you understand this document you understand the product. Build details: [`PRD.md`](PRD.md). Formal specs: money and shares (`02`), the book (`03`), the timeline (`04`), questions and safety (`05`, `14`, `15`), accounts and instructions (`06`), off-chain machinery (`07`, `08`).
