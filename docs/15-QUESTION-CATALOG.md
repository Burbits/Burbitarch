# Question Catalog

The complete space of prediction questions Burbit can run on bonding-curve tokens: every question type, its exact settlement rule, its manipulation analysis, whether and how it is offered, and the behavioral hypothesis behind it. Grounded in `14-BONDING-CURVE-BEHAVIOR.md`; settlement grades (A one-way flag, B ever-witness, C snapshot, D aggregate) are defined there in section 5. This catalog is normative for listing policy: a market may only be created if its question type appears here with an offerability class other than NOT OFFERED.

---

## 1. Conventions that apply to every question

1. **Phrasing discipline.** Irreversible events settle YES the moment witnessed and NO at the deadline. Reversible quantities are only ever offered as **"ever reaches X before T"** (witnessed at one slot and recorded by the permissionless `record_witness` crank or any trading instruction that observes it) or as **"value at slot S"** (a single named snapshot). Nothing settles on an average, a vendor label, or an off-chain fact.
2. **Time.** All deadlines are cluster time; a "UTC day" is published as an explicit slot range at market creation. Ties break by slot, then transaction index.
3. **Reads.** Settlement references on-chain semantics (reserves, flags, signers, instruction executions), never byte offsets; readers fail closed into VOID (`05-MARKETS-AND-SETTLEMENT.md`).
4. **Offerability classes.**
   - **OPEN**: order book with the open-interest cap = α × forcing cost, recomputed live.
   - **SMALL**: order book with a fixed small cap (default $50) because a participant can force the outcome; the app labels why.
   - **MICRO**: fixed cap $10; novelty markets where forcing is near-free for some party; labeled.
   - **CREATOR**: creator-written and creator-bonded only (`05` section 7 mechanics).
   - **PROPOSAL**: settled by bonded proposal with a challenge window (`05` section 8), for aggregates no single account holds.
   - **NOT OFFERED**: listed here with the reason, so the exclusion is policy, not oversight.
5. **Forcing-cost formulas used below.**
   - `F_grad(s)` = cost to force graduation from progress s: buy out the remaining curve, minus recovery selling into the post-graduation pool (the table in `05` section 4; 19.5 SOL at launch, 8.6 at 70%, 2.5 at 90%).
   - `F_push(s→x)` = cost to push progress from s to x **and back**: a round trip on the curve returns the attacker's SOL minus fees, so `F_push ≈ 2 × 1.25% × SOL_required(s→x)` plus curve asymmetry. This is the honest forcing cost of every "ever reaches X%" question, and it is small: caps on those markets are correspondingly small by formula, not by exception.
   - `F_wash(V)` = cost to fabricate V of volume ≈ 2.5% × V (round-trip fees). This is why volume-family questions are not open markets.
   - `F_sybil(N)` ≈ 0.002 SOL × N (token-account rent) to fabricate N holders. This is why holder-count questions are not open markets.

---

## 2. Family A: Graduation of a single token

The flagship family. Settlement grade **A**: `complete == true` on the token's curve account, observed before the deadline (the observation slot is recorded at halt). Forcer: YES by buying out the curve (`F_grad`), NO by insiders stalling (residual, disclosed). Creator barred from trading. Cap = α × F_grad × SOL/USD floor, live.

