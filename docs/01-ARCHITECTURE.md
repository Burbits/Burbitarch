# Architecture

How every component of Burbit is built and how they connect. Read `02-CORE-CONCEPTS.md` for the economic model and `06-ONCHAIN-PROGRAM-SPEC.md` for exact data layouts.

---

## 1. Design principles

1. **The program is the exchange.** Matching, custody, settlement and resolution are all enforced by one on-chain program. There is no off-chain matching engine in v1, no operator with special powers over trades, and no component that can lose user funds by being wrong or offline.
2. **Crankless, atomic settlement.** The order book and every trader's balances live inside the market account. A transaction that fills orders already has write access to every maker it touches, so fills credit makers in the same instruction. There is no event queue, no consume-events crank, and no window in which a fill exists but its money has not moved.
3. **Everything off-chain is permissionless and replaceable.** Keepers advance lifecycles for on-chain fees anyone can earn. The indexer serves data anyone could re-derive from the chain. If Burbit-the-company disappears, markets still halt, resolve, pay out and close.
4. **Read the launchpad, never trust anyone about it, never touch it.** Burbit builds no curve, no AMM and no pool; its only relationship to any launchpad is parsing that launchpad's public accounts. Every instruction that depends on the token's state takes the token's launchpad launchpad state account as an input and parses it with a strict reader. Unparseable data fails closed into a void, never into a wrong settlement. Because the integration surface is a read-only reader, any bonding-curve launchpad can be supported by adding one reader to the registry, with no change to markets, matching, custody or settlement.
5. **The admin plane cannot touch funds.** Admin keys can pause new activity, register readers, and tune parameters behind a timelock. No admin instruction can move vault balances, mutate seats, or set outcomes.

## 2. Component map

### 2.1 On-chain

| Component | What it is | What it does |
| --- | --- | --- |
| **Burbit program** | One Solana program | Markets, order books, seats, vaults, matching, settlement, resolution, redemption |
| **Config account** | PDA, one global | Fee rates, cap parameter, auction length, close gap, keeper fees, reader registry, treasury, admin, pause flag, timelock state |
| **Market account** | PDA per market | Header (question, times, state, cap, pairs, fees) + a growable block pool holding the bids tree, the asks tree and the seat table |
| **Market vault** | SPL token account per market, program-authority PDA | Holds all USDC in the market: escrow, pair collateral, fees |
| **Session key registry** | PDA per user | Scoped trading keys with expiries |
| **Bond accounts** | PDA per creator-written rug market | Creator custody and bond state |
| **SOL/USD price account** | Pyth price feed (external, read-only) | Converts SOL-denominated amounts into the USDC-denominated market size limit; read with staleness and confidence checks |
| **Launchpad launchpad state accounts** | External, read-only | The ground truth every market is about |

### 2.2 Off-chain

| Component | Trust required | Paid by |
| --- | --- | --- |
| **Market-creator keeper** | None (permissionless) | Rent refund at close + creation fee from market fees |
| **Uncross / halt / resolve keepers** | None | Small per-call keeper fee from market fees |
| **Sweep keeper** | None | Per-seat sweep fee |
| **Close keeper** | None | Claims residual rent refunds routed per config |
| **Indexer** | None with funds; users trust it for convenience data only | Burbit (infrastructure) |
| **API + WebSocket service** | Same as indexer | Burbit |
| **Fee-payer service** | Pays network fees for sponsored user transactions; can spend only its own SOL | Burbit (covered by trading fees) |
| **Reference quoter** | Open source; runs on its operator's own funds | Spread + maker rebates |

## 3. Data flow, end to end

### 3.1 Listing a token

1. The indexer streams every supported launchpad program and records each new token: mint, launchpad state account address, creator, creation slot.
2. It tracks progress, reserves and creator balances in real time and publishes them in the launch feed.
3. When a token crosses a listing milestone (for example 70% of its curve sold), the market-creator keeper submits `create_market`. The program independently re-parses the launchpad state account, verifies the milestone, computes the open-interest cap, allocates the market account and vault, and opens the market in the Auction state.
4. Nothing about listing requires the launchpad's cooperation. Readers parse public account data that anyone can fetch.

### 3.2 Trading

1. The user's wallet connects to the app. On first trade the app registers a **session key**: an instruction signed once by the wallet that whitelists a browser-held keypair allowed to place and cancel orders (never withdraw) until an expiry the user chose.
2. The user taps a price. The app builds `place_order`, signs it with the session key, and sends it through the **fee-payer service**, which co-signs as the transaction fee payer. The user sees no wallet popup and pays no network fee. The trade lands within a slot (about 400 ms).
3. `place_order` runs entirely in-program: it parses the live launchpad state account (halting the market on the spot if the token has graduated), voids any resting order whose progress guard the current progress violates, locks the taker's escrow into their seat, matches against the book with price-time priority, settles each fill atomically (transfer, mint or merge; see `03-ORDER-BOOK-SPEC.md`), charges the taker fee, credits the maker rebate, and rests any remainder.
4. Fills are emitted as structured events through the program's own log instruction (a self-invoke carrying typed event data), which the indexer consumes to update books, trades and positions in real time.

### 3.3 Settlement and payout

