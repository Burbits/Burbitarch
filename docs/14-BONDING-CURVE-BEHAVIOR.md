# Bonding-Curve Token Behavior

The complete behavioral study of bonding-curve tokens on Solana launchpads: how the venues work, how tokens actually behave from creation to graduation and after, who the actors are and what they do, and which signals predict what.

**Why this document exists:** Burbit builds none of this machinery. There is no Burbit curve, no Burbit AMM, and no dependency on any launchpad's cooperation; Burbit only reads launchpad state to settle predictions about it. This study exists because building good predictions around these tokens requires knowing their behavior cold: which events are real and irreversible (settleable), which are forgeable (never settleable), where the uncertainty lives (listable), and what the base rates are (priceable). One venue dominates the data below because it hosts 80 to 95% of the activity; every conclusion is written to generalize, and the reader architecture integrates any bonding-curve launchpad the same way. This document is the empirical foundation for the question catalog (`15-QUESTION-CATALOG.md`): every question Burbit lists is grounded in a behavior documented here, and every base rate the reference quoter prices from starts here. Statistics carry their measurement window because this ecosystem shifts regime with every fee and feature change; the collector service re-measures everything continuously.

---

## 1. The venue: what it does and what we observe

### 1.1 The shape of a launch

Burbit does not implement any of this; it watches it. Stated as an observer would:

- A token launches with a fixed supply and a fixed amount it must take in from buyers before it "finishes". On the dominant venue that amount is about **85 SOL** of net buying.
- While it is filling, buying moves it forward and selling moves it back, so **progress is not monotonic**: a token at 80% can fall to 60%. That single fact shapes how every milestone question in `15-QUESTION-CATALOG.md` must be phrased ("ever reaches X before T", never "is at X").
- When the required amount has been taken in, the token **finishes**: the venue marks it complete and moves it to an open market. This is automatic, permissionless and **irreversible**, which is what makes it the ideal thing to settle on.
- After that, the token trades on an ordinary open market. Liquidity there cannot be pulled by the creator (the position is burned at migration), so later collapses are ordinary selling, not liquidity theft.

The USD framing people quote (a market cap around $69k) floats with the price of SOL. **The amount taken in is the on-chain constant, and it is the only figure settlement or size limits ever reference.**

### 1.2 What Burbit reads, and nothing more

The venue keeps one public account per token. Burbit's reader locates it at the address the venue derives for that token, checks it is owned by the venue's program, checks its type marker, and reads four things: **has it finished**, **how much has it taken in and how much is still needed**, **who created it**, and (where the venue offers variants) **what currency it is denominated in and which launch options were chosen**. Everything else in the account is ignored.

Two consequences worth stating: progress is a number derived from the amounts, not a model we run; and venues rename or append fields across upgrades, which is why readers parse by meaning, validate strictly, and **fail closed into a void rather than guess** (`05-MARKETS-AND-SETTLEMENT.md`).

### 1.3 Fees and features timeline (each shift moved behavior)

| Date | Change | Behavioral effect |
| --- | --- | --- |
| 2024 to early 2025 | 1% flat curve fee; external-AMM migration with 6 SOL fee | Baseline era of most published datasets |
| Mar 2025 | Own AMM launches; instant free migration; LP burned | Post-graduation "LP rug" eliminated by construction; dumps remain |
| May 2025 | Creator revenue sharing (5 bps of volume) | Creators gain an income motive beyond dumping |
| Sep 2025 | Dynamic creator fees tiered by market cap (0.95% under $300k down to 0.05% over $20M) | Creator payouts jumped 10× in 24 hours; incentivized keeping tokens alive |
| Jan 2026 | Creator-fee overhaul (multi-wallet splits, rebalanced) | Corrected over-rewarding of low-effort launches |
| Feb 2026 | **Cashback coins**: creator irreversibly routes all creator fees to traders (on-chain flag at creation) | Graduations rose above 300/day; the flag is a listable market attribute |
| May 2026 | **USDC-quote curves** (and tokenized-stock quotes) via `create_v2`; USDC curves start at $4,000 mcap, bond at $58,783 | ~3% of curve trades; a distinct settlement regime the reader must handle via `quote_mint` |
| Ongoing 2026 | Current schedule: **curve fee 1.25% total** (0.30% creator + 0.95% protocol); AMM fees tiered by market cap 1.25% → 0.30% | Forcing-cost model uses 1.25% per side |
| Feature | **Mayhem mode**: opt-in at creation, an AI agent trades the coin for its first 24 hours, 2B supply, unsold agent tokens burned after 24h | Another on-chain creation flag; distinct behavior cohort |
| Feature | **King of the Hill**: ~$30k mcap (~45 SOL virtual, ~53% bonded) takes the front-page top slot | The visibility inflection traders race for; the badge is off-chain but the SOL threshold is a clean on-chain proxy |

