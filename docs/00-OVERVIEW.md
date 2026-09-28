# Burbit: Overview

**An order-book prediction market for bonding-curve tokens on Solana.**

*Build documentation set, version 1.0*

---

## 1. What Burbit is

Burbit is a prediction market for tokens that are still trading on bonding curves. From the moment a token launches on a supported launchpad until it graduates or dies, Burbit runs markets on the questions that lifecycle creates:

- Will this token graduate before 18:00?
- Will it graduate within 10 minutes?
- Which of these three launches graduates first?
- Will the creator dump their bag before the hour is out?

Users trade **YES** and **NO** shares on these questions through a **real central limit order book (CLOB)**. Every share pair is backed one-to-one by USDC locked in the market's vault. A winning share redeems for exactly $1.00. A losing share redeems for nothing.

**Burbit builds no bonding curve, no AMM, no pool and no launchpad, and never touches any of them.** Tokens launch, trade and graduate on existing launchpads; Burbit's entire relationship to that machinery is **reading its public on-chain state**. The curve exists in this documentation only because settling and pricing predictions requires understanding exactly how it behaves. Burbit reads each launchpad's public accounts, attaches yes/no conditions to each launch, validates every trade against the token's live curve state, and settles each market directly from that state. There is no data provider, no oracle committee, and no human resolver for the core market families. The answer is read from the chain in the same instruction that pays it out.

