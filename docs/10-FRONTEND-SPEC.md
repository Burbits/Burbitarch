# Frontend Specification

Every screen, every flow, and the interaction grammar of the Burbit app. The app is a thin, fast client over the API (`08-INDEXER-AND-API-SPEC.md`) and the program (`06-ONCHAIN-PROGRAM-SPEC.md`): it holds no funds, and every consequential action is a Solana transaction the user's keys authorize.

---

## 1. Principles

1. **Hide the plumbing, never the custody.** Users see dollars, cents and probabilities; they never see lamports, blocks or accounts. But the wallet is theirs, withdrawal is always one tap, and nothing ever asks them to trust Burbit with funds.
2. **One tap to bet.** After a one-time session-key grant, placing a bet is: tap price, confirm amount, done. No popup, no fee, sub-second confirmation.
3. **The feed is the product.** The launch feed, a live view of every token racing up its curve with odds attached, is the front door and is valuable even to people who never trade.
4. **Prices are probabilities.** 57¢ and "57% chance" are the same number shown in the right context.

## 2. Onboarding and wallet flows

- **Connect**: standard Solana wallet adapters (Phantom, Backpack, Solflare, and the injected/WalletConnect long tail). No email or social login; no embedded-wallet vendor. Connecting signs nothing.
- **Enable trading (first trade)**: one wallet-signed transaction that (a) registers a browser-generated **session key** valid for a user-chosen duration (default 24 h, max 7 days), scoped to place/cancel only, and (b) if needed, makes the user's first deposit. The modal explains in one sentence: "This key can place and cancel bets. It can never withdraw. Revoke anytime."
- **Deposit**: USDC amount input with wallet balance shown; builds `deposit` for the chosen market, or a pre-fund into the market the user is on. (Because seats are per market and balances sweep back automatically at market close, the app frames deposits as "bet balance" and tops up per bet from the wallet when the session flow can't cover cost from free balance; the common path locks funds only when an order is placed.)
- **Withdraw**: from Positions, one wallet-signed `withdraw` per market with a free balance, or "withdraw all" batching them. Destination defaults to the wallet's USDC account and may be any account the wallet owns.
- **Revoke session key**: Settings, one tap, immediate.

## 3. Screens

### 3.1 Launch feed (home)

- A live-sorted list (WebSocket `launches` channel) of tokens approaching graduation. Each row: token image/symbol/name, launchpad badge, **progress bar** with live percentage, SOL in curve, age, creator flags (bonded badge if a rug market exists), and, when a Burbit market is open, **the live YES price as odds** ("Graduates in 15 min: 12%") with inline **Yes/No quick-buy buttons** and the market countdown.
- Filters: progress band, launchpad, has-market, bonded-only. Sort: progress, volume, newest.
- Rows animate on progress ticks and flip to a "Graduated" state with a link to the resolved market.

### 3.2 Market screen

Layout, desktop (single column stacked on mobile with the trade panel as a bottom sheet):

- **Header**: question ("Will $TOKEN graduate before 18:00?"), token identity, market state chip (Auction with countdown / Live / Halted / Resolved YES/NO/Void), deadline countdown, and the live **progress bar** under it, because the underlying is always visible context.
- **Chart**: YES price (probability) over time, 1s to 1m resolutions from `candles`; the headline number is the bid-ask midpoint; last-trade fallback when the spread exceeds 10¢.
- **Order book**: bids and asks around the spread, price in cents, size in shares and dollars, cumulative depth shading; the user's own resting orders marked inline with one-tap cancel.
- **Trade panel** (sticky right rail / bottom sheet):
  - Tabs **Buy | Sell**, outcome toggle **YES | NO**, mode **Quick | Set odds**.
  - **Quick** (IOC at best price with slippage bound): a dollar input with chips (+$1, +$5, +$20, Max), live estimate of shares, average price and **"To win $X"**, one button. During Auction the panel switches to "Join the opening auction" with a limit-price field and the note that all auction fills clear at one price.
  - **Set odds** (limit): price stepper in cents (fine ticks unlock below 5¢/above 95¢), shares or dollar input, optional expiry (30 s / 2 min / 5 min / custom), optional **progress guard** ("cancel if the curve passes 85%") pre-filled to sensible bounds, post-only toggle for quoters.
  - Fees shown inline: "Fee 2% ($0.40); part of this goes to the sellers of the shares you take."
  - Confirmation is inline (no modal): the button becomes "Bought 100 YES @ 21¢" on the fill event.
- **Position strip**: the user's YES/NO shares, average cost, live value at mid, unrealized PnL, buttons Sell, Merge (enabled when holding both sides), and, post-resolution, **Claim**.
- **Tabs**: Trades (live tape), Holders (top YES and NO holders by size), Activity (all order events), Rules (the exact settlement rule, the launchpad state account address, the deadline and close time, the cap and current open interest, plus the residual-risk disclosure for the family), About (token metadata and links).
- **Rug protection block** on tokens with a creator-written rug market: bonded badge, coverage ratio, YES price as "insurance cost", one-tap buy.

### 3.3 Positions (portfolio)

- Header: total value (free balances + positions at mid), claimable winnings, "Withdraw all".
- **Open** tab: rows per market: question, side and size, avg cost, mid, value, PnL, state chip, quick Sell/Cancel actions.
- **Open orders** tab: every resting order with price, remaining, expiry, guard, one-tap cancel and cancel-all.
- **History** tab: fills, redemptions, sweeps, deposits, withdrawals with explorer links.
- Resolved-but-unclaimed positions surface a prominent **Claim** (single `redeem` + `withdraw`), though the sweeper normally pays out before users return; swept payouts appear in History as "Paid out automatically".

### 3.4 Leaderboard and profiles

- Leaderboard by PnL and volume over day/week/all (positions are public on-chain; this is stated plainly).
- Public profile per wallet: PnL curve, win rate, biggest wins, current positions. Opt-out hides nothing on-chain and the UI says so.

### 3.5 Creator console

- For a connected wallet that created a tracked token: "Write a rug market" flow: escrow the bag, choose bond and expiry, review "withdrawing custody before expiry pays YES holders from your bond", sign. Live view of coverage ratio, YES sales, premium earned; the custody-withdraw button carries the consequence in red.

## 4. States, errors and edge UX

| Situation | UX |
| --- | --- |
| Market halts while the user types | Panel flips to "Trading ended: graduated 🎉 (or closed)"; resting orders show "returned to balance" |
| Order guard-voided | Toast: "Your quote was cancelled: the curve moved past your guard. Funds returned." |
| Cap reached | Buy panels show "Market is at its safety cap; only matching against existing orders is possible"; mint-requiring quantities are clipped with an explainer link |
| Auction phase | All prices labeled "indicative until the opening cross"; a countdown to uncross |
| Sponsor service down | Transparent fallback to wallet-paid fees ("network fee ~$0.001") |
| Session key expired | Inline re-grant, one signature |
| Void | Banner: "This market was voided (unreadable launchpad data). Every share pays $0.50. Funds returned." |
| Insufficient balance | The shortfall is added to the order transaction as a deposit when the wallet is present; session-only context prompts a top-up |

## 5. Design language

- Dark, dense, fast: near-black canvas, high-contrast numerals, a terminal that degens trust. Light theme available.
- **Green = YES/bids/profit, red = NO/asks/loss**, one accent color for actions.
- Prices in **cents** everywhere tradable; **% chance** in feed and social contexts; payouts always framed as "To win $X".
- Progress bars are first-class citizens: every market visual anchors to the curve.
- Numbers animate on change; nothing on the screen is ever more than a slot stale (WebSocket-driven).
- Mobile: bottom-tab navigation (Feed, Markets, Positions, Leaderboard), trade panel as bottom sheet, swipe to quick-bet from the feed.

## 6. What the frontend never does

It never holds keys beyond the session key it generated (stored in origin-scoped storage, wrapped where the platform allows), never proxies withdrawals, never signs anything with anything but the user's keys, and never shows a balance that is not reconstructible from chain state. If api.burbit.xyz vanished, an SDK script against any RPC could reproduce every number on the screen.