### 1.4 Other launchpads (reader roadmap)

| Launchpad | Curve and graduation | Notes |
| --- | --- | --- |
| LetsBonk (LaunchLab-based) | 85 SOL graduation to an external AMM, LP burned; 1% fee with 30% burning its ecosystem token | Flipped the leader for ~6 weeks in mid-2025 (peaking above 60% share, 200 to 300 graduations/day) then collapsed ~98% to ~5/day within a month: launchpad dominance is itself volatile and bettable |
| LaunchLab (shared infra) | Default fixed 85 SOL ("JustSendit") or custom curves/vesting; fee caps 100 bps platform + 50 bps creator; 90% LP burned, 10% to creator as a fee-earning position | Many third-party pads are this underneath |
| Moonshot / Moonit | ~500 SOL / ~$73 to 75k mcap graduation; token burns at migration; creator milestone buybacks | Higher bar, slower graduations |
| Boop | ~400 SOL bonded to graduation + ecosystem emissions | |
| Believe | Launch by social post; anti-snipe fees decaying to 2%; ~$100k mcap graduation | Peaked at $6.3M fees in a day, then faded |
| Heaven | **No curve, no graduation**: instant pool with ~35 virtual SOL | Out of scope for graduation markets; in scope for creator/dump families |
| Bags | Curve to DEX; 1% of every trade to the creator forever; social-verified creators | |

Market share as of late 2026: the primary venue holds roughly 80 to 95% of launches and graduations; the 2025 share war (54.8% → 64% for the challenger in July, back to 70 to 90% for the incumbent by September) proves cross-launchpad comparatives are genuinely uncertain events.

## 2. The lifecycle in numbers

### 2.1 Volume of launches and the graduation funnel

- **11.9M+ tokens** launched on the primary venue since January 2024. Daily launches: record **~52,000** (Feb 2025); ~21,900/day average (Sep 2025, 655,770 in the month); 25,000+ on strong days in late 2025; ~42,000 on a peak day in mid-2026.
- **Graduation rate**: lifetime ~1.4%; measured windows: 0.37 to 1.78% daily (Apr 2025), **0.63%** (Sep 2025, full-month cohort), ~0.5% at the trough (~80/day), **1.15% weekly** (Feb 2026, after cashback coins), **0.198%** in the May to Jun 2026 regime by 24-hour survival analysis of 832,941 launches. Daily graduate counts: ~80 at troughs, ~170+ (Aug 2025), 300+ (post-cashback), 1,045 on a single peak day (Aug 2026). **The rate is a moving regime, not a constant**: it responded visibly to every fee change, feature launch and competitor wave, which is precisely what makes over/under markets on it viable products and why Burbit's collector re-measures base rates continuously rather than hardcoding them.
- **Speed**: for tokens that do graduate, **median time-to-graduation ≈ 4.4 minutes** and **median ≈ 457 trades**, heavily right-skewed (a long tail graduates hours or days later). The Sep 2025 graduated cohort absorbed ~370,000 SOL (~$74M) of net inflow.
- **Death**: survival across progress thresholds decays near-exponentially; most tokens die at low progress within minutes. 98.6% of tokens from Jan 2024 to Mar 2025 showed rug or pump-and-dump traits; only ~97k of 7M+ ever held ≥ $1,000 of liquidity; only 96 tokens ever reached $1M market cap and 18 exceeded $10M; ~3% of participants have made more than $1,000.

