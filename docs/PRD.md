# Burbit: Product Requirements Document (Developer Build Document)

**Version 1.0. Status: build-ready.**

This is the single document a developer needs to understand *what* Burbit is, *why* every piece exists, and *what exactly to build*. It states the product, the domain model, every functional requirement, the complete account and instruction design, the money flows with worked numbers, the test environment, and the acceptance criteria that define done.

Companion documents: [`WALKTHROUGH.md`](WALKTHROUGH.md) explains the product in plain language and should be read first. The numbered specs (`00` to `15`) are the formal references this PRD points into.

---

## Contents

1. [Product definition](#1-product-definition)
2. [Problem and opportunity](#2-problem-and-opportunity)
3. [Goals and non-goals](#3-goals-and-non-goals)
4. [Users](#4-users)
5. [Product principles](#5-product-principles)
6. [Domain model](#6-domain-model)
7. [How the market works: the mechanism, in full](#7-how-the-market-works-the-mechanism-in-full)
8. [Functional requirements](#8-functional-requirements)
9. [System architecture](#9-system-architecture)
10. [On-chain data model and rent](#10-on-chain-data-model-and-rent)
11. [Instruction reference](#11-instruction-reference)
12. [The matching engine](#12-the-matching-engine)
13. [Money flows, worked](#13-money-flows-worked)
14. [Test environment and test USDC](#14-test-environment-and-test-usdc)
15. [Non-functional requirements](#15-non-functional-requirements)
16. [Acceptance criteria and test plan](#16-acceptance-criteria-and-test-plan)
17. [Build sequence](#17-build-sequence)
18. [Risks and open decisions](#18-risks-and-open-decisions)

---

## 1. Product definition

**Burbit is an order-book prediction market for freshly launched tokens on Solana.**

Users buy and sell YES and NO shares on questions about what a newly launched token will do: will it graduate, how fast, will the creator dump, will it crash after graduating. Each share pays exactly **$1.00 in USDC** if its outcome is correct and $0 otherwise. Every share pair in existence is backed by exactly $1.00 of USDC locked in that market's vault, so the market is always fully solvent and the platform never takes a position.

Markets settle by **reading the launchpad's own public on-chain accounts** inside the settlement instruction. No oracle, no data provider, no human resolver, no voting.

**Burbit builds no bonding curve, no AMM, no pool, and no launchpad.** The launchpad is the event source, as a football match is the event source for a sports market. Any bonding-curve launchpad can be supported by adding one read-only account parser.

---

## 2. Problem and opportunity

Tens of thousands of tokens launch daily on Solana launchpads. Under 1% graduate. The typical graduation takes minutes. Creators are insiders in nearly every launch and most graduated tokens collapse within twenty minutes.

Despite enormous attention on this cycle, **the only way to express a view is to buy the token itself**, which forces you to take insider risk, sniper risk and total-loss risk to bet on something as simple as "this will not graduate". There is no way to short, no way to size a view, and no way to bet on creator behavior at all.

Existing attempts are pooled parimutuel products resolved from off-chain data feeds: no price discovery, no ability to exit before settlement, and a trust dependency on the operator's data. **Nobody runs a real order book with trustless on-chain settlement on this event space.**

---

## 3. Goals and non-goals

### Goals

| # | Goal | Measure of success |
| --- | --- | --- |
| G1 | A real CLOB: limit orders, price-time priority, maker/taker, cancel, partial fills | Users set their own odds and exit before settlement |
| G2 | Full collateralization, always | Vault balance reconciles to the ledger after every instruction, in all tests |
| G3 | Trustless settlement seconds after the event | Time from on-chain event to claimable funds under 60 seconds |
| G4 | Nobody can profitably game a market | Forcing an outcome always costs multiples of what winning pays, proven at every size limit |
| G5 | Betting feels instant and free | One tap, no wallet popup, no visible fee, sub-second confirmation |
| G6 | No custody, ever | No instruction exists that moves user funds anywhere but to that user |
| G7 | Rent-neutral operation | Every market refunds 100% of its rent at close |
| G8 | Any launchpad can be integrated | Adding one requires only a new reader, no other change |

### Non-goals

- No AMM, no protocol-owned liquidity, no house-taken positions, ever.
- No leverage, margin, funding or liquidation.
- No custodial balances, no embedded-wallet custody, no off-chain matching authority in v1.
- No price-level or market-cap questions (see `15-QUESTION-CATALOG.md` for the exclusion register and why).
- No governance token in scope.

---

## 4. Users

| Persona | What they want | What they need from the product |
| --- | --- | --- |
| **The degen** watching launches all day | To bet a view instantly without buying the token | One-tap Quick Bet, live launch feed with odds, sub-second fills, instant cash-out |
| **The sharp** with a read on launch quality | To size a view and exit at the right price | Limit orders, depth, progress guards, published base rates, public settlement rules |
| **The market maker / quoter bot** (a behavior, not a registered role) | Spread and rebates on high-frequency markets | Post-only orders, expiry slots, progress guards, instant cancel, maker rebates, a documented SDK |
| **The token creator** | To signal honesty and earn from it | Creator-written markets, bonded custody, a coverage ratio their community can see |
| **The keeper operator** | Fees for running infrastructure | Permissionless, idempotent, fee-paying instructions and an open-source keeper binary |
| **The integrator** | Data and programmatic trading | Public REST/WebSocket APIs, an SDK, no permission required |

---

## 5. Product principles

1. **The program is the exchange.** Matching, custody, settlement and resolution are enforced on-chain. No off-chain component can lose or misdirect funds.
2. **Crankless and atomic.** A fill moves money, credits both sides and charges fees in a single instruction. There is never a state where a trade exists but its money has not moved.
3. **Read the event source, never trust anyone about it.** Every state-dependent instruction parses the launchpad account passed into it, and fails closed to a refunding void if it cannot.
4. **Everything off-chain is permissionless and replaceable.** Keepers and indexers earn fees or serve convenience; none are trusted with funds.
5. **The admin plane cannot touch money.** Pause and parameter changes only, behind a timelock.
6. **Refuse questions we cannot protect.** If an outcome can be forced or faked cheaply, it is not listed, and the reason is documented.

---

## 6. Domain model

| Entity | Definition | Lives in |
| --- | --- | --- |
| **Market** | One question about one token, with a deadline and an outcome | One Solana account |
| **Share** | A claim on $1.00 if its outcome wins. YES or NO | A number in a seat |
| **Pair** | 1 YES + 1 NO. Always worth exactly $1.00. The unit of collateral | A counter in the market header |
| **Seat** | A trader's row in a market: cash free/locked, YES free/locked, NO free/locked | A block in the market account |
| **Order** | An intent to buy or sell at a price, with escrow behind it | A block in the market account |
| **Book** | Bids and asks trees, quoted in YES prices | Blocks in the market account |
| **Vault** | The market's USDC token account, PDA-owned | An SPL token account |
| **Reader** | Read-only parser for one launchpad's token account | Program code |
| **Keeper** | Anyone running the permissionless lifecycle instructions | Off-chain |

### Units, fixed

| Quantity | Unit | Type | Note |
| --- | --- | --- | --- |
| USDC | base units, 6 dp | u64 | $1.00 = 1,000,000 |
| Shares | micro-shares, 6 dp | u64 | 1 winning micro-share pays exactly 1 USDC base unit: redemption has zero rounding |
| Price | mills | u16, range 1..999 | $0.57 = 570. **Structurally cannot exceed $0.999** |
| Progress | basis points | u16, 0..10000 | Read from the launchpad, never computed by us |

`cost = price × size / 1000`, computed in u128, floored. Tick: $0.01 normally, $0.001 permitted when the price is below $0.05 or above $0.95. Minimum order: $1.00 notional.

---

## 7. How the market works: the mechanism, in full

This section is the conceptual core. A developer who does not internalize it will build the wrong thing.

### 7.1 One book, four intents

Each market has **one physical order book, quoted in YES prices**. The UI renders two views (a YES book and a NO book) that are mirrors of it: every bid at price p on the YES view is an ask at 1 − p on the NO view. Four user intents map onto that one book:

| Intent | Escrow taken | Lands as |
| --- | --- | --- |
| BUY_YES at p | `p × size / 1000` USDC + max fee | **Bid** at p |
| SELL_YES at p | `size` YES shares | **Ask** at p |
| BUY_NO at q | `q × size / 1000` USDC + max fee | **Ask** at (1000 − q) |
| SELL_NO at q | `size` NO shares | **Bid** at (1000 − q) |

**Critical consequence: an ask can be created by someone holding no shares at all**, by buying NO. This is why a market with zero shares in existence still has a two-sided book, and why no designated market maker is required. See `WALKTHROUGH.md` section 5.

### 7.2 Four settlement kinds

When a bid and an ask cross, what physically happens depends on the two intents behind them:

| Bid intent | Ask intent | Kind | Effect | Pairs |
| --- | --- | --- | --- | --- |
| BUY_YES | SELL_YES | **TRANSFER_YES** | Buyer's cash to seller; seller's YES to buyer | unchanged |
| BUY_YES | BUY_NO | **MINT** | Both buyers' cash (p + (1000−p) = $1.00/share) locks as collateral; new pair created; YES to one, NO to the other | **+size** |
| SELL_NO | SELL_YES | **MERGE** | The two sellers' shares form pairs and are destroyed; each pair's $1.00 leaves the vault and splits p / (1000−p) | **−size** |
| SELL_NO | BUY_NO | **TRANSFER_NO** | Buyer's cash to seller; seller's NO to buyer | unchanged |

Exact per-kind ledger mutations: `03-ORDER-BOOK-SPEC.md` section 2.

**MINT is why a market needs no seller to begin.** **MERGE is why anyone can always exit.** Together they make share supply elastic, which is what bounds prices inside $0 to $1 (section 7.3).

### 7.3 Why price is bounded, and why that matters to the code

A share pays at most $1.00 and supply is unlimited at $1.00 per pair (anyone may `split`). If YES were bid near $1.00, anyone would split and sell into it risk-free, so the bid cannot hold there. Symmetrically, merging floors it at $0. Therefore:

- **YES price + NO price = $1.00** by arbitrage.
- **Every price is in (0, 1)**, enforced by the u16 mills type at 1..999.
- There is no "moon" case to handle: no code path anywhere needs to price a share above $1.00, and any arithmetic that could produce one is a bug the tests must catch.

### 7.4 Maker, taker, priority

- **Maker**: an order resting on the book when a trade occurs. **Taker**: the incoming order that crosses.
- **Price-time priority**, implemented by the order key `(price_mills << 64) | seq`, with `seq` bit-inverted for bids so numeric ordering equals price-time ordering on both trees.
- Trades execute **at the maker's price**; the taker keeps any price improvement and unused escrow is released.
- **Self-trade prevention**: an incoming order skips resting orders owned by the same seat.

### 7.5 Custody model

USDC lives in **one vault token account per market**, authority a program PDA. User balances and shares are ledger rows in the market account. Money enters by the user's own signed transfer and leaves only by `withdraw` (owner-signed) or `sweep_positions` (which can only pay a seat's owner). Trading performs **zero token transfers**: it is arithmetic between rows inside one account. Per-market vaults isolate blast radius.

---

## 8. Functional requirements

Numbered, testable. Every requirement maps to acceptance tests in section 16.

### 8.1 Accounts and funds

- **FR-1** A user can deposit USDC into a market and receive an equal credit to their seat's free balance, atomically.
- **FR-2** A user can withdraw any amount up to their free balance at any time, in any market state, to any token account they own. Only the seat owner's signature authorizes this.
- **FR-3** No instruction shall move a seat's funds to any destination other than that seat's owner, except as the documented result of a trade, mint, merge, fee or redemption.
- **FR-4** A user may register a **session key** scoped to place and cancel orders only, with an expiry. It must be incapable of withdrawing, transferring shares, or registering further keys.
- **FR-5** The vault's token balance must always equal `Σ free + Σ locked + pairs × $1.00 + accrued fees`.
- **FR-5a** **Zero protocol capital.** No instruction exists by which the protocol, its admin, its treasury or any privileged account deposits funds into a market, places an order, holds a share position, or acts as counterparty to a trade. A market is created with an empty vault and its first funds come from a trader. This is testable and must be asserted: across every test sequence, the set of share holders and order owners must contain no protocol-controlled address, and market creation must transfer zero collateral.
- **FR-5b** **No protocol liability.** Every payout is funded exclusively by collateral already locked by the losing side. There is no insurance fund, no backstop, no treasury guarantee and no circumstance in which the protocol owes a user money it does not already hold on that user's behalf.

### 8.2 Market creation

- **FR-6** Anyone may create a market for a token that has reached a listing milestone; the program independently verifies the milestone from the launchpad account rather than trusting the caller.
- **FR-7** Market creation records: the question type and parameters, the token and its launchpad account, the reader id, the creator address (barred from trading), the auction/close/deadline times, the size limit, and the rent payer.
- **FR-8** The size limit is `α × (amount the token still needs)`, α = 10%, converted to USDC using a price floor (feed price minus confidence, rejected if stale beyond 30 slots).
- **FR-9** Duplicate markets for the same (token, question, deadline) are impossible by PDA derivation.

### 8.3 Orders and matching

- **FR-10** A user may place LIMIT, IOC, FOK or POST_ONLY orders in any of the four intents, at a valid tick, at or above the minimum notional.
- **FR-11** Placing an order escrows its worst case: cash cost plus maximum fee for buys, shares for sells. Escrow is taken before any matching occurs.
- **FR-12** Matching follows price-time priority, executes at the maker's price, supports partial fills, and skips the taker's own resting orders.
- **FR-13** Every fill settles atomically: balances move, the maker is credited, the fee is split, events are emitted, all in the same instruction.
- **FR-14** A MINT fill must not cause `pairs × $1.00` to exceed the size limit recomputed from the launchpad's live state.
- **FR-15** An order may carry a **progress guard** (valid-range) and an **expiry slot**; violating orders are voided and refunded when encountered during matching, or by `prune_expired`.
- **FR-16** Every trading instruction must read the launchpad account passed to it and, if the outcome has already occurred, halt the market instead of trading.
- **FR-17** The market creator (the token's creator address) must be rejected from trading that token's open markets.
- **FR-18** A user may cancel any of their resting orders individually or all at once, with escrow released in the same instruction.

### 8.4 Share supply

- **FR-19** `split(n)` locks `n × $1.00` and credits n YES and n NO to the caller, subject to the size limit.
- **FR-20** `merge(n)` burns n YES and n NO and releases `n × $1.00`, and is permitted in **every** market state including after halt and after resolution.
- **FR-21** `transfer_shares` moves free shares between seats. Creator-locked NO shares in creator-written markets are non-transferable.

### 8.5 Lifecycle

- **FR-22** A market opens in **Auction**: orders collect and escrow, nothing matches, cancels allowed.
- **FR-23** `uncross` computes the single clearing price maximizing matched volume (ties: smaller imbalance, then lower price), fills all crossing orders at that one price, and transitions to **Continuous**. Submitting the same order set in any permutation must produce identical results.
- **FR-24** The market **halts** on the first of: the outcome observed on-chain, or the close time (deadline minus 5 minutes).
- **FR-25** At halt, every resting order's escrow is released to its owner's free balance. The operation must be resumable across transactions for arbitrarily deep books.
- **FR-26** `resolve` sets the outcome exactly once, from the launchpad account (or creator token account, or destination pool, per question family), or VOID if the reader cannot parse.
- **FR-27** Void causes every share on both sides to redeem for $0.50.

### 8.6 Payout

- **FR-28** `redeem` pays $1.00 per winning share (or $0.50 per share on void) into the owner's free balance, with no deadline, forever.
- **FR-29** `sweep_positions` is permissionless and pushes each seat's redeemed value and free balance to its owner's token account, creating it if needed.
- **FR-30** `close_market` is permitted only when all seats are settled and fees swept, and must refund all rent to the recorded payers.

### 8.7 Fees

- **FR-31** Both sides pay at rates set by their own trailing 30-day volume tier. Takers pay `taker_rate(tier)` of their own fill value, from 2.00% (tier 0) to 1.20% (tier 5). Makers at tiers 0 to 2 pay `maker_rate(tier)` of their own fill value (1.00%, 0.60%, 0.25%); tier 3 is free; tiers 4 and 5 receive a rebate instead.
- **FR-32** A tier-4 or tier-5 maker is credited 15% or 30% of the **taker fee on that fill**, to their free balance in the same instruction. The rebate is never a percentage of the maker's own notional: on a mint the maker's notional can exceed the taker's many times over, so a notional-based rebate could exceed the fee collected. The rebate share is bounded in-program at 30%, so `taker_fee + maker_fee − maker_rebate > 0` at every tier combination and the fee system can never run at a loss. A maker is never both charged and paid on the same fill. Rebate eligibility is exactly: the order was resting on the book, an incoming taker matched it, and the fill charged a fee. Roles are computed **per fill**, so one order may pay a taker fee on the portion that fills on arrival and earn rebates on the remainder once it rests. `POST_ONLY` guarantees maker status by rejecting any order that would fill immediately. No account is ever registered as a maker or market maker; the program has no such concept.
- **FR-33** Auction fills charge each filled participant half their own taker rate (1.00% at tier 0), with no rebate to either side, since nobody was resting.
- **FR-34** Deposits, withdrawals, cancels, splits, merges, transfers and redemptions charge no protocol fee.
- **FR-33a** A trader's fee tier is read from their global `TraderStats` account and stamped into their seat at seat creation, inside a transaction the trader signs. Fills read the tier from the seat only, so tiering must add **zero accounts and zero CPIs** to the settlement path. A tier is fixed for the life of one market.
- **FR-33b** Each seat accumulates its filled notional, and that volume is rolled into the owner's `TraderStats` when the market is swept or closed. The 30-day window uses daily buckets rotated lazily on write. A missing `TraderStats` account means tier 0.
- **FR-33c** Each side of a fill is charged on its **own** tier, independently of the other side's tier.
- **FR-35** Keeper fees are paid from the market's accrued fees per the Config schedule.

### 8.8 Off-chain

- **FR-36** Keepers for create, uncross, halt, resolve, prune, sweep and close are permissionless, idempotent at the state-machine level, and fee-bearing.
- **FR-37** The indexer serves the launch feed, markets, books, trades, candles, positions and leaderboards, reconstructible entirely from chain state.
- **FR-38** REST and WebSocket APIs expose all read data; no write endpoint exists except the fee sponsor, which can only co-sign whitelisted instructions.
- **FR-39** The frontend must never hold a key other than the session key it generated, and must display no number that cannot be derived from chain state.

---

## 9. System architecture

```
   Launchpads (external, read-only)
        |
        | public account reads
        v
  +-----------------------------+        +---------------------------+
  |   Burbit program (Solana)   |<-------|  Keepers (permissionless) |
  |   Config                    |        |  create/uncross/halt/     |
  |   Market accounts           |        |  resolve/prune/sweep/close|
  |     header + block pool     |        +---------------------------+
  |       bids tree             |
  |       asks tree             |        +---------------------------+
  |       seats tree            |<-------|  Traders (wallet +        |
  |   Vault (USDC token acct)   |        |  session key)             |
  |   Readers (parsers)         |        +---------------------------+
  +-----------------------------+
        |  events (self-CPI log)
        v
  +-------------+     +------------------+     +-----------+
  |  Indexer    |---->|  REST + WS API   |---->|  Web app  |
  +-------------+     +------------------+     +-----------+
                              ^
                      +---------------+
                      | Fee sponsor   |  (co-signs; spends only its own SOL)
                      +---------------+
```

Components and their trust: only the program is trusted with funds. Keepers affect liveness, never safety. The indexer and API serve convenience data. The fee sponsor can lose only its own SOL.

---

## 10. On-chain data model and rent

### 10.1 Accounts

| Account | Seeds | Purpose |
| --- | --- | --- |
| Config | `["config"]` | Global parameters, reader registry, treasury, admin, pause |
| Market | `["market", source_account, question_tag, deadline]` | Header + block pool (book + seats) |
| Vault | `["vault", market]` | The market's USDC token account |
| Vault authority | `["vault-auth", market]` | PDA that signs transfers out |
| SessionKeys | `["session", owner]` | Scoped trading keys |
| Bond | `["bond", mint, creator]` | Creator-written market custody |
| Proposal | `["proposal", market]` | Bonded proposals for aggregate questions |

Full field-level layouts: `06-ONCHAIN-PROGRAM-SPEC.md`.

### 10.2 The block pool

The market account is a fixed header followed by a pool of **112-byte blocks**. Three red-black trees and one free list interleave over the same pool: **bids**, **asks**, **seats**, **free**. A block is exactly one of:

```
OrderBlock:  key(price|seq) | tree links | owner_seat | remaining |
             escrow_remaining | intent | order_type | flags |
             guard_min | guard_max | expiry_slot | client_id | rent_payer

SeatBlock:   owner | tree links | usdc_free | usdc_locked |
             yes_free | yes_locked | no_free | no_locked |
             open_orders | flags
```

The account **reallocates one block at a time** as it needs capacity; freed blocks are reused before growing. Best-bid and best-ask indexes are cached in the header for O(1) reads.

### 10.3 Rent economics (build this in from day one)

Rent exemption is a **refundable deposit**. The design's rule: every account closes, every deposit returns.

| Account | Size | Deposit | Payer | Refund |
| --- | --- | --- | --- | --- |
| Market, initial | ~7.5 KB | ~0.053 SOL | creation keeper | at close |
| Vault | 165 B | ~0.002 SOL | creation keeper | at close |
| Each added block | 112 B | ~0.00078 SOL | the transaction that needed it | at close, to that payer |
| SessionKeys | ~200 B | ~0.0023 SOL | user, once | on revoke-all |

Requirement: **every block records its rent payer**, and the market header maintains `unsettled_seats` and block accounting so `close_market` can refund correctly. A market that never trades must still close and refund.

**Why not SPL mints for shares** (the analysis, so nobody re-litigates it mid-build): two mints per market is 0.0029 SOL, plus 0.00204 SOL per trader per side held, scattered across user wallets where it is almost never reclaimed; ledger seats cost 0.00078 SOL per trader for both sides plus their cash, inside one account that closes automatically. At thousands of short-lived markets per day the SPL design leaks hundreds of SOL into abandoned accounts and adds a token-program CPI to every fill. Ledger balances are the design. A tokenize-on-demand instruction can be added later without changing anything else.

---

## 11. Instruction reference

Full signatures, accounts, checks and effects: `06-ONCHAIN-PROGRAM-SPEC.md` section 2. Summary with the requirement each satisfies:

| Instruction | Signer | Satisfies |
| --- | --- | --- |
| `initialize_config`, `propose_config_update`, `apply_config_update`, `pause`, `unpause` | admin | Governance, timelocked |
| `create_market` | anyone | FR-6..9 |
| `deposit`, `withdraw` | owner | FR-1, FR-2 |
| `register_session_key`, `revoke_session_key` | owner | FR-4 |
| `place_order` | owner or session key | FR-10..17 |
| `cancel_order`, `cancel_all` | owner or session key | FR-18 |
| `prune_expired` | anyone | FR-15 |
| `split`, `merge`, `transfer_shares` | owner | FR-19..21 |
| `uncross` | anyone | FR-23 |
| `halt`, `halt_continue` | anyone | FR-24, FR-25 |
| `record_witness` | anyone | Threshold-style questions (`15`) |
| `resolve` | anyone | FR-26, FR-27 |
| `redeem` | owner | FR-28 |
| `sweep_positions`, `sweep_fees` | anyone / treasury | FR-29 |
| `close_market` | anyone | FR-30 |
| `create_bond`, `deposit_custody`, `write_rug_market`, `withdraw_custody` | creator | Creator-written markets (`05` §7) |
| `propose_outcome`, `challenge_outcome`, `finalize_outcome` | anyone, bonded | Aggregate questions (`05` §8) |

**Universal precondition**: any instruction that can trade, halt or resolve takes the launchpad state account, validates owner and address, parses with the registered reader, and on parse failure enters the void path rather than erroring in a way that leaves the market live.

---

## 12. The matching engine

Inside `place_order`, after state checks, launchpad read, creator check, tick and notional validation, and escrow:

```
side      = BID if intent in {BUY_YES, SELL_NO} else ASK
yes_price = price if intent in {BUY_YES, SELL_YES} else 1000 - price
remaining = size

loop while remaining > 0:
    M = best resting order on the opposite tree
        skip + remove: expired orders          -> void, release escrow
        skip + remove: guard-violated orders   -> void, release escrow
        skip only:     orders owned by caller  (self-trade prevention)
    if M is None: break
    if side == BID and M.yes_price > yes_price: break
    if side == ASK and M.yes_price < yes_price: break

    kind = derive(bid_intent, ask_intent)          # 4 kinds, section 7.2
    q    = min(remaining, M.remaining)
    if kind == MINT:
        q = min(q, remaining_capacity_under_size_limit())
        if q == 0: break

    settle(kind, M.yes_price, q)                   # maker credited in place
    fee    = 2% of taker notional
    rebate = 20% of fee -> M.owner.usdc_free
    accrue  80% of fee -> market.fees_accrued
    emit Fill
    remaining -= q
    if M.remaining == 0: remove M, emit OrderDone

if remaining > 0:
    LIMIT     -> rest remainder (escrow already held), emit OrderPosted
    IOC       -> release remainder escrow
    FOK       -> whole instruction reverts unless fully filled
    POST_ONLY -> revert if any fill occurred
```

Depth is bounded by the compute budget, not a fixed fill cap; each fill is a handful of in-account writes with no CPIs. On approaching the budget the instruction posts or releases the remainder and succeeds rather than reverting.

The **uncross** algorithm (auction) is specified in `04-MARKET-LIFECYCLE.md` section 3 and must be permutation-invariant.

---

## 13. Money flows, worked

A complete market, to the cent. Question: "Will $WOOF graduate within 15 minutes?" Size limit at creation: $646.

**Auction, 60 seconds.** Orders collected:

| Trader | Intent | Price | Size | On the YES book |
| --- | --- | --- | --- | --- |
| Alice | BUY_YES | 7¢ | 150 | Bid 70 |
| Carol | BUY_YES | 5¢ | 100 | Bid 50 |
| Bob | BUY_NO | 94¢ | 50 | Ask 60 |
| Dan | BUY_NO | 96¢ | 100 | Ask 40 |

**Uncross.** Max matched volume is 150 at both 60 and 70 mills; tie goes to the lower price: **clearing price 6¢**.

- Alice's 150 fills against Dan (100, cheapest ask first) then Bob (50). All **MINTs** at 6¢.
- Alice pays 150 × 0.06 = **$9.00** + her 1% auction fee **$0.09**.
- Dan pays 100 × 0.94 = **$94.00** + **$0.94**. Bob pays 50 × 0.94 = **$47.00** + **$0.47**. Auction fees total **$1.50**, exactly 1% of the $150 minted.
- **150 pairs exist. Vault holds $150.00 collateral** (under the $646 limit) + $0.18 fees.
- Carol's bid at 5¢ did not cross; it rests.

**Continuous.** Alice posts sell 100 YES at 20¢ (maker). Eve takes it (**TRANSFER_YES**).

- Eve pays $20.00 + taker fee 2.00% = $0.40, total **$20.40**.
- Alice, a tier-0 maker, receives $20.00 less her own maker fee 1.00% = $0.20, netting **$19.80**. Treasury accrues $0.60.

**Graduation at minute 9.** The next instruction halts the market. Carol's $5.00 escrow returns in full.

**Resolve YES. Redeem.**

| Trader | Paid | Received | Net |
| --- | --- | --- | --- |
| Eve | 20.40 | 100.00 (100 YES) | **+79.60** |
| Alice | 9.09 | 19.80 net of maker fee + 50.00 (50 YES) | **+60.71** |
| Dan | 94.94 | 0 | **−94.94** |
| Bob | 47.47 | 0 | **−47.47** |
| Carol | 0 | 0 (escrow returned) | **0** |
| Treasury | n/a | 1.50 auction + 0.40 taker + 0.20 maker | **+2.10** |

Check: 79.60 + 60.71 + 2.10 = 142.41 = 94.94 + 47.47. **Zero-sum to the cent.** The vault ends holding exactly the accrued fees; `sweep_fees` then `close_market` empties it and refunds all rent.

---

## 14. Test environment and test USDC

### 14.1 Test collateral mint

The program reads its collateral mint from `Config.usdc_mint` and **must never hardcode it**. For devnet and local testing we mint our own SPL token and use it as USDC.

```bash
# 1. Create the test mint: 6 decimals, matching real USDC
spl-token create-token --decimals 6 --url devnet
#    -> save the mint address, e.g. TestUSDC1111111111111111111111111111111111

# 2. Create a treasury account and mint a supply for the faucet
spl-token create-account <MINT> --url devnet
spl-token mint <MINT> 10000000 --url devnet        # 10,000,000 test USDC

# 3. Point the program at it
#    Config.usdc_mint = <MINT>   (set at initialize_config)
```

Requirements:

- **TR-1** The mint must have **6 decimals** so all amount arithmetic matches mainnet exactly.
- **TR-2** Mint authority is held by a dev keypair for the faucet; it is never referenced by the program.
- **TR-3** The app's faucet button requests test USDC from a faucet service for the connected wallet, devnet only, rate-limited.
- **TR-4** Promotion to mainnet is a Config value change plus redeployment of the frontend's cluster setting. **No program code changes.**
- **TR-5** The test suite must run against a locally created mint, never a hardcoded address.

### 14.2 Test launchpad state

Graduation markets need launchpad accounts to read. Two modes:

- **Local/unit**: a mock launchpad program that exposes the same account shape, letting tests drive progress and completion deterministically (including layout-corruption cases for the void path).
- **Devnet integration**: mirror real mainnet token account states onto devnet accounts and replay them over time, so keepers, readers and the app run against realistic sequences.

### 14.3 Environments

| Environment | Cluster | Collateral | Launchpad state | Purpose |
| --- | --- | --- | --- | --- |
| Local | localnet | locally minted | mock program | Unit + differential tests in CI |
| Devnet | devnet | test USDC mint | mirrored/replayed | Full-stack integration, demos |
| Mainnet-beta | mainnet | real USDC | real launchpads | Production, after audits |

---

## 15. Non-functional requirements

| Area | Requirement |
| --- | --- |
| **Compute** | `place_order` no-fill ≤ 25k CU; ≤ 6k CU marginal per fill; launchpad parse + price read ≤ 10k CU combined |
| **Latency** | Order to confirmed fill within one to two slots. Indexer no more than one slot behind |
| **Cost to user** | Trading transaction fees sponsored; user out-of-pocket is their seat rent only, refunded |
| **Throughput** | One writable account per market bounds per-market throughput; markets shard naturally |
| **Availability** | Program availability equals the cluster's. All user exits (cancel, withdraw, merge, redeem) must work with zero off-chain services running |
| **Security** | Checked math everywhere, escrow-first, PDA re-derivation on every account, no arbitrary CPI, reentrancy-guarded fill paths, full vault reconciliation at sweep and close. Two independent audits before mainnet funds. Admin timelock 24h, no pre-signable admin paths |
| **Observability** | Structured events for every state change via self-CPI log; keeper metrics for lag, landing rate and fees earned |
| **Data integrity** | Every number in the app derivable from chain state; indexer reconciles against account snapshots and logs divergences as defects |

---

## 16. Acceptance criteria and test plan

### 16.1 The reference implementation is the spec

A Python reference implements the entire exchange (seats, escrow, four intents, four settlement kinds, priority, ticks, fees, rebates, guards, expiries, size limits, auction, halt, resolve, void, redeem, sweep, close). Every worked example in these documents is a test case in it.

### 16.2 Differential testing (the primary gate)

Randomized instruction sequences run through **both** the reference and the compiled program in an in-process Solana VM. After **every instruction**, all balances must match exactly and all invariants must hold.

Generator must cover: all intents and order types; auction and continuous phases; deposits, withdrawals, splits, merges, transfers; cancels; keeper calls interleaved at random legal points; launchpad state changes driving guards, size limits and completion at every moment (including mid-auction and between escrow and rest); price-feed drift and staleness; multi-outcome sets; bond lifecycles; sequences ending in resolve or void, full redemption, sweep and close.

Minimum: **1,000 sequences × 300 operations per commit**, 100k-sequence soak nightly.

### 16.3 Invariants asserted after every step

1. `vault = Σ usdc_free + Σ usdc_locked + pairs × $1.00 + fees_accrued`
2. Pre-resolution: `Σ yes = Σ no = pairs`
3. Size limit respected at every mint and split against live state
4. No negative balance anywhere, ever
5. Every resting order's escrow reconstructs exactly from its terms
6. Tree validity: red-black properties, key ordering, cached best indexes correct, free list disjoint and complete, no dangling seat references
7. Post-close: vault empty, rent refunds sum exactly to rent paid
8. Zero-sum: Σ user nets + treasury = 0 across any completed market
9. **No price above 999 mills or below 1 mill exists in any structure at any time**

### 16.4 Named adversarial scenarios (must all pass)

- **Forced outcome**: attacker pays to finish the token; assert maximum extractable win stays below what they spent, at every reachable book state at the size limit.
- **Stale-quote sniper**: launchpad state jumps past a maker's guard in the same slot as a taker's order; assert the maker voids, never fills.
- **Completion race**: the outcome lands between submission and execution; assert halt, full escrow release, no fill.
- **Size-limit exhaustion under pressure**: concurrent mint pressure at the limit; assert clipping, never exceeding.
- **Auction permutation**: identical order sets in every submission order produce identical uncross results.
- **Self-trade laundering**: a wallet on both sides cannot extract rebates or move price against itself.
- **Dust and rounding**: 1-unit orders, 1-mill prices, maximal fills; assert conservation to zero units created or destroyed.
- **Halt-depth grief**: a maximally deep book halts across multiple `halt_continue` calls with exact total release.
- **Reader corruption**: mutated launchpad account bytes mid-market; assert void path and $0.50 redemptions conserve the vault exactly.
- **Abandoned market**: created, never traded, no keeper for an hour; assert it still halts, resolves, and closes with full rent refund.

### 16.5 Definition of done for v1

- [ ] All FR-1 to FR-39 implemented and covered by tests
- [ ] Differential soak green at 100k sequences
- [ ] All nine invariants asserted continuously; all ten adversarial scenarios passing
- [ ] 100% branch coverage on fund-path modules
- [ ] CU budgets in section 15 met under a replay benchmark
- [ ] Full market lifecycle reproducible end to end on devnet in under 15 minutes with rent conserved to zero
- [ ] Two independent audits complete with zero unresolved criticals
- [ ] App flows complete: connect, session key, fund, quick bet, limit order, cancel, cash out, claim, withdraw

---

## 17. Build sequence

| Phase | Deliverable | Exit gate |
| --- | --- | --- |
| **0. Foundations** | Reference implementation extended to this spec; test USDC mint and mock launchpad program; CI skeleton | Reference reproduces every worked example in these docs |
| **1. Core program** | Block pool and trees as a standalone library with property tests; Config, Market, Vault, seats, deposit/withdraw | Trees pass property tests; funds move and reconcile |
| **2. Trading** | place_order with all four intents and kinds, fees, rebates, guards, expiries, size limits, cancel, split/merge/transfer, events | Differential harness green on trading-only sequences |
| **3. Lifecycle** | Auction and uncross, halt and halt_continue, record_witness, resolve, redeem, sweep, close, void path; first launchpad reader; price floor | Full lifecycle green in differential tests including adversarial set |
| **4. Off-chain** | Keeper binary (all roles), indexer, REST + WS API, SDK | Keepers sustain markets unattended on devnet |
| **5. App** | Launch feed, market screen, quick bet, set odds, positions, claim, session keys, sponsor service, faucet | All app flows complete on devnet |
| **6. Hardening** | Audits, fuzzing, soak, bug bounty, admin timelock and key ceremony | Definition of done met |

---

## 18. Risks and open decisions

| Risk | Mitigation |
| --- | --- |
| **Liquidity**: long-tail markets may only trade in the auction | List only at high-attention milestones; opening auction; maker rebates; open-source reference quoter; unfilled orders cost nothing and are refunded |
| **Insider stalling**: large holders can block some graduations and profit on NO | Short windows, size limits, creator barred, disclosed in the app and priced into published base rates |
| **Launchpad changes**: layouts and rules change | Strict readers, fail closed to a refunding void, fuzzed against mutated bytes, one reader per venue |
| **Program bug** | Per-market vault isolation bounds blast radius; differential testing; two audits; pause; void-and-refund as the designed unwind |
| **Admin key compromise** | No fund-moving admin instruction exists; 24h timelock; hardware-isolated multisig; no pre-signed transactions; monitoring on authority changes |
| **Regulatory** | Access policy decided before mainnet; geofencing at the app layer; the program itself remains permissionless |

### Open decisions

| # | Decision | Recommendation |
| --- | --- | --- |
| OD-1 | Session key default lifetime | 24 hours, max 7 days, user-chosen |
| OD-2 | Fee sponsor per-key velocity cap | Set so a compromised session key cannot burn more than a few dollars of SOL |
| OD-3 | Whether to ship `tokenize_position` in v1 | No. Ledger shares only; revisit if composability demand appears |
| OD-4 | Initial α (size limit fraction) | 10%, re-derived once the collector has live data |
| OD-5 | Minimum order size | $1.00; revisit after observing real ticket sizes |
