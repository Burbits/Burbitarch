# Indexer and API Specification

The read-side infrastructure: how chain state becomes the launch feed, books, trades and positions, and the exact REST and WebSocket interfaces the app and third-party bots consume. Nothing in this layer holds funds or authority; everything it serves is re-derivable from the chain, and all write operations happen through the program directly (see `06-ONCHAIN-PROGRAM-SPEC.md`).

---

## 1. Indexer

### 1.1 Inputs

1. **Burbit program events**: the self-CPI event log (every Fill, OrderPosted, OrderDone, Halted, Resolved, and so on) via transaction subscription on the program id, plus periodic full-account snapshots of Market accounts for reconciliation.
2. **Launchpad accounts**: account subscription on each registered launchpad program; every curve-account update yields `{mint, progress, real_sol_reserves, complete, creator}`; every create instruction yields a new launch row.
3. **Creator wallets**: token-account subscriptions for creators of tokens with active dump or rug markets.
4. **Pyth SOL/USD**: for displaying caps and USD conversions.

### 1.2 Derived state

| Store | Contents | Source of truth |
| --- | --- | --- |
| `launches` | Every tracked token: mint, launchpad, creator, created_at, live progress, reserves, complete, graduated_at | Launchpad accounts |
| `markets` | Every Burbit market: question, state, times, cap, pairs, outcome, volume, last price, best bid/ask | Market accounts + events |
| `books` | Full depth per market, maintained from OrderPosted/Fill/OrderDone events, reconciled against account snapshots | Events (chain is truth) |
| `trades` | Every fill: market, price, size, kind, maker, taker, fee, slot | Fill events |
| `candles` | 1s/15s/1m OHLCV of YES price per market | Trades |
| `positions` | Per owner per market: shares, locked, USDC balances, open orders, realized and unrealized PnL | Events |
| `leaderboard` | PnL and volume per wallet over day/week/all | Trades + redemptions |

Reconciliation rule: on any divergence between event-derived state and an account snapshot, the snapshot wins, the divergence is logged as a defect, and affected consumers are refreshed.

### 1.3 Guarantees

At-least-once event processing with idempotent upserts keyed by `(market, seq)`; monotonic per-market sequence exposure so consumers can detect gaps; snapshot-plus-diff subscription semantics on the WebSocket (section 3). Target freshness: one slot behind the chain.

## 2. REST API

Base: `https://api.burbit.xyz/v1`. All responses `{ "data": ..., "ts": <ms> }`. Public, no auth (there are no write endpoints; trading is on-chain). Pagination via `limit` (≤ 500) and opaque `cursor`. Amounts are strings of base units; prices are mills.

### 2.1 Launches and markets

| Endpoint | Returns |
| --- | --- |
| `GET /launches?status=active&min_progress=6000&sort=progress` | Launch feed rows: mint, symbol, name, image, launchpad, creator, progress_bps, real_sol_reserves, age_s, complete, market_ids |
| `GET /launches/{mint}` | One launch + its markets |
| `GET /markets?state=continuous&family=graduation&sort=volume` | Market summaries: id, question, mint, state, deadline, close_ts, best_bid, best_ask, mid, last, pairs, cap, volume, fees |
| `GET /markets/{id}` | Full market detail incl. question params, times, outcome, creator flags |
| `GET /markets/{id}/book?depth=50` | `{bids: [[price, size], ...], asks: [...], seq}` |
| `GET /markets/{id}/trades` | Recent fills |
| `GET /markets/{id}/candles?res=15s&from=&to=` | OHLCV |
| `GET /markets/{id}/holders` | Top YES and NO holders with sizes |

### 2.2 Accounts

| Endpoint | Returns |
| --- | --- |
| `GET /accounts/{owner}/positions?status=open` | Seats across markets: balances, shares, open orders, claimable |
| `GET /accounts/{owner}/orders?market=` | Open orders with ids, prices, sizes, remaining |
| `GET /accounts/{owner}/activity` | Deposits, orders, fills, redemptions, sweeps |
| `GET /accounts/{owner}/pnl?period=all` | Realized + mark-to-mid unrealized |
| `GET /leaderboard?by=pnl&period=day` | Ranked wallets |

### 2.3 Reference data and transactions

| Endpoint | Returns |
| --- | --- |
| `GET /config` | Live protocol parameters (fees, α, ticks, keeper fees) |
| `GET /stats` | Totals: markets, volume, open interest, fees, graduation base rates by milestone |
| `POST /tx/sponsor` | Body: a fully signed-by-session-key transaction, base64. The fee-payer service validates it is a Burbit `place_order`/`cancel` for the signing session key, co-signs as fee payer, submits, returns the signature. Rate-limited per key. This is the only POST endpoint and it can spend nothing but its own SOL |

## 3. WebSocket API

`wss://ws.burbit.xyz/v1`. JSON messages `{op, channel, ...}`; subscribe with `{op:"sub", channel, params}`; every channel sends one `snapshot` then `diff` messages carrying the market `seq` so clients can detect gaps and resubscribe.

| Channel | Params | Snapshot | Diffs |
| --- | --- | --- | --- |
| `book` | market | Full depth + seq | `{seq, side, price, size}` per level change (size 0 removes) |
| `trades` | market | Last 50 | Each fill: price, size, kind, taker side, ts |
| `market` | market | Summary | State changes, best bid/ask, pairs, cap, halted, resolved |
| `launches` | filters | Current feed page | Progress ticks, new launches, graduations |
| `account` | owner (signed challenge: the client signs a nonce with the wallet or a session key to prove ownership; read-only privacy gate, not custody) | Positions + open orders | Order posted/filled/done, balance changes, redeemable |

Heartbeat: server `ping` every 10 s; client replies `pong` within 10 s or is dropped. No replay on reconnect: clients resubscribe and receive fresh snapshots.

## 4. Bot integration flow (reference quoter and third parties)

1. `GET /markets?state=continuous` (or the `launches` channel) to find targets; read `config` for ticks and fees.
2. Maintain books via the `book` channel; maintain own state via the `account` channel.
3. Place quotes by building `place_order` transactions directly against the program (the SDK wraps this), with curve guards tight around current progress and `expiry_slot` a few hundred slots out as a dead-man switch; refresh by `cancel_all` + re-place in one transaction.
4. There is no off-chain order gateway in v1 and therefore no API key, no HMAC, and no rate limit on trading itself: the chain is the rate limit. The only authenticated surfaces are the private `account` channel and the sponsor endpoint.

## 5. SDK

A TypeScript SDK (`@burbit/sdk`) ships with: account fetch/decode for every layout in `06-ONCHAIN-PROGRAM-SPEC.md`; instruction builders for every user-facing instruction; a session-key manager (create, register, store, expire); a sponsor-submit helper; book/position watchers over the WebSocket; and the reference quoter built on all of it. A Rust crate (`burbit-client`) mirrors the account decoding and instruction builders for keeper authors.
