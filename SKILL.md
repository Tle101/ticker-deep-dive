---
name: ticker-deep-dive
description: Run structured due diligence on a specific stock ticker using the Unusual Whales MCP. Use this skill whenever the user asks to "deep dive," "run DD," "quick pull," "analyze," "pull data on," "check the flow on," or "read the tape on" a ticker, or asks about options flow, sweeps, dealer gamma, GEX/VEX, dark pool levels, max pain, or whether a name is worth trading. Also trigger when the user names a ticker and asks "what's the read" or "is there a setup here." Quick mode is flow/tape-first; deep mode adds dealer positioning, dark pool, and catalysts. Hunt mode ("find me tickers," "scan the market," "what's moving," "anything with clean flow or heavy volume") screens the whole market for dive candidates. Synthesis treats "no edge" as a valid conclusion.
---

# Ticker Deep Dive (Unusual Whales DD)

Flow-first due diligence on a single ticker. Quick mode reads the tape: who is paying up, for what, with how much urgency. Deep mode layers dealer positioning (GEX/VEX), dark pool structure, and catalysts on top. Framework language is options flow + ICT/SMC price context, with GEX/VEX reserved for deep mode. Output is a decision-grade read, not a trade recommendation.

## Operating rules (non-negotiable)

1. **Token discipline.** Never carry raw API responses forward. After each phase, compress findings into ≤10 lines, keeping exact strikes, expiries, premium figures, and counts. Compress prose, never numbers.
2. **Call budget with an accuracy override.** Quick mode: ≤6 calls per ticker. Deep mode: ≤14. The budget exists to force compression and prevent context bloat — it is NOT permission to skip mandatory checks. The multileg reconstruction, thin-tape strike pull, and ITM blind-spot rules always outrank the budget: when one of them requires a call past the cap, take up to 2 contingency calls and name them in the output ("+1 over budget: resolving multileg sibling legs"). Exceeding budget for anything else (exploration, re-pulls, curiosity) means the read is unclear — say so instead of pulling more.
3. **Parallelize within phases.** Batch independent calls in a single turn. Never make sequential round trips for calls that don't depend on each other.
4. **Disconfirmation is mandatory.** Every synthesis contains ≥2 concrete disconfirming observations from data actually pulled.
5. **"No edge" is a first-class verdict.** Flow that is mixed, thin, or hedging-dominated = say so plainly. Do not manufacture a lean.
6. **Date hygiene.** Get current date/time before pulling anything.
7. **Multi-ticker requests:** quick mode runs on each ticker (batch identical call types across tickers in parallel). Deep mode is capped at ONE ticker per run — a multi-ticker deep dive request gets quick mode on all names plus the question "which one gets the full deep dive?" Three deep dives is ~40 calls of mostly-redundant context; one focused deep dive after quick triage is both faster and more accurate.

## Environment adaptation (verified 2026-07-06)

Detect capability, not platform: the question is whether a shell (jq/python) exists for large-payload processing. `get_flow_per_strike` has no server-side narrowing (ticker + date only) and returns 70–100K+ characters on active large caps — it WILL overflow or bloat context on busy names. Handle by environment:

- **Shell available (Claude Code):** oversized results get dumped to files. Before reading ANY file-dumped result, verify identity from the payload itself — check the `ticker` field with jq, never infer from call order (parallel calls return files in arbitrary order; misattribution invalidates the card). Extract top-N strikes by combined premium with jq; never cat a full dump into context.
- **No shell (claude.ai web/mobile/desktop app):** the full payload lands in context. Read it ONCE and immediately compress to: top ~8 strikes by combined call+put premium with ask/bid splits, any ITM concentration, and nothing else. Never scroll back into the raw payload — the compressed block is the record. When running 3+ tickers without a shell, sequence per ticker (snapshot → expiry → strike → compress) rather than batching all strike pulls at once, so no more than one giant payload is ever uncompressed in context.
- **Phone-first output always:** cards must survive a ~380px screen — short lines, no tables wider than 3 columns, no code blocks inside cards, levels and premiums inline in bold.

## Mode selection