| ID | Question template | Parameters | Class | Notes and hypothesis |
| --- | --- | --- | --- | --- |
| A1 | Will TOKEN graduate before time T? | T = wall-clock deadline | OPEN | The canonical market. Listed at 70 to 90% progress with 5-to-15-minute windows |
| A2 | Will TOKEN graduate within N minutes of listing? | N ∈ {5, 10, 15} | OPEN | Base rates by listing milestone: at 70% progress, 3.9% (5 min), 5.4% (15 min); at 90%, 22.0% and 27.3%. Hypothesis: retail overprices YES near hype peaks, giving NO-side edge that the quoter monetizes |
| A3 | Will TOKEN graduate within N minutes of its creation? | N | OPEN | Median graduate takes 4.4 minutes from creation; long right tail. Only listed while the answer is open |
| A4 | Time-to-graduation bucket | buckets: under 5 min / 5 to 15 / 15 to 60 / never (from listing) | OPEN, multi-outcome | Complete-set market (`05` section 6). Uses the recorded completion slot; per-second-sensitive bucket edges are not listed |
| A5 | Final-stretch express: graduates within 5 minutes? | listed at ≥ 90% | OPEN | High base rate (22%+), tiny cap (F_grad ≈ 2.5 SOL at 90%), fast turnover; the "scalp" product |
| A6 | Will TOKEN ever graduate before end of day? | day boundary | OPEN | Longer windows raise insider-blocking from 1.7% toward 9.1%; disclosed in Rules; cap unchanged |
| A7 | Will this USDC-curve token graduate before T? | `quote_mint` = USDC | OPEN | Reader branches on `quote_mint`; forcing cost is natively in USDC (bond at $58,783), so **no price oracle is needed for the cap**: the cleanest cap math in the catalog |
| A8 | Will this mayhem-mode token graduate within its 24-hour agent window? | `is_mayhem_mode` | OPEN | A distinct behavioral cohort (an AI agent trades it for 24h, 2B supply); collector measures its own base rate before wide listing |
| A9 | Will this cashback coin graduate before T? | `is_cashback_coin` | OPEN | Cashback launches lifted platform graduations above 300/day; flag read at creation |
| A10 | Race-to-graduation vs deadline for a token on another launchpad | reader per launchpad | OPEN | Same mechanics under each registered reader; thresholds differ (85 SOL, ~400, ~500 SOL) so caps differ |
| A11 | "Will it NOT graduate?" | any of the above | n/a | Not a separate market: it is the NO side of the same book. Documented so nobody lists a duplicate |
| A12 | Will migration execute within N slots of completion? | N | NOT OFFERED | Degenerate since the venue's own AMM era: migration is instant and permissionless; the answer is always yes |

## 3. Family B: Curve progress and milestones

Grade **B** (ever-witness). Forcer: anyone, via `F_push`, so caps are small by formula. These are the "momentum" products: fast, cheap, high-frequency.

| ID | Question template | Settlement read | Class | Notes and hypothesis |
| --- | --- | --- | --- | --- |
| B1 | Will TOKEN's curve ever reach X% before T? | progress from reserves, ever-witnessed | OPEN (cap = α × F_push) | The stepping-stone market (for example "reaches 90% within 10 minutes" listed at 60%). Death is overwhelmingly likely below 20%, so lines sit in the 50 to 95% band where uncertainty is real |
| B2 | Will real SOL in the curve ever reach X SOL before T? | `real_sol_reserves` | OPEN (α × F_push) | Equivalent to B1 in different units; listed only when it reads better than a percent |
| B3 | Will TOKEN hit the King-of-the-Hill threshold before T? | curve-computed SOL market cap ≥ 45 SOL, ever-witnessed | OPEN (α × F_push) | The proxy for the front-page crown (~53% bonded). The badge itself is off-chain and never referenced. Hypothesis: the KotH race is the most-watched sub-graduation event in the ecosystem and has no market anywhere |
| B4 | Will TOKEN reach X% before its creator first sells? | first-witnessed of: progress ≥ X vs creator-signed sell | MICRO | A two-event race the creator half-controls; micro cap, labeled |
| B5 | Will price ever reach k× launch before T? | curve price | NOT OFFERED | Pure pump-and-revert: an attacker round-trips the curve for fees only; the manipulation studies put averaged/snapshot price settlement at 42 to 85% attacker profitability. Progress questions (B1) already express the same view safely because their forcing cost formula is honest |
| B6 | Trade count ≥ N before T | buy/sell instruction count | NOT OFFERED | Forgeable at ~zero cost per trade |
| B7 | Cumulative buy volume ≥ V before T | sum of buys | NOT OFFERED | `F_wash(V) ≈ 2.5% × V`: a $10,000 volume line costs ~$250 to force; no cap worth running survives that |
| B8 | A single buy ≥ X SOL occurs before T ("whale print") | any buy instruction ≥ X | MICRO | Anyone with X SOL forces it for round-trip fees; kept as a $10 novelty because demand exists |
| B9 | Named wallet W trades TOKEN before T ("will the whale ape?") | any curve/pool trade signed by W | MICRO | Crisp and hugely watched (copy-trading culture), but W or anyone colluding with W forces it freely; micro cap, prominently labeled |