### 2.2 The progress bands (the map every question hangs on)

| Band (progress) | What happens there |
| --- | --- |
| 0 to ~20% | The death zone: the overwhelming majority of launches stall and die here within minutes; early trading intensity in the first few tens of transactions is the single strongest graduation predictor |
| ~25 to 95% (20 to 80 virtual SOL) | **The dump zone**: coordinated creator/insider dump events cluster here, after liquidity forms but before graduation; 92.2% of analyzed tokens had at least one dump event |
| ~53% (~45 SOL, ~$30k mcap) | The King-of-the-Hill threshold: maximum visibility, the race inflection |
| 70 to 90% | Burbit's default listing band for graduation markets: outcome genuinely uncertain (3.9% to 27.3% base rates by window), forcing the outcome still expensive, attention maximal |
| 95%+ | The final stretch: graduation probability climbs toward certainty and forcing it becomes cheap, so size limits pinch to near zero and no new markets are listed here |
| Post-complete | Graduation is final; migration executes instantly; the post-graduation phase begins |

### 2.3 Post-graduation behavior

- Within **20 minutes** of migration, **73% of tokens trade below 40% of their migration price** and **60.3% collapse below 20%**. Selling before graduation is structurally more profitable than after, which selects for pre-graduation exits and post-graduation dumps.
- LP is burned at migration, so post-graduation collapse is always a **dump**, never a liquidity pull.
- Sustained success is vanishingly rare (the 96-ever-above-$1M figure). Implication: post-graduation crash markets have high and priceable YES base rates, and "survival" snapshots (liquidity above X at T+24h) are honest longshots.

## 3. The actors

### 3.1 Creators

- **98.7% of launches include an immediate creator purchase in the mint transaction.** Creator holdings run higher in high-risk launches.
- **Serial creation is the norm**: 243k creators launched 656k tokens in one month (mean 2.7); top factories issue 50 to 268 tokens/month, and their graduation ratios (2.3 to 8.4%) beat the 0.63% baseline several-fold: deployer identity is predictive.
- **Dump behavior**: 92.2% of tokens experience at least one dump event; dumps cluster mid-curve (the 20 to 80 SOL band); failed tokens show immediate creator distribution while graduating tokens show net buying from the first blocks. Prior modeling assumed creators sell within 1/5/15 minutes at rates around 20/54/63%; the collector replaces these with live per-band measurements.
- **Fee income changed the game**: since the 2025 fee reforms, creators earn 0.30% of curve volume plus tiered AMM fees; record weeks paid creators $20M platform-wide, with top streamers earning six figures in a day. A creator with fee income has a measurable incentive not to kill the token, and **fee-claim transactions are on-chain events** Burbit can settle on.
- Deployer profit networks are documented (one mapped network netted $3.8M), and creator wallets rotate; settlement always references the literal on-chain `creator` signer, never "the same person".

### 3.2 Snipers and bundlers