Because integration is read-only, **Burbit can integrate any bonding-curve launchpad**: supporting a new one means writing one reader (a strict parser for that launchpad's public account layout, plus its progress and forcing-cost math) and registering it. No permission, partnership or contract call into the launchpad is ever needed, and nothing else in the protocol changes per venue.

## 2. What makes Burbit different

1. **A real order book, not a pool and not a synthetic derivative.** Users set their own odds with limit orders or take the best available odds instantly. Makers and takers meet at real prices with price-time priority. Positions are fully-paid assets, not margined contracts: there is no leverage, no funding, no liquidation engine, and no house-backed liquidity pool acting as counterparty. Every share held by anyone is matched by the opposite share held by someone else.

2. **Shares are created and destroyed by traders themselves.** When a YES buyer and a NO buyer agree on a price, the program locks their combined $1.00 and mints a new share pair between them. When a YES seller and a NO seller meet, their pair dissolves back into $1.00. No market maker is required to start a market, and anyone can always exit.

3. **Settlement is trustless and instant.** The program parses the token's bonding-curve account on its launchpad inside the settlement instruction. The moment the curve's completion flag is true, the market halts and YES is final. No proposal, no bond, no challenge window, no vote, no waiting.

4. **Markets cannot be profitably gamed.** Each market's open interest is capped below the cost of forcing the outcome, and that cost is computed on-chain from the token's live curve state at every mint.

5. **Makers are protected by curve guards.** Every order can carry the range of curve progress it is valid for. If the token's curve jumps outside that range, the order voids itself before anyone can hit it.

6. **Fully on-chain and crankless.** Matching, settlement of fills, and maker crediting all happen atomically inside the same transaction. There is no event queue to consume, no settlement crank, no operator whose engine must stay honest and online. The order book and every trader's balances live inside the market account, so a taker's transaction already has write access to every maker it fills.

7. **Burbit never risks its own capital.** Every share is funded by users. Burbit earns a flat 2% taker fee, of which 20% is rebated to the maker on the other side.

## 3. The system at a glance

```
Launchpads (existing, public on-chain accounts)
      |            read-only
      v
+-----------------------------+       +---------------------+
| Burbit program (Solana)     | <---- | Keepers             |
|  - Config                   |       |  create, uncross,   |
|  - Market accounts          |       |  halt, resolve,     |
|    (book + seats + state)   |       |  sweep, close       |
|  - USDC vault per market    |       +---------------------+
|  - Readers (curve parsing)  |
+-----------------------------+
      ^            ^
      | txs        | account subscriptions
      |            v
+-----------+   +---------------------------+
| Traders   |   | Indexer                   |
| (wallets, |   |  launch feed, books,      |
|  session  |   |  trades, positions        |
|  keys)    |   +---------------------------+
+-----------+          |
                       v
              +------------------+
              | REST + WebSocket |
              | API              |
              +------------------+
                       |
                       v
              +------------------+
              | Web app          |
              +------------------+
```

- **The program** holds every market: its order book, every trader's balances, the USDC vault, the state machine, and the settlement logic. It is the only component that touches funds.
- **Keepers** are permissionless bots that advance market lifecycles (create at milestones, uncross auctions, halt on graduation, resolve, sweep payouts, close). Each call pays a small on-chain fee, so anyone can run them profitably.
- **The indexer** streams launchpad accounts and Burbit accounts, and serves the launch feed, books, trades and positions to the app over REST and WebSocket. It holds no funds and no authority.
- **The app** is a thin client over the API and the program. One-tap trading is delivered through session keys and sponsored transaction fees, so betting feels instant and free while remaining fully non-custodial.

## 4. Chain, collateral, units

| Parameter | Value |
| --- | --- |
| Chain | Solana |
| Collateral | USDC (6 decimals) |
| Share payout | $1.00 per winning share (1,000,000 base units) |
| Price range | $0.001 to $0.999, quoted in cents in the app |
| Price = probability | A YES at $0.57 means the market prices the event at 57% |
| Standard tick | $0.01, refining to $0.001 when the best price is below $0.05 or above $0.95 |
| Minimum order | $1.00 notional |
| Taker fee | 2% of the taker's USDC notional |
| Maker rebate | 20% of the taker fee |
| Maker fee | none |

## 5. Glossary

| Term | Meaning |
| --- | --- |
| **Market** | One yes/no question about one token, with a deadline |
| **Share** | A claim on $1.00 if its outcome wins; YES and NO per market |
| **Pair** | 1 YES + 1 NO; always worth exactly $1.00; the unit of open interest |
| **Mint** | Creating a pair by locking $1.00 (two buyers on opposite sides) |
| **Merge** | Destroying a pair and releasing its $1.00 (two sellers on opposite sides) |
| **Seat** | A trader's ledger row inside a market account: USDC and share balances, free and locked |
| **Book** | The bids and asks trees inside the market account, quoted in YES prices |
| **Intent** | One of buy YES, sell YES, buy NO, sell NO |
| **Maker** | The order resting on the book when a trade happens |
| **Taker** | The incoming order that crosses and executes against resting orders |
| **Curve guard** | A progress range attached to an order; the order voids outside it |
| **Open interest** | Pairs outstanding × $1.00; equals the pair collateral in the vault |
| **Forcing cost** | What it would cost an attacker to force the market's outcome on the launchpad |
| **Cap** | The open-interest ceiling, α × forcing cost, recomputed from live curve state |
| **Reader** | Program code that parses one launchpad's curve-account layout, read-only |
| **Graduation** | The launchpad marking a token's curve complete and migrating it to an open pool |
| **Halt** | Trading frozen; all resting orders released back to free balances |
| **Void** | Market annulled; every share on both sides redeems for $0.50 |
| **Keeper** | Anyone running the permissionless bots that advance market lifecycles |
| **Session key** | A scoped hot key registered on-chain that may place and cancel orders but never withdraw |

## 6. The documentation set

| File | Contents |
| --- | --- |
| `WALKTHROUGH.md` | **Start here**: the plain-language, end-to-end product walkthrough: what is traded, where money sits, how the YES/NO books work, who creates the first orders, and what happens to everyone at resolution |
| `00-OVERVIEW.md` | This file |
| `01-ARCHITECTURE.md` | Full system architecture and how every component connects |
| `02-CORE-CONCEPTS.md` | Shares, pairs, collateral, pricing, the invariants that hold everything together |
| `03-ORDER-BOOK-SPEC.md` | The book, the four intents, the four settlement kinds, matching, order types, fees |
| `04-MARKET-LIFECYCLE.md` | Every market state and transition, the opening auction, halting, resolution, redemption, closing |
| `05-MARKETS-AND-SETTLEMENT.md` | The question families, settlement rules per family, caps and safety rules |
| `06-ONCHAIN-PROGRAM-SPEC.md` | Accounts, instructions, data layouts, errors, events, compute and rent |
| `07-KEEPERS.md` | Every keeper, its algorithm and its incentive |
| `08-INDEXER-AND-API-SPEC.md` | The indexer, the REST API and the WebSocket API |
| `09-FEES-AND-ECONOMICS.md` | Fee formulas, rebates, treasury, revenue model |
| `10-FRONTEND-SPEC.md` | Every screen, every flow, wallet and session-key UX, design language |
| `11-SECURITY.md` | Threat model, fairness rules, the admin plane, audit plan |
| `12-TESTING.md` | The reference implementation, differential testing, invariants, adversarial cases |
| `13-ROADMAP.md` | Build order, phases, and the v2 signed-order gateway |
| `14-BONDING-CURVE-BEHAVIOR.md` | The complete behavioral study of bonding-curve tokens: venue mechanics, lifecycle statistics, actors, signals, settlement-quality grades |
| `15-QUESTION-CATALOG.md` | Every question type Burbit can run: settlement rule, forcing analysis, offerability class, hypothesis, and the automated listing playbook |