## 4. Family C: Creator behavior

Grade **A/B**. The creator controls most of these outcomes, which is exactly why the flagship product makes the creator the writer and the payer (`05` section 7). Open versions exist only at SMALL caps as sentiment gauges.

| ID | Question template | Settlement read | Class | Notes and hypothesis |
| --- | --- | --- | --- | --- |
| C1 | Will the creator sell any tokens before T? | a `sell` instruction signed by the curve's `creator` (transfers deliberately excluded; the crisp definition) | SMALL | Prior estimates: ~54% of creators sell within 5 minutes, 92.2% of tokens see at least one dump event; collector re-measures per progress band. The most-asked question in the ecosystem |
| C2 | Will the creator's balance ever fall below X% of their initial bag before T? | creator ATA, ever-witnessed | SMALL | The graded version of C1 ("sold half") |
| C3 | Will the creator dump everything before T? | creator ATA reaches 0 after being > 0 | SMALL | |
| C4 | Will the creator claim fees before T? | `collect_creator_fee` (curve) or the AMM fee-claim instruction, signed by creator | MICRO | Creator-controlled but a real behavioral tell since fee income became significant in 2025 |
| C5 | Will cumulative creator fees on TOKEN reach X SOL by T? | fee-vault balance + claimed amounts | OPEN (cap = α × F, where F ≈ 3.2 × X: a creator wash-trading their own token nets 0.30% back of the 1.25% they pay, so fabricating X of creator fees costs about 3.2X) | The one volume-linked question with an honest forcing floor, because the forcer loses most of the wash cost. A proxy market on "does this token have a real afterlife" |
| C6 | Will the creator launch another token within T? | another `create`/`create_v2` signed by the same wallet | MICRO (open) / CREATOR (bonded pledge) | Serial deployers launch 50 to 268 tokens/month. The creator-bonded version ("I will not deploy again for 7 days, bonded") is a sellable commitment device |
| C7 | Will the creator still hold ≥ X% at graduation? | creator ATA at the completion slot, conditional on completion (void if no graduation) | CREATOR | The "diamond-hands bond": creators sell YES-on-holding as a credibility instrument |
| C8 | Creator-written rug market: will the creator withdraw custody before expiry? | custody withdrawal in the bond program is the event itself | CREATOR | The flagship (`05` section 7): bond escrowed, NO locked to creator, YES sold as insurance, coverage ratio displayed |
| C9 | Will the creator sell nothing before graduation or deadline? | complement of C1 as a creator-written market | CREATOR | The positive framing of C1 that a confident creator can monetize |

## 5. Family D: Death and stalls

No separate markets: every death question is the NO side of a Family A or B book ("does not graduate by T" = NO on A1; "never reaches 50%" = NO on B1). Documented so listing policy never duplicates a book and splits liquidity. "No trade for W minutes" style absence questions are NOT OFFERED: settlement by absence needs a canonical full-history read that a single account cannot witness, and the same view is expressible as the NO side of a milestone market.

## 6. Family E: Post-graduation

Grade **B/C** on the destination pool account (a second reader parses the AMM pool: reserves, LP state). Listed automatically the moment a graduation halts a Family A market, so attention rolls forward.