- **Scan mode** (4+ tickers, watchlist sweeps): snapshot + tape baseline (2 calls/ticker; batch tape calls across tickers sharing a premium threshold), PLUS up to 1 flow-per-strike call per name whose tape is thin or ambiguous — the thin-tape rule is mandatory and its call is additive to the baseline, not carved out of it. Cards note the tier so the reader knows depth was traded for breadth.

  **Group-tape crowding (verified 2026-07-06 — mandatory).** A single multi-ticker `get_option_trades` call returns one shared, premium-sorted row set, so a name with heavy dollar flow buries a quieter name regardless of share price (observed: MU's $ 2B+ premium day filled all 30 rows with $ 2M–$ 21M prints, leaving LLY — whose largest print was $ 645K — with zero rows). The discriminator is per-print premium scale, not the stock price. Group tickers ONLY when their expected top-print premium sits within ~1 order of magnitude; otherwise split into separate tape calls or raise `limit` so the quietest name still surfaces (rule of thumb: `limit ≥ 10 × number_of_tickers`). Then verify EVERY grouped ticker returned ≥1 row — a zero-row name inside a group is a crowding artifact, NOT "no flow," and requires a solo re-pull. That re-pull is a mandatory-accuracy call, not budget-exempt exploration.

**ETF handling (GDX/GLD/SLV-type names, verified 2026-07-06):** ETF tape is routinely dominated by cross trades and multileg floor structures — expect most top prints to be correctly discarded as non-directional. That is normal ETF microstructure, NOT evidence institutions are absent; never output "no institutional participation" for an ETF from tape evidence alone. For ETFs, weight repetition campaigns (same contract hit repeatedly @ ask/bid) and strike/expiry aggregates above single-print size, and skip the earnings gate entirely.
- **Quick mode** (default; "quick pull," "check," "what's the flow," 1–3 tickers): flow and tape ONLY. No greeks, no gamma, no max pain. Phases 0, F1, F2.
- **Deep mode** ("deep dive," "full DD," multi-day swing consideration): Phases 0, F1, F2, F3 (week sweep), then D1–D3.
- **Confirm mode** ("confirm [TICKER]," "did that flow open," any next-day follow-up on a prior card): ≤2 calls. This is how UNCONFIRMED verdicts get resolved instead of dangling — see the Confirm section.
- **Hunt mode** ("find me tickers," "scan the market," "what's moving," no tickers named): market-wide screener funnel → coherence-scored SHORTLIST of dive candidates. Never verdicts, never levels. Three profiles: EARLY (pre-move discovery), LOUD (biggest flow extremes), CONFIRM (follow-through). Distinct from scan mode (scan = user names 4+ tickers; hunt = user names none). See the Hunt section.
- **Sector mode** ("strongest in semis," "scan software," "which mag 7 is leading," "how do the nuclears look" — a THEME named, not tickers): one basket-screener call → ranked leaderboard of the theme. Distinct from scan (scan = user's tickers, per-name cards) and hunt (hunt = whole market). See the Sector section.
- **Premium-buy mode** ("best naked call/put idea," "what should I buy calls on," "clearest option to buy," any request for a long-premium trade idea): runs a normal quick/deep read FIRST, then applies the naked-premium gate on top. The gate is a hard **market-regime filter** before anything else — see the Premium-buy section. This mode can and often should conclude "no card — stand down."

**Skill version: v14.** State mode AND version at the start of every output — e.g., "Mode: QUICK (v11)". The stamp is the only reliable way to tell which uploaded copy answered when the same skill lives in Claude Code, claude.ai, and mobile. Bump the number on every edit to this file.

**No stale-data shortcuts.** Every invocation of this skill — including a deep dive requested after an earlier quick pull or deep dive in the same conversation — re-runs every phase its mode requires, even if near-identical calls appear earlier in the session. Flow, tape, dealer positioning, and dark pool are the fast-moving layers this skill exists to read correctly; reusing them from even 20–30 minutes earlier defeats the purpose, especially in a deep dive where the person is relying on completeness. Do not skip a phase to save calls. If genuinely re-pulling would exceed the call budget, say so explicitly in the output rather than silently substituting old data.

**Post-close exception (verified 2026-07-06).** The no-stale-data rule guards against intraday drift. When the market is CLOSED and a same-day pull of the same endpoint/ticker already exists in this session, that data is final — reuse it, label it in the card ("post-close, reused from earlier pull"), and spend zero calls re-fetching it. Observed cost of ignoring this: the same ticker pulled 3× in one evening session, byte-identical every time. The exception applies only to same-day data after the close; premarket or intraday, the full rule stands.

## Phase 0 — Snapshot (1 call)

`get_ticker_ohlc_latest_or_date` with **limit=15** — same single call, fifteen daily rows (verified 2026-07-06/07: history is free; limit=1 throws the week away; rows 8–15 exist solely to make ATR-14 free). Row 1 (today) is dense: spot/OHLC, net_premium, bullish/bearish premium, call/put volume with ask/bid splits, iv_rank, implied_move, OI. Read ALL of it — net_premium vs price direction is the first divergence check, iv_rank feeds the IV requirement. Rows 2–7 give the WEEK: read the daily net_premium sequence against the closes. A one-day divergence and a regime flip look identical in a single row (observed: +$ 109M on a red day read as "⚠️ divergence" alone, but the 7-row view showed −$ 266M/−$ 123M crash days immediately before — first positive net after capitulation, a far stronger signal). Flag: streaks (3+ same-sign days), sign flips after large moves, and IV-rank trajectory. `search_tickers` only if the symbol is ambiguous.

**Earnings date sourcing (verified 2026-07-06):** the snapshot payload does NOT contain next_earnings_date. It appears only in tape print payloads (`get_option_trades` rows). Read it from the first tape pull. If tape comes back empty and the earnings gate matters to the verdict, the card must declare the blind spot: "ER date unavailable from this pull." ETFs have no earnings; skip the gate.

**API clamps:** `min_premium` floors at $ 25,000 and `min_size` at 150 server-side. Thresholds below these are silently raised — never assume a sub-$ 25K pass ran.

## Move calibration (zero extra calls — computed from the Phase 0 rows; added v12)

Every directional card carries a MOVES line built from the snapshot alone:

- **ATR-14** (avg true range over the 15 OHLC rows) = expected intraday RANGE. Job: **stop hygiene** — a stop closer than ~1 ATR is noise-stoppable; the card must say so and suggest the level outside it.
- **Weekly implied move** = `implied_move_30` × √(7/30) (daily = ×√(1/30)) = expected close-to-close displacement. Job: **target feasibility** — a target outside the weekly cone is a multi-week thesis; label it.
- **ATR ≈ 2× implied-daily is NORMAL** (range vs displacement; verified 2026-07-07 on LLY/PLTR/SNOW, ratio 2.0–2.2× on all three). Never read the gap as mispricing.
- **Regime flag:** when trailing close-to-close moves run >~1.3× implied-daily, add "realized > implied — cone conservative this week."
- **Straddle upgrade (1 call, ONLY when the name moved >3–4% today or IVR jumped):** `get_chains_for_expiry` at the nearest Friday, read ATM ±2 strikes. The payload has NO bid/ask (theo/last only) and overflows (~80K chars) — jq-extract the ATM slice, never read whole. √t scaling understates on rip days: term structure inverts (verified NET 2026-07-07: 3.1%/day near vs 2.8%/day at 10d — straddle cone ~25% wider than scaled).
- Interpretation discipline: the cone is a seller-priced ~68–80% boundary, not a forecast (haircut ~15% for a median-expected move). Log each cone in the ledger and grade containment at Friday's close — the running hit-rate calibrates the whole feature.

Card line format: `Moves: ATR $X/d (range) · ±$Y/wk implied (closes) — [target vs cone / stop vs ATR, one judgment]`

## Phase F1 — Aggregated flow (2 calls, batched)

1. `get_flow_per_strike` — call vs put premium by strike, and critically **ask-side vs bid-side premium** at each strike.
2. `get_flow_per_expiry` — near-dated vs far-dated concentration, OTM premium share.

Compress into: net directional premium, the 2–3 strikes absorbing the most premium, which side of the market is the aggressor, and the expiry profile (event bet vs positioning).

## Phase F2 — Tape (1–2 calls)

`get_option_trades` filtered to the ticker: `intraday_only`, `min_premium` set by market-cap tier — do not eyeball it (observed failure: a $ 300K threshold on an $ 8B name produced a false "no institutional flow"):
- mega >$ 200B → $ 500K
- large $ 50B–200B → $ 250K
- mid $ 10B–50B → $ 100K
- small <$ 10B → $ 50K (or the $ 25K API floor for micro/illiquid names)

Read `marketcap` from the first tape row to verify the tier guess; an empty first pass at the wrong tier means re-run at the correct tier before concluding thin tape (that re-run is a correction, not budget-exempt exploration — it replaces the miscalibrated call). `is_otm: true` for the first pass, exclude deep ITM. If the first pass shows a cluster, one follow-up call zoomed to that contract/expiry (e.g., `report_flag: ["sweep"]` or `opening: true`) is allowed within budget.

## Phase F3 — Week sweep (deep mode only, 1 call)

`get_option_trades` with `newer_than` = 7 calendar days back, `min_premium` ≈ 4× the ticker's tier threshold, order by premium desc, limit 20. This is a **persistence detector, not a direction reader** — multileg reconstruction degrades across days (sibling legs won't appear in a filtered pull), so never flip a verdict on a week-old print's tag alone. Read it for:

- **Cross-day campaigns:** same contract hit @ ask across multiple sessions — the highest-conviction repetition signal, invisible to any single-day pull.
- **Precedent:** do today's flagged prints have earlier tranches? Day-4 of accumulation ≠ a day-1 punt.
- **Who owns the week:** identify the largest package of the week and mark its current P&L vs spot. When it dwarfs today's flow, the card MUST name it (observed: a $ 904K day-of bull campaign vs a still-held $ 25M synthetic short opened 4 days earlier at the lows — the card was directionally incomplete without the second fact).
- **Held vs closed:** cross-check heavy week prints against current OI — OI ≈ print volume means still held; OI collapsed means closed/covered.

## How to read the tape (expert discipline — apply every time)

**Aggressor matrix.** Direction comes from side × type, not volume:
- Calls at ask = bullish demand. Calls at bid = call selling (bearish/neutral, or covered-call supply).
- Puts at ask = bearish demand or hedging. Puts at bid = put selling = bullish (someone wants the shares or the premium).
- Mid-priced prints carry low information; discount them.

**Premium-weight everything.** One $ 2M block outweighs 500 lots of $ 0.30 weeklies. Rank by dollars, not contracts.

**Opening vs churn.** Volume > existing OI = likely opening interest — that's real new positioning. Volume < OI is ambiguous (could be closing). Label today's read UNCONFIRMED until next morning's OI update, and say so in the synthesis.

**Multi-leg contamination.** Spread legs print on both sides and destroy naive ask/bid reads. Check `is_multi_leg`; read single-leg prints for direction, treat multi-leg premium as structure-dependent (a big put spread bought is bearish but risk-defined — different conviction than naked puts).

**Urgency tells.** Sweeps across exchanges = urgency, paying up now. Single-exchange blocks = negotiated, patient, often institutional. Floor prints = neither urgent nor informative about direction by default.

**Repetition = campaign.** The same contract hit repeatedly on the ask across the session is accumulation. One print, however large, is a data point; five prints is intent.

**Expiry coherence.** Heavy near-dated OTM = event speculation (check what's on the calendar before calling it conviction). Far-dated ATM/ITM size = positioning with patience. A read where strikes, expiries, and urgency all agree is high-conviction; scattered premium is noise.

**Hedge detection.** Large put buying alongside no matching bearish tape elsewhere may be portfolio hedging, not a directional bet. Say "bearish or hedging" when you cannot distinguish — do not upgrade ambiguity into signal.

## Phase D1 — Dealer positioning (deep mode only, 3–4 calls, batched)

`get_greek_exposure_by_ticker` (read gamma AND vanna/charm — the VEX side), `get_greek_exposure_by_strike` (walls, flip level), `get_max_pain`, optionally `get_greek_exposure_by_expiry` for roll-off risk. Compress into regime, wall strikes, flip, max pain vs spot. Note explicitly where flow agrees or fights the dealer map — flow fighting a strong wall usually loses until the wall rolls off.

## Phase D2 — Dark pool (deep mode only, 1–2 calls)

`get_dark_pool_volume_price_group` for institutional reference levels; optionally `get_dark_pool_trades` for notable single prints. 2–3 levels, above/below spot.

## Phase D3 — Catalysts (deep mode only, up to 3 calls, only what bears on the thesis)

Upcoming earnings/expected move, short data, insider transactions, market events inside the horizon. Do not pull reflexively.

## Confirm mode (next-day OI check, ≤2 calls)

Purpose: resolve a prior card's *"unconfirmed until tomorrow's OI update"* tag. Volume without an OI print the next morning is churn, not positioning.

1. Requires a prior card in the conversation (or the user naming the contracts). If neither exists, say so and offer quick mode instead.
2. Calls: `get_open_interest_changes` for the ticker (1 call). Optionally 1 snapshot for spot vs the card's invalidation level. Nothing else.
3. For EACH contract flagged in the prior card: did its OI rise by roughly the flagged print's size?
   - **OPENED** (OI up ~size or more) → the positioning is real; upgrade the verdict's conviction one notch and say so.
   - **CLOSED/CHURN** (OI flat or down) → the print was closing or day-traded; downgrade or kill the verdict.
   - **PARTIAL** → say which fraction opened; hold conviction.
4. Output is a 3–5 line delta card, not a full re-read: per-contract OPENED/CLOSED/PARTIAL, spot vs invalidation, one-line updated verdict per the reconciliation rule. Confirm mode never re-runs flow phases — if the user wants a fresh read, that is a new quick/deep run.
5. **Package reconstruction for free (verified 2026-07-07):** the OI-changes payload's `volume_candles` double as a spread detector — same-timestamp, same-size volume spikes across two contracts of the same underlying are one package (observed: a 7,500× 170/180 call credit spread reconstructed from paired 14:10 candles, no tape re-pull needed). Check for sibling spikes before reading any single confirmed contract directionally.

## Hunt mode (market scan — find dive candidates, ≤7 calls)

Hunt produces a SHORTLIST, never verdicts — no levels, no conviction, no directional card. Every tag is provisional until quick/deep mode runs on the name. Budget: ≤7 calls (typically 2–4 per profile).

**Early-session gate (verified 2026-07-07):** in the first ~30 min of RTH, premium/skew are immature fractions of a day and the UNUSUAL ratio screen is structurally EMPTY (no name can reach 2× its own 30-day options volume that early — an empty result is a clock artifact, not a finding). Before ~10:00 ET, either run hunt on prior-close final data and flag only intraday deltas, or stamp the output "EARLY-SESSION — metrics immature" and skip the ratio screen entirely.

**H1 — Pick a profile by intent (all `get_stock_screener`; API quirks verified 2026-07-07):**

**API unit/sort traps (cost a full empty run when missed):** `min_change`/`max_change` are FRACTIONS — "0.005" = +0.5%; passing "0.5" means +50% and returns nothing (observed). `stock_volume_vs_avg30_volume` is a FILTER only, not a sort key on this API. `net_premium` = net_call_premium − net_put_premium (verified against per-ticker cards).

**EARLY profile (default in the morning) — "interest is starting, move not obvious yet." 2 calls, bull + bear mirror:**
- BULL: `min_stock_volume "500000"`, `min_stock_volume_vs_avg30_volume "1.2"`, `min_premium "250000"`, `min_net_premium "100000"`, `min_bullish_perc "0.55"`, `max_put_call_ratio "0.9"`, `min_change "0.005"`, `min_underlying_price "10"`, `min_earnings_dte "7"`, `issue_types ["Common Stock"]`, `hide_index_etf true`, order `net_premium` desc
- BEAR: mirror — `min_net_put_premium "100000"`, `min_bearish_perc "0.55"`, `min_put_call_ratio "1.1"`, `max_change "-0.005"`, order `net_put_premium` desc
- Variant: drop min/max_change to also catch contrarian dip/rip accumulation (observed 2026-07-06: GEO + CXW — same industry, both near 52wk highs, P/C 0.04/0.01 call accumulation on red days — a sector cluster no loud scan surfaces).
- EARLY has NO mcap floor by design; flag anything <$1B as thin-tape-weak (flow reads degrade per the thin-tape rule).
- **MOMENTUM retag (added v12):** cross-check the 5-day price run before tagging EARLY — a name already up >25% in ~5 sessions is mid-move, not pre-move; tag **MOMENTUM** instead (observed: TENB tagged "EARLY" on day ~5 of a +50% news run — the tag promised discovery the data didn't support).

**LOUD profile — "what's the biggest flow today." 3 calls:**
- BULL tail: `order=net_premium` desc · BEAR tail: same asc · UNUSUAL: `order=volume` desc with `min_ratio_30_day_total "2"` (options volume ≥2× the name's OWN 30-day norm)
- Shared: `min_marketcap "2000000000"`, `min_premium "5000000"` ("2000000" on UNUSUAL), `hide_index_etf true`, `issue_types ["Common Stock","ADR"]`
- NEVER rank by absolute volume without the ratio filter — it returns the same megacaps every day (observed: ratio scan surfaced ORLY at 9.4× and UNM at 4.8× normal call volume — names no absolute sort ever shows). During earnings season the raw tails are mostly pre-ER positioning (observed: TSLA/META/AMZN/NFLX all inside ~3 weeks of ER).

**CONFIRM profile (late-day / next-day) — "the move is proving itself." 2 calls, bull + bear mirror of EARLY with tightened floors:**
- `min_stock_volume "1000000"`, `min_stock_volume_vs_avg30_volume "1.8"`, `min_premium "750000"`, net premium floor "300000", skew floor "0.65", P/C ≤0.7 (bull) / ≥1.3 (bear), `min_change "0.02"` / `max_change "-0.02"`
- CONFIRM names arrive with pumped IV by construction — any dive on them MUST weigh the IV requirement before structure.

**PROMOTION LADDER (requires the ledger — persistence IS the signal).** Ledger convention: keep `huntlog.md` in the project scratchpad/working directory; every hunt run appends its shortlist + tags, every dive appends its verdict, and the weekly (Friday) wrap-up grades two things — tag direction vs actual outcome, and weekly-cone containment (did the close land inside the implied cone logged earlier in the week?). An EARLY name that reappears on a later run with (a) rising stock-vol-vs-avg30, (b) expanding net premium, (c) skew strengthening not weakening, and (d) price confirming the direction, graduates toward CONFIRM → that graduate is the priority dive candidate. One appearance = watch; repeat appearance with improving readings = act.

**Two operating rhythms — pick by when the user is working:**
- **OVERNIGHT (primary — after-hours DD to build next-day focus names):** one evening pass on final session data: EARLY bull+bear + LOUD (≤5 calls) → H2 + confluence → diff the ledger for persistence → deep dive the 1–2 graduates → output a next-day focus card (direction, levels, invalidation, "unconfirmed until OI"). H2c runs RETROSPECTIVE: window the tape to the session's final 60–90 minutes — flow clustering into the bell is a next-day follow-through tell; all-day dribble is not. Post-close data is final and deterministic (verified 2026-07-07), so evening runs are drift-free and same-day re-pulls are forbidden per the post-close rule. **THEN, next morning, ≤2 calls: Confirm mode on each dived focus name — did the flagged volume become new OI overnight?** OPENED → the focus name stands, conviction up one notch. CHURN → strike it from the day's list before risking anything. The overnight rhythm is not complete without the morning OI check; a focus card that never gets confirmed is a dangling verdict.
- **INTRADAY (occasional):** morning EARLY → midday recheck (cut skew-faders — a morning burst that fades was distribution) → live H2c on finalists → late-day CONFIRM for names proving out.

**SPLIT/CORPORATE-ACTION DETECTOR (mandatory before any tag, added after CRWD 2026-07-07):** options-volume ratio spiking while stock relative_volume is normal · total OI multiplying >2× in ≤2 sessions · fractional/eighth-dollar strikes in the chain · implied_move dollar vs pct implying a different underlying than the close. Any two present → tag SPLIT-ARTIFACT, exclude from CLEAN/SPIKE, and any dive must use post-action sessions only (observed: CRWD 4:1 split inflated its ratios to 2.8×/1.8× and corrupted OI, greeks history, and max pain).

**H2 — Coherence scoring (0 extra calls — every input is already in the H1 payload).** A name is CLEAN only if ALL pass:
1. **Skew:** bullish_premium/(bullish+bearish) ≥ 0.60 for bulls (≤ 0.40 for bears). This kills the #1 failure mode: a huge net that is really 53/47 mush (observed: MU topped the bull sort at +$109M net while running $1.07B bull vs $0.96B bear — mush, not signal; same for TSLA, AMD, SNDK that day).
2. **Aggressor coherence:** bulls need call ask-side volume > call bid-side (bears: put ask > put bid, or dominant call bid-side selling). Net sign and aggressor side must agree.
3. **Participation:** call_volume (put_volume for bears) ≥ ~1.5× its avg_30_day equivalent — a campaign, not one print dressed as a trend.
4. **ER gate:** next_earnings_date >14 days out, else tag EVENT — scan-level data cannot distinguish conviction from hedging across a binary.
5. Floors already enforced by the H1 filters (mcap ≥$2B, real premium).

Tags: **CLEAN-BULL** · **CLEAN-BEAR** · **VOLUME-SPIKE** (participation elevated, direction mixed — often the best dive candidates) · **EVENT** (ER <14d) · **MUSH** (big net, failed skew — display once as a warning, never dive-worthy on this evidence) · **MOMENTUM** (>25% 5-day run — mid-move, not discovery) · **CALL-SELLER** (large negative net_call_premium that fails bear skew — an early-warning tag, watch not dive; observed: INTC flagged at −$90M call selling/44% skew, matured into CLEAN-BEAR at −7% two sessions later) · **SPLIT-ARTIFACT** (corporate-action contamination, excluded).

**CONFLUENCE SCORE (0–7) — attach to every shortlisted name; ≥5 = dive-worthy.** Tags say WHAT a name is; the score says HOW MUCH agrees: (1) stock volume above its own 30d norm · (2) options premium above its own 30d norm · (3) net premium directional · (4) skew aligned with the net's direction · (5) clustered or repeated flow (same-session clustering via H2c, or multi-run persistence from the ledger) · (6) no ER/split/arbitrage distortion · (7) price confirming the direction — judged **open-to-close AND close-vs-prior**, never close-only (observed 2026-07-06: WULF "+4.9%" close/close masked a −8% open-to-close failed-gap reversal; the reversal WAS confirmation of the bear read, and close-only nearly mis-scored it). One loud metric with a 2/7 is noise by definition; a quiet name at 6/7 outranks a loud name at 3/7.

**H2c — Urgency check (1 call, market hours only, finalists only).** The MCP exposes no interval-flow endpoint — approximate it with one time-windowed tape pull: `get_option_trades` on the finalist, `newer_than` ≈ 45 minutes ago, `intraday_only`, tier premium threshold. Ask-side concentration + low multileg ratio + size clustered inside the window = urgency (upgrades confluence #5); the same premium dribbled across the whole session = downgrade. Urgency distinguishes "someone needs this position NOW" from "someone is patiently averaging" — both matter, but they are different trades.

**Junk to strike before scoring:** deep-ITM call prints near a `next_dividend_date` (dividend-capture, non-directional) · arbitrage/cross-tagged prints · big options premium with WEAK stock participation (relative_volume well under ~0.9 while options ratios scream — roll/split suspicion, run the detector) · names whose stats deteriorate on the midday recheck (skew fading intraday = the morning burst was distribution or noise, not a campaign; screener names can drop out of filters as later trades dilute the averages — a name that survives the whole session is itself a signal).

**H2b — Optional print check (1–2 calls).** Only when the user will act on the hunt itself rather than dive next: one `get_option_trades` pull covering the top 2–3 survivors (obey the group-tape crowding rule — group only names within ~1 order of magnitude of expected top-print premium). A candidate whose direction shows no clean single-leg ask/bid print at its tier threshold downgrades to AGGREGATE-ONLY. Skipping H2b is correct when a dive follows anyway — the dive re-runs tape properly, and hunt tags must never masquerade as tape reads.

**Output card (phone-first, max 8 rows):**
`TICKER $spot ±% · net $X · skew B% · vol ×N vs 30d · IVR · ER-days · TAG`
End with ONE line naming the 1–2 tickers that earn the dive and why — then stop. Hunt picks where the next 6–14 calls go; it never replaces them.

## Sector mode (theme leaderboard, 1–3 calls; added v13)

**Trigger:** the user names a THEME, not tickers — "strongest in semis," "scan the mags," "how's memory looking," "best software name." Driver: a Mag-7 "which is ready to run" request cost 7 Phase-0 calls (one per name); one basket-screener call reproduced ~90% of the ranking signal (verified 2026-07-07 on semis + memory baskets — the sector leaderboard independently matched a full day of per-name dives: same leader, same laggard, same divergence names).

**S1 — Resolve the theme (0 calls).** Resolver table below. Unknown theme → best-guess a basket from knowledge, NAME the proxy used in the output, and invite correction ("scan shipping: used X/Y/Z"). User-supplied tickers always override.

| Theme | Basket (curated, dated 2026-07) |
|---|---|
| mag7 / mags | AAPL MSFT GOOGL AMZN NVDA META TSLA |
| semis | NVDA AMD AVGO TSM INTC QCOM TXN AMAT LRCX KLAC MRVL ADI ASML ARM |
| semicap | AMAT LRCX KLAC ASML |
| memory/storage | MU WDC STX SNDK |
| networking/optics | ANET CRDO ALAB COHR LITE MRVL |
| software | MSFT ORCL CRM NOW SNOW MDB DDOG NET PLTR |
| cybersecurity | CRWD PANW ZS NET S FTNT |
| AI-infra/neocloud | CRWV NBIS IREN APLD WULF VRT SMCI DELL |
| quantum | IONQ RGTI QBTS QUBT |
| nuclear | OKLO SMR LEU CCJ VST CEG NLR |
| space | RKLB ASTS LUNR RDW |
| defense | LMT RTX NOC GD KTOS AVAV |
| crypto-linked | COIN MSTR HOOD MARA RIOT WULF IREN |
| EV/autonomy | TSLA RIVN LCID UBER |
| fintech/brokers | HOOD SOFI COIN SQ PYPL |
| GLP-1/pharma | LLY NVO AMGN VKTX |
| energy | XOM CVX COP SLB OXY (or sectors=["Energy"]) |
| financials | JPM GS MS BAC WFC C (or sectors=["Financial Services"]) |
| china tech | BABA PDD JD BIDU |
| big-cap GICS sectors | use the screener `sectors` enum directly |

**Endpoint trap (verified 2026-07-07):** the screener's `etfs` constituent param returned EMPTY for SMH — do NOT rely on it. Curated `ticker` comma-lists are the primary path; `sectors` enum works for the 11 GICS sectors only. Baskets go stale — the table is dated; membership edits are normal maintenance, not version bumps.

**S2 — The one ranking call.** `get_stock_screener` with `ticker="A,B,C,..."`, `order=perc_change desc`, limit ≥ basket size. Every ranking metric arrives in one shot: perc_change, net call/put premium, bullish/bearish premium, IVR, put/call ratio, relative_volume, next_earnings_date.

**S3 — Rank (0 calls).** Score each name on 4 pillars: **price leading** (day % vs basket median) · **flow confirming** (net-premium sign agrees with price; bull/bear premium split ≥55% helps) · **participation** (relative_volume ≥ ~1) · **clean** (ER >14d, no split/distortion flags). LEADER = strongest on ≥3. Also mark: **FLOW-DIVERGENT** (price and flow disagree — e.g., −7% day with the basket's biggest net call buying = dip accumulation; price up with put-heavy flow = fade risk) and the **LAGGARD** (weakest price + flow agreeing down = the short-side candidate). Never rank on price alone — the divergence column is half the value (observed: a hunt's "MRVL call accumulation" read lacked the −7.4% price context; the sector table caught it in the same call).

**S4 — Regime + output (phone-first).** Note the basket's aggregate tilt (all-red/all-green = sector trade, not stock-picking) and the IVR regime (basket IVR ≥~85 = sell-premium regime; flag that long options overpay). Leaderboard card, max ~8 rows:
`TICKER $spot ±% · net $X · skew · IVR · ER-days · tag(LEADER/CONFIRMED/FLOW-DIVERGENT/LAGGARD/EVENT-GATED)`
End with ONE line: which name earns the dive and why. **Sector mode ranks; it never issues verdicts or levels** — screener rows are same-day snapshots with no trend/ATR history, so the MOVES line and any trade framing require the leader's own Phase-0 pull (that's the dive).

**S5 — Optional drill (1–2 calls):** Phase-0 snapshot on the leader (adds trend + ATR/cone), or `get_market_sector_etfs` for sector-level context when the ask is macro ("which sectors are strong" with no theme).

## Premium-buy mode (naked calls/puts — the trade-idea gate; added v14)

For requests that want a specific long-premium trade ("best naked call/put," "what do I buy calls on"). This mode exists because naked premium has three ways to lose — direction, time, vol — and a bullish-and-right read still loses money most ways. It was built from a 4-regime blind historical backtest (2026-07; de-risk, chop, downtrend, rally/vol-crush windows, ~180 candidates). **The backtest overturned the intuitive rules and produced one dominant finding, so this mode is deliberately spare.**

**THE FINDING (why this mode is mostly a filter, not a generator):** across all four regimes, **market-direction alignment dwarfed every flow pillar.** In the rally window trend-aligned bulls went 4/4 while bears went 1/8; in the downtrend window bears won and bulls buying bounces went 2/7; in chop nothing directional worked. ~90% of ALL losses in every regime were **directional reversal** — theta-bleed and vega-crush losses were both rare (even in the window built to punish vega, only 1 of 12 cards was crushed, and it outran the crush). **The risk in naked premium on a ~1-2 week hold is being wrong on direction, not decay and not IV.**

**STEP 0 — REGIME GATE (do this FIRST; it decides whether a card can exist at all).**
Read the tape's direction before screening anything: SPY/QQQ price vs its 10- and 20-day trend, plus the sector-ETF breadth sweep (Sector-mode macro variant).
- **UPTREND** (index above rising 10/20d, breadth green) → only **CALL** cards allowed. Suppress put ideas.
- **DOWNTREND** (index below falling 10/20d, breadth red) → only **PUT** cards allowed. Suppress call ideas.
- **CHOP** (index net-flat over ~10d with normal daily wiggle) → **STAND DOWN. Emit no directional card.** Chop lost money in every configuration tested; the correct output is "no trade — chop tape, premium buying has no edge here, wait for a trend or trade a defined-risk spread instead."
Counter-trend cards are the single largest loss source in the data — the gate's whole job is to refuse them.

**STEP 1 — Candidate: trend-aligned flow only.** In an uptrend, take the strongest bullish name from a quick/hunt/sector read (net call premium + bull skew + price confirming); downtrend, the strongest bearish. The name must agree with BOTH the flow AND the regime. A bullish-flow name in a downtrend is not a call candidate — it's a bounce to fade or ignore.

**STEP 2 — NAKED-SCORE checks (regime-scoped, not a rigid gate):**
1. **Trend-aligned** (mandatory — from Step 0). No alignment → no card.
2. **Strike inside the weekly cone** (v12 MOVES line): target ≤ the cone edge, strike ≤ ~0.5 cone OTM. Strikes outside the cone need a multi-week expiry, not a weekly. (This pillar held up as sane across all regimes.)
3. **Earnings clear** — no ER inside the intended hold (still a real binary; keep it).
4. **Anti-chase is REGIME-CONDITIONAL** — the backtest's sharpest correction. In CHOP/early/weak trends, do NOT chase (fail if the prior 3-day move ≥ 0.75 cone, and skip if the entry gaps ≥0.25 cone further your way). In a STRONG confirmed trend, anti-chase INVERTS — continuation wins, so do not veto a trend-aligned entry just because it ran. (Chasers won 56% and 45% in the two trending windows, lost in chop.)
5. **IVR is DISCLOSURE, not a veto.** State IVR and the breakeven in implied-days on the card; do not reject on high IV. Elevated IV rarely crushed the buyers in testing — it usually means "priced to move," and the move outran the vega. One caution to print: the IVR 60–90 band was the worst-performing cohort in 3 of 4 windows (it tends to flag extended, about-to-revert names), so on a 60–90 name lean on the trend/cone checks harder. IVR ≥90 in a real mover is fine.

**STEP 3 — The PREMIUM-BUY card** (only if Step 0 passed and it's trend-aligned):
`BUY [strike/expiry] · regime: UPTREND/DOWNTREND · trend-aligned ✓ · entry (next open) · stop (invalidation, ≥1 ATR away) · target (≤ cone edge) · breakeven: N implied-days · IVR X (disclosure) · what kills it: [the reversal level]`
Every card names the **reversal level** as the kill — because reversal, not decay, is the risk. Grade/quote fills at the NEXT OPEN, never the signal close (overnight gaps averaged ~0.2 cone and halve the edge).

**When to emit nothing:** chop tape · no trend-aligned name with clean flow · only counter-trend ideas available · strike only reachable outside the cone with a weekly. "No card, here's why" is the correct and common answer — this mode refuses far more than it emits. Status: framework validated on historical replay (4 regimes); treat live cards as tracked hypotheses until the forward test logs real option P&L, not proven edge.

## Synthesis (no calls)

Quick mode output is a **report card per ticker — tight bullets, phone-readable:**

**TICKER** $spot (±day%) — **VERDICT** (NO EDGE / BULLISH / BEARISH / WAIT, + conviction)
- Net: $X (bull $Y / bear $Z) — append **⚠️** when the net direction disagrees with the largest prints
- Wk: the 7-day net-premium shape in one compact line (e.g., "wk: +267M → −266M → −123M → **+109M** — first positive since the flush") — from the snapshot rows, no extra call. Name streaks, flips, and whether today's net confirms or breaks the week's pattern
- Prints: the top 1–3 trades by premium, each as `$prem · contract · side` (e.g., "$ 3.4M · Dec-27 125c · bought @ ask"). Aggressor side is mandatory — "bought/sold @ ask/bid," never just the tag. If nothing clears the threshold, say so: "no prints >$ 150K" is the finding.
- IVR + next ER date (with days-out when inside ~45d)
- Moves (directional cards only): ATR-14 + weekly implied cone with the one-line target/stop judgment (see Move calibration)
- → Expert read, one line: what the tape says AND the level that kills it. E.g., "Whales structurally long; bearish net is small-lot noise. Read dies below 138."

**Print supremacy rule.** The VERDICT derives from the largest prints — identifiable actors with size — never from net premium alone. Net premium is a screening stat: display it, but when net and prints diverge (they did on ORCL: -$ 15M net vs $ 5.5M of bullish structural prints), the prints win and the ⚠️ must appear. A verdict justified only by the net aggregate is an invalid card.

**Print hygiene (apply before anything else):** (1) Discard any print with `canceled: true` — it never stood. (2) Group prints of the same contract executed in the same second across different exchanges: that is ONE order fragmented by a sweep, and UW's per-venue tags will conflict (the same IREN order was tagged bullish, bearish, and neutral across four venues). Sum the fragments, read the side from fill price vs NBBO on the aggregate, never from any single fragment's tag.

**Multileg reconstruction (mandatory before any print enters the verdict).** For every top print, check the payload fields: `multi_vol > 0`, `stock_multi_vol > 0`, or condition codes `mlft`/`mlat`/`tlft`/`slft` mean the print is one leg of a package and MUST NOT be read standalone — UW's bullish/bearish tag on a single leg is meaningless for a spread. Reconstruct the package: search the same tape pull for prints with the same timestamp (to the second) and same/similar size in other contracts of the same underlying. Read the PACKAGE (net debit/credit, net delta, which leg is the anchor), not the leg. If the sibling legs can't be found in the pull, mark the print `AMBIGUOUS (multileg, legs unresolved)` and exclude it from the verdict — a $ 400K leg of an unknown structure is not evidence of anything. Cross trades (`no_side`, cross_trade flag) are pre-negotiated and non-directional: display, never count toward a lean.

End multi-ticker pulls with one shared line: *"Unconfirmed until tomorrow's OI update"* when applicable. Nothing else — no methodology recap, no per-ticker caveats already covered by the tag.

## Post-run audit (the improvement loop)

After delivering the report, silently verify four checks; surface only violations:
1. **Print supremacy** — does every verdict trace to specific prints (or their absence), not the net aggregate?
2. **Multileg** — was every top print checked for multi_vol/condition codes, and packages read as packages?
3. **Earnings gate** — was next_earnings_date checked against the dominant flow's expiry and the hold window?
4. **Format** — card structure followed; aggressor side stated on every print; invalidation level present on every directional lean.

If a check fails, state the violation in one line at the end of the output and what the corrected read is. Skill edits happen ONLY from logged violations or from verdicts that conflicted with real outcomes — never from formatting preference. One observed failure = one targeted patch.

Deep mode adds below the card: dealer map, merged key-levels ladder (walls + dark pool + max pain), catalyst dates, whether flow confirms or fights positioning, and a fuller "what kills this thesis" (≥2 disconfirming facts) since deep mode informs sizing decisions.

**Verdict discipline** — *directional lean* requires validating AND invalidating levels; *wait* requires the specific condition that changes the read; *no edge* stands alone. Never position sizing.

**Earnings gate.** Before assigning conviction to any flow signal, check the next earnings date (it's in the snapshot payload as next_earnings_date). If the dominant flow sits in an expiry that spans that date, cap conviction at MODERATE and label it "directional or event hedge" — put buying across a binary with a history of double-digit reactions is what hedgers do, not just bears. Only flow in expiries that resolve BEFORE earnings, or flow with urgency signatures (sweeps, repetition) that hedging doesn't produce, can carry HIGH conviction near a report date.

**Verdict reconciliation.** If a prior verdict on this ticker exists earlier in the conversation, the new card must state in one line what changed and why (new data, refined read, or error in the prior call). Silent revisions erode trust in every future card.

**IV requirement (directional leans only).** The Phase 0 snapshot already returns iv_rank and implied_move — no extra call needed. Every directional-lean card MUST include iv_rank in its evidence line (e.g., "IVR 73 — puts are expensive"). Rich IV changes the structure implication of the lean; crushed IV changes it the other way. A lean without IV context is an incomplete card.

## Failure handling

- Errored/empty calls: note once, adjust confidence, move on. Never fabricate.
- **DATA GAP vs thin tape (verified 2026-07-06, two occurrences).** If the snapshot's options fields return null/zero on a name with real OI and average volume, AND the strike pull returns an empty array, AND the tape is empty — that is a missing feed, not absent flow. Procedure: retry the pulls once; if byte-identical, spend 1 call on `get_trading_states` to rule out a halt; then output **NO READ — DATA GAP** with price-only observations. NEVER output "no institutional flow" from a data gap — thin tape is a market finding, a data gap is a plumbing finding, and conflating them fabricates a signal. Re-pull next session; if the gap persists, tell the user to check the UW connector.
- Permission/auth errors ("no approval," credential failures): terminal — do NOT retry the same call. Name the resulting blind spot once in the output (e.g., "no short-interest read available") and continue. Suggest the user check connector permissions only if the gap was material to the verdict.
- Thin tape is a finding, not a failure: if the first tape pass returns fewer than ~5 prints at the premium threshold, that IS the complete institutional BLOCK picture — but before reporting "institutions are not participating," pull flow-per-strike (mandatory, even in scan mode). When reading the strike pull, scan ITM strikes explicitly, not just OTM: the tape pass runs `is_otm: true` and is structurally blind to deep-ITM call accumulation (stock replacement), which is among the highest-conviction flow that exists (observed: OSCR's dominant flow was ~$ 950K of ITM call buying invisible to the OTM tape). Any card whose tape was OTM-filtered and where no strike pull ran must name the ITM blind spot in one line. Flow arrives in two styles: blocks (visible on tape) and accumulated clips below print thresholds (visible only in strike aggregates). A name with an empty tape can still carry millions in strike-concentrated positioning (observed: $ 5.3M at a single APP strike with zero prints >$ 150K). Only after both views are thin may the card claim no institutional flow. Never lower the tape threshold to manufacture signal.
- Stale timestamps: flag in synthesis.
- Thin options tape (low premium everywhere): flow read has weak power on this name — say so and downgrade to price structure only.
- Quick-mode caveat to include once when relevant: flow reads carry positioning-blindness — a strong flow signal can still be running into a dealer wall the quick pull cannot see. If the flow read is HIGH conviction and the user is sizing up, recommend the deep-mode positioning check before entry.