- **More than half of tokens are sniped in their creation block.** A one-month study found 15,000+ tokens sniped by wallets funded directly by the deployer (4,600+ sniper wallets, 10,400+ deployers, ~1.75% of issuance), with **87% of snipes profitable, 55% fully exited within 1 minute and ~85% within 5 minutes**, netting 15,000+ SOL, concentrated in US business hours.
- **Bundled launches**: coordinated accounts held **36.5% of supply on average** across 41,470 launches; public bundler tooling creates a token plus 25 buys in one bundle. Bundle detection raises measured top-10 concentration by a median +24% in high-risk tokens.
- **Wash and bots**: 21.4% of pre-migration transactions in one dataset were wash trades; bots generate 60 to 80% of volume on some tokens; tokens with **more than ~70% human (non-bot) trade share graduate materially more often**. Consequence for Burbit: **volume, trade-count and unique-wallet metrics are forgeable at roughly the cost of fees and are therefore never open-settlement criteria** (see the catalog's not-offered list).

### 3.3 Whales and copy traders

- A handful of famous wallets move curves: the most-copied ran 14,794 trades across 1,287 tokens in a week (28.5% hit rate, +$663k in 30 days), holding seconds to minutes. Tracked influential wallets show median 57% win rates, and 4+ influential wallets converging on a token with $1,000+ combined spend measurably outperforms.
- Copy-trade bots amplify any visible whale entry within the same block. "Named wallet W buys token X" is a crisp, settleable, genuinely uncertain on-chain event, and one of the most-watched signals in the ecosystem.

### 3.4 The insider-blocking residual

Bundled insiders holding large supply can stall a near-graduation by dumping into it. Simulation puts blocked would-be graduations at 1.7% (5-minute windows) rising to 9.1% (50-minute windows). This is Burbit's disclosed residual risk and the reason default windows are short (`05-MARKETS-AND-SETTLEMENT.md` section 5).

## 4. What the industry measures (and what that teaches question design)

Trading terminals converge on a shared vocabulary: bonding %, migration status, dev holding % and dev-sold alerts, top-10 %, holder count, sniper %, bundle %, insider %, fresh wallets, LP burned, authorities revoked, risk scores, plus off-chain flags (paid listings, social handle reuse, community takeovers). Two hard lessons fall out of auditing their definitions:

1. **Vendor labels are not settlement material.** "Sniper %" means the first ~70 buyers to one tool, same-block buyers to another, first-hour cohorts to academics. "Insider" means transfer-recipients in one tool and team wallets in another. "Bundled" spans same-transaction, same-block, relay-bundle and funding-graph definitions. Burbit never settles on any of them; where the underlying behavior matters, the catalog reduces it to a pinned primitive ("≥ X% of supply purchased in the creation block") or does not offer it.
2. **Off-chain signals are pricing features only.** Social presence multiplies graduation odds (advertised launches graduated at 1.485% vs 0.166% unadvertised in one 860k-launch study), paid listings and front-page badges move flow, and streams mint fees. None of it is settleable; all of it is priceable by traders, which is exactly the division of labor a prediction market wants.

## 5. Settlement-quality hierarchy (the bridge to the catalog)

From all of the above, every measurable event falls into one of four settlement grades, which `15-QUESTION-CATALOG.md` applies to every question:

| Grade | Definition | Examples |
| --- | --- | --- |
| **A: one-way flag, single read** | Irreversible once true; witnessed by one account read or one instruction | `complete == true`; `migrate` executed; pool created; fee-claim instruction; creator-signed sell; a named wallet's trade; creation-mode flags |
| **B: threshold with an "ever" witness** | Reversible quantity, phrased as "ever reaches X before T", witnessed at one slot | progress ≥ X%; real SOL ≥ X; KotH-threshold proxy; creator balance ever below X |
| **C: snapshot** | Value at a single named slot | pool liquidity at T+24h; top-10 share at deadline (with pinned exclusions) |
| **D: aggregate count** | Monotone count over a pinned window and cohort | daily graduations; daily launches; cross-launchpad comparatives |
| **Rejected** | Vendor labels, off-chain facts, forgeable-at-fee-cost metrics, oracle-priced USD levels | sniper/bundle/insider %, social signals, volume and trade counts as open markets, USD price targets |

Grades A and B settle inside Burbit's program from the launchpad state account and instruction witnesses recorded by permissionless cranks; grade C settles from a snapshot read at the named slot; grade D settles by bonded proposal (`05-MARKETS-AND-SETTLEMENT.md` section 8) because it aggregates beyond single accounts. Every question in the catalog carries its grade, its exact read, who could force it and at what cost, and its offerability class.