| ID | Question template | Settlement read | Class | Notes and hypothesis |
| --- | --- | --- | --- | --- |
| E1 | Will the pool's SOL reserve ever fall below X within N minutes of migration? | pool reserves, ever-witnessed | SMALL | Any large holder can force by dumping (they bear the dump's cost, which is real but modest for insiders); capped small, sold as crash insurance |
| E2 | Will price ever fall below 40% of migration price within 20 minutes? | pool price vs migration print, ever-witnessed | SMALL | The canonical stat: base rate ~73% YES (and ~60% for the below-20% variant). Deliberately listed as the "reality check" product; hypothesis: newcomers systematically underprice YES |
| E3 | Will pool liquidity be ≥ X at T + 24h? | snapshot | MICRO | Snapshot forceable by a temporary LP deposit around the slot; micro novelty only |
| E4 | Will TOKEN trade above its migration price at T + 1h? | price snapshot | NOT OFFERED | Price-snapshot rule (paintable in one print) |
| E5 | LP burned? Mint/freeze authority revoked? | pool/mint accounts | NOT OFFERED | Degenerate on this launchpad: true by construction at migration/creation; kept as app-displayed facts, not markets |

## 7. Family F: Races and cohorts

Grade **A** across multiple named curve accounts (each passed into settlement), or **D** for open cohorts.

| ID | Question template | Settlement | Class | Notes |
| --- | --- | --- | --- | --- |
| F1 | Which of these K tokens graduates first? | earliest recorded completion among K named curves; "none" bucket at deadline | OPEN, multi-outcome | Cap = α × min forcing cost in the group. The trench-warfare product: pick the winner of the hour |
| F2 | Will any of these K tokens graduate before T? | any completion among named set | OPEN | Cap by cheapest F_grad in the set |
| F3 | Will the first token created after T0 to graduate be created within N minutes of T0? and similar open-cohort firsts | full-history over the launchpad | PROPOSAL | Cohort enumeration exceeds single-account reads |
| F4 | Will today's fastest creation-to-graduation time beat X minutes? | cohort minimum | PROPOSAL | The "speedrun record" market |

## 8. Family G: Launchpad-wide aggregates

Grade **D**, settled by bonded proposal (`05` section 8). The proven headline format: general-purpose venues have run daily-graduation-count binaries at exactly this shape, and the underlying counts are public dashboard staples.

| ID | Question template | Class | Notes and hypothesis |
| --- | --- | --- | --- |
| G1 | Daily graduations over/under N? | PROPOSAL | Line set from the collector's trailing base rate. Regime history spans ~80/day troughs to 1,045/day peaks, and rates jumped on every fee/feature change: genuine macro uncertainty |
| G2 | Daily launches ≥ N? | PROPOSAL | 21,900/day averages with 52,000 peaks |
| G3 | Will ≥ p% of tokens created on day D graduate within 24 hours? | PROPOSAL | Cohort pinned to created-that-day; survival-analysis base rates ranged 0.198% to 1.15% across 2025 to 2026 regimes |
| G4 | Will launchpad L out-graduate the incumbent on day D? | PROPOSAL | The 2025 share war (a challenger at 60%+ share for weeks, then a 98% collapse) proves this is genuinely bettable |
| G5 | Weekly graduations bucket | PROPOSAL, multi-outcome | |
| G6 | Will any token created today graduate within 10 minutes of creation? | PROPOSAL | |
| G7 | Will cumulative protocol buyback/fee inflows reach X SOL by date D? | PROPOSAL | SOL-denominated only; the venue's revenue and buyback wallets are public |
| G8 | Will ≥ N tokens cross the King-of-the-Hill threshold today? | PROPOSAL | |

## 9. Family H: Creation-mode and structural facts

Creation-time flags (`is_mayhem_mode`, `is_cashback_coin`, `is_holder_reward`, `creator_fee_bps`, `quote_mint`) are **facts fixed at launch**: they are market *qualifiers* (they define cohorts like A7 to A9 and filter the feed), not questions, because they are already determined when any market could open. Event-shaped structural questions:

| ID | Question | Class | Reason |
| --- | --- | --- | --- |
| H1 | Will TOKEN's metadata be changed before graduation? | MICRO | Creator-controlled event; crisp (update transaction witnessed) |
| H2 | Will ≥ X% of supply be burned before T? | MICRO | Holder/creator-controlled; cumulative burn instructions |
| H3 | Will today's graduations include ≥ N cashback coins? | PROPOSAL | Aggregate over creation flags |
| H4 | "≥ X% of supply was bought in the creation block" | n/a | A retrospective fact, fully determined at launch: used as a feed filter and a Rules disclosure on affected tokens, never a market |