1. The moment the token graduates on its launchpad, the next instruction that touches the market, or a keeper's explicit `halt`, freezes trading, records the completion slot, and releases every resting order's escrow back to seat free balances.
2. `resolve` reads the launchpad state account (or the creator's token account, for dump questions) and fixes the outcome. YES if the event happened before the deadline, NO if the deadline passed without it, VOID if the account no longer parses.
3. Winners call `redeem` for $1.00 per share, or the **sweep keeper** pushes every seat's payout and free balance back to each owner's USDC token account so nobody has to return for dust.
4. When every seat is settled and swept, `close_market` deallocates everything and returns all rent to the recorded payers.

## 4. Why this architecture and not the alternatives

Three architectures can host a CLOB on Solana. Burbit v1 uses the first; the second is a planned v2 extension; the third is rejected.

| | v1: fully on-chain, crankless | v2 extension: signed-order gateway | Rejected: external matching operator as the only path |
| --- | --- | --- | --- |
| Order placement | A transaction (~400 ms, fee sponsored) | A free signed message relayed to the chain by anyone | A free signed message accepted only by the operator |
| Matching | In-program, price-time enforced by code | In-program at settlement; the relayer only chooses which valid crossings to submit | Off-chain engine; program can only verify, not order |
| Liveness | The chain itself | Degrades to v1 path if relayers censor | Dead if the operator is down |
| Trust | Program only | Program only | Operator sequencing and uptime |
| Infrastructure burden | Indexer + small keepers | Adds a relay service | A 24/7 engine with correctness obligations before the first user |

The deciding facts:

- Burbit's markets live minutes, with bursty flow that the opening auction absorbs. They do not need microsecond matching; they need trustless settlement at the exact moment a curve completes, which the on-chain path checks atomically inside every fill.
- The user-experience gap closes with session keys plus sponsored fees: one tap, no popup, no visible cost, sub-second confirmation.
- Makers are protected primarily by progress guards evaluated at match time, which only the on-chain path can enforce atomically, and secondarily by per-order expiry slots that act as dead-man switches.
- Everything custodial and economic is identical between v1 and the v2 gateway, so nothing built for v1 is thrown away: the gateway is one added instruction (verify an ed25519-signed order via the instructions sysvar, then run the same matching path) plus off-chain relay infrastructure, shipped when volume justifies it. See `13-ROADMAP.md`.

## 5. The market account: one account, three structures

Each market is a single account that grows on demand:

```
+--------------------------------------------------+
| MarketHeader (fixed size)                        |
|  question, state, times, cap, pairs, fees,       |
|  tree roots, best-price indexes, free list head, |
|  counters, bumps, rent payer                     |
+--------------------------------------------------+
| Block pool: N x 112-byte blocks                  |
|                                                  |
|   [bid order] [ask order] [seat] [free] [seat]   |
|   [ask order] [free] [bid order] [seat] ...      |
|                                                  |
|  Three red-black trees + one free list           |
|  interleaved over the same pool:                 |
|    - Bids   (sorted by price-time key)           |
|    - Asks   (sorted by price-time key)           |
|    - Seats  (sorted by owner pubkey)             |
+--------------------------------------------------+
```

- A **block** is either a resting order, a seat, or free. The account reallocates one block at a time as traders and orders arrive; each block's rent (about 0.0008 SOL) is paid by the transaction that needed it and recorded for refund at close.
- **Seats** are the ledger: `{ owner, usdc_free, usdc_locked, yes_free, yes_locked, no_free, no_locked }`. Because seats live in the market account, a fill mutates the maker's seat directly: crankless settlement.
- **Best-price indexes** are cached in the header so best bid and best ask are O(1) reads.
- Creating a market costs about 0.01 SOL of rent up front, all of it reclaimed at close. See `06-ONCHAIN-PROGRAM-SPEC.md` for exact layouts.

## 6. Custody model, stated precisely

- All USDC in a market sits in that market's **vault**, an SPL token account whose authority is a program PDA. Only the program moves it, and only through the instructions specified in `06-ONCHAIN-PROGRAM-SPEC.md`.
- A user's balances are **ledger entries in their seat**. Funds enter the seat when the user's own signed transaction transfers USDC into the vault (at order placement, net of existing free balance, or via an explicit deposit). Funds leave only through `withdraw`, which requires the seat owner's wallet signature, or through the sweep crank, which can only push a seat's balance to its owner's own token account.
- The program can debit a seat during matching because the user's placed order is the standing authorization, its price and size are hard bounds, and the program is the owner of the account holding the ledger. No instruction exists by which an admin, keeper or any third party can redirect a seat's funds anywhere except to its owner.
- Per-market vaults isolate blast radius: a defect in one market's accounting can never touch another market's funds.

## 7. Failure behavior

| Failure | Effect | Recovery |
| --- | --- | --- |
| Indexer down | App shows stale data; trading via program unaffected | Restart; state re-derived from chain |
| Fee-payer service down | Sponsored transactions fail | App falls back to user-paid fees (fractions of a cent) |
| All keepers down | Auctions do not uncross, markets do not halt/resolve on time | Anyone can run a keeper; instructions are permissionless and fee-bearing |
| Launchpad changes its account layout | Reader rejects the data | Market fails closed: halted, voided, every share redeems $0.50, all collateral returned |
| Solana congestion | Higher priority fees for keepers and traders | Escrowed funds and resting orders are unaffected; markets extend nothing and settle by deadline reads |
| Program bug | The bounded worst case is one market's vault | Per-market isolation; pause stops new markets; audits and the differential test suite (see `12-TESTING.md`) exist to prevent this class |