## 10. Family I and the full NOT OFFERED register

Every excluded question type, with the reason stated as policy:

| Question type | Reason for exclusion |
| --- | --- |
| Any USD price or market-cap target (reaches $X, mcap $69k, "$1M club") | Needs a price oracle and settles on paintable prints; SOL-progress questions express the same views with honest forcing math |
| Any "price at time T" or averaged-price rule | Manipulation-profitable in 42 to 85% of simulated attempts; the founding exclusion |
| ATH price/mcap within a window | Settles on trade prints that MEV and wash trades can set |
| Volume ≥ V, trade count ≥ N, buys/sells ≥ N | `F_wash ≈ 2.5% × V` and near-zero per-trade cost: forgeable for pocket change |
| Unique buyers ≥ N, holder count ≥ N, top-10 share thresholds | `F_sybil ≈ 0.002 SOL per holder` and free wallet-splitting |
| Sniper %, bundle %, insider %, fresh-wallet %, cluster %, bot %, any vendor risk score, "rugged" labels | Contested proprietary definitions; no two tools agree; never settleable. Where the behavior matters, it appears only as a pinned primitive (H4) or a disclosure |
| Anything off-chain: social accounts and renames, follower counts, influencer mentions, paid listings and boosts, front-page badges, livestreams, community-takeover status, exchange listing announcements | Not readable by the program; used as pricing context by traders, never as settlement |
| Migration-executes, LP-burned, authorities-revoked on this launchpad | Degenerate: true by construction |
| Absence-of-activity windows ("no trade for 10 minutes") | Settlement-by-absence exceeds single-account witnessing; expressible as NO sides of milestone markets |

## 11. Hypothesis library (what each family monetizes)

| Market | Base rate (measured window) | The tradeable hypothesis |
| --- | --- | --- |
| A2 graduation at 70%/15 min | 5.4% (simulation; collector re-measures) | Hype flow overprices YES; quoters harvest the gap; sharps fade launches with bot-dominated tape (>70% human share predicts graduation) |
| A5 final stretch at 90%/5 min | 22.0% | The tightest race; insiders' stalling power is the real YES/NO edge |
| B3 King-of-the-Hill | to be measured live | First market anywhere on the ecosystem's most-watched sub-graduation event |
| C1 creator sells before 5 min | ~54% prior; 92.2% of tokens see a dump event | Insurance demand meets creator-reputation signaling; serial-deployer history (2.3 to 8.4% graduation ratios) is public pricing alpha |
| C5 creator fees ≥ X | new since 2025 fee regime | A clean "does this token have a real afterlife" proxy with honest forcing math |
| E2 below 40% within 20 min | ~73% YES | The reality-check market; prices teach the base rate the ecosystem ignores |
| G1 daily graduations | regime-dependent (80 to 1,045/day) | Macro bets on fee changes, feature launches, and launchpad wars |
| F1 first-to-graduate races | n/a | Converts trench tribalism into priced competition |

## 12. Listing playbook (default automation)

The market-creator keeper applies this policy; the program independently re-verifies every milestone:

| Trigger | Markets auto-created |
| --- | --- |
| Token crosses 50% progress with qualifying velocity | B3 King-of-the-Hill (10-minute window) |
| Token crosses 70% | A2 graduation (15 min) + A4 timing buckets; C1 creator-sells (SMALL) if the creator still holds |
| Token crosses 90% | A5 final-stretch express (5 min) |
| Graduation halts a Family A market | E2 crash market (20-minute window) auto-listed on the pool |
| Creator escrows a bond | C8 rug market (+ C7/C9 at creator's option) |
| Daily at 00:00 UTC | G1 over/under with the collector-set line; G4 when a challenger launchpad is within striking distance |
| Three or more trending tokens in the 60 to 85% band | F1 race market |

Every instance is parameterized from this catalog; anything outside it requires a catalog change first, which keeps settlement quality a reviewed property of the protocol rather than a per-market judgment call.
