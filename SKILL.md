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

**Skill version: v11.** State mode AND version at the start of every output — e.g., "Mode: QUICK (v11)". The stamp is the only reliable way to tell which uploaded copy answered when the same skill lives in Claude Code, claude.ai, and mobile. Bump the number on every edit to this file.

**No stale-data shortcuts.** Every invocation of this skill — including a deep dive requested after an earlier quick pull or deep dive in the same conversation — re-runs every phase its mode requires, even if near-identical calls appear earlier in the session. Flow, tape, dealer positioning, and dark pool are the fast-moving layers this skill exists to read correctly; reusing them from even 20–30 minutes earlier defeats the purpose, especially in a deep dive where the person is relying on completeness. Do not skip a phase to save calls. If genuinely re-pulling would exceed the call budget, say so explicitly in the output rather than silently substituting old data.

**Post-close exception (verified 2026-07-06).** The no-stale-data rule guards against intraday drift. When the market is CLOSED and a same-day pull of the same endpoint/ticker already exists in this session, that data is final — reuse it, label it in the card ("post-close, reused from earlier pull"), and spend zero calls re-fetching it. Observed cost of ignoring this: the same ticker pulled 3× in one evening session, byte-identical every time. The exception applies only to same-day data after the close; premarket or intraday, the full rule stands.

## Phase 0 — Snapshot (1 call)

`get_ticker_ohlc_latest_or_date` with **limit=7** — same single call, seven daily rows (verified 2026-07-06: history is free; limit=1 throws the week away). Row 1 (today) is dense: spot/OHLC, net_premium, bullish/bearish premium, call/put volume with ask/bid splits, iv_rank, implied_move, OI. Read ALL of it — net_premium vs price direction is the first divergence check, iv_rank feeds the IV requirement. Rows 2–7 give the WEEK: read the daily net_premium sequence against the closes. A one-day divergence and a regime flip look identical in a single row (observed: +$ 109M on a red day read as "⚠️ divergence" alone, but the 7-row view showed −$ 266M/−$ 123M crash days immediately before — first positive net after capitulation, a far stronger signal). Flag: streaks (3+ same-sign days), sign flips after large moves, and IV-rank trajectory. `search_tickers` only if the symbol is ambiguous.

**Earnings date sourcing (verified 2026-07-06):** the snapshot payload does NOT contain next_earnings_date. It appears only in tape print payloads (`get_option_trades` rows). Read it from the first tape pull. If tape comes back empty and the earnings gate matters to the verdict, the card must declare the blind spot: "ER date unavailable from this pull." ETFs have no earnings; skip the gate.

**API clamps:** `min_premium` floors at $ 25,000 and `min_size` at 150 server-side. Thresholds below these are silently raised — never assume a sub-$ 25K pass ran.

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

## Hunt mode (market scan — find dive candidates, ≤7 calls)

Hunt produces a SHORTLIST, never verdicts — no levels, no conviction, no directional card. Every tag is provisional until quick/deep mode runs on the name. Budget: ≤7 calls (typically 2–4 per profile).

**H1 — Pick a profile by intent (all `get_stock_screener`; API quirks verified 2026-07-07):**

**API unit/sort traps (cost a full empty run when missed):** `min_change`/`max_change` are FRACTIONS — "0.005" = +0.5%; passing "0.5" means +50% and returns nothing (observed). `stock_volume_vs_avg30_volume` is a FILTER only, not a sort key on this API. `net_premium` = net_call_premium − net_put_premium (verified against per-ticker cards).

**EARLY profile (default in the morning) — "interest is starting, move not obvious yet." 2 calls, bull + bear mirror:**
- BULL: `min_stock_volume "500000"`, `min_stock_volume_vs_avg30_volume "1.2"`, `min_premium "250000"`, `min_net_premium "100000"`, `min_bullish_perc "0.55"`, `max_put_call_ratio "0.9"`, `min_change "0.005"`, `min_underlying_price "10"`, `min_earnings_dte "7"`, `issue_types ["Common Stock"]`, `hide_index_etf true`, order `net_premium` desc
- BEAR: mirror — `min_net_put_premium "100000"`, `min_bearish_perc "0.55"`, `min_put_call_ratio "1.1"`, `max_change "-0.005"`, order `net_put_premium` desc
- Variant: drop min/max_change to also catch contrarian dip/rip accumulation (observed 2026-07-06: GEO + CXW — same industry, both near 52wk highs, P/C 0.04/0.01 call accumulation on red days — a sector cluster no loud scan surfaces).
- EARLY has NO mcap floor by design; flag anything <$1B as thin-tape-weak (flow reads degrade per the thin-tape rule).

**LOUD profile — "what's the biggest flow today." 3 calls:**
- BULL tail: `order=net_premium` desc · BEAR tail: same asc · UNUSUAL: `order=volume` desc with `min_ratio_30_day_total "2"` (options volume ≥2× the name's OWN 30-day norm)
- Shared: `min_marketcap "2000000000"`, `min_premium "5000000"` ("2000000" on UNUSUAL), `hide_index_etf true`, `issue_types ["Common Stock","ADR"]`
- NEVER rank by absolute volume without the ratio filter — it returns the same megacaps every day (observed: ratio scan surfaced ORLY at 9.4× and UNM at 4.8× normal call volume — names no absolute sort ever shows). During earnings season the raw tails are mostly pre-ER positioning (observed: TSLA/META/AMZN/NFLX all inside ~3 weeks of ER).

**CONFIRM profile (late-day / next-day) — "the move is proving itself." 2 calls, bull + bear mirror of EARLY with tightened floors:**
- `min_stock_volume "1000000"`, `min_stock_volume_vs_avg30_volume "1.8"`, `min_premium "750000"`, net premium floor "300000", skew floor "0.65", P/C ≤0.7 (bull) / ≥1.3 (bear), `min_change "0.02"` / `max_change "-0.02"`
- CONFIRM names arrive with pumped IV by construction — any dive on them MUST weigh the IV requirement before structure.

**PROMOTION LADDER (requires the ledger — persistence IS the signal):** an EARLY name that reappears on a later run with (a) rising stock-vol-vs-avg30, (b) expanding net premium, (c) skew strengthening not weakening, and (d) price confirming the direction, graduates toward CONFIRM → that graduate is the priority dive candidate. One appearance = watch; repeat appearance with improving readings = act.

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

Tags: **CLEAN-BULL** · **CLEAN-BEAR** · **VOLUME-SPIKE** (participation elevated, direction mixed — often the best dive candidates) · **EVENT** (ER <14d) · **MUSH** (big net, failed skew — display once as a warning, never dive-worthy on this evidence) · **SPLIT-ARTIFACT** (corporate-action contamination, excluded).

**CONFLUENCE SCORE (0–7) — attach to every shortlisted name; ≥5 = dive-worthy.** Tags say WHAT a name is; the score says HOW MUCH agrees: (1) stock volume above its own 30d norm · (2) options premium above its own 30d norm · (3) net premium directional · (4) skew aligned with the net's direction · (5) clustered or repeated flow (same-session clustering via H2c, or multi-run persistence from the ledger) · (6) no ER/split/arbitrage distortion · (7) price confirming the direction. One loud metric with a 2/7 is noise by definition; a quiet name at 6/7 outranks a loud name at 3/7.

**H2c — Urgency check (1 call, market hours only, finalists only).** The MCP exposes no interval-flow endpoint — approximate it with one time-windowed tape pull: `get_option_trades` on the finalist, `newer_than` ≈ 45 minutes ago, `intraday_only`, tier premium threshold. Ask-side concentration + low multileg ratio + size clustered inside the window = urgency (upgrades confluence #5); the same premium dribbled across the whole session = downgrade. Urgency distinguishes "someone needs this position NOW" from "someone is patiently averaging" — both matter, but they are different trades.

**Junk to strike before scoring:** deep-ITM call prints near a `next_dividend_date` (dividend-capture, non-directional) · arbitrage/cross-tagged prints · big options premium with WEAK stock participation (relative_volume well under ~0.9 while options ratios scream — roll/split suspicion, run the detector) · names whose stats deteriorate on the midday recheck (skew fading intraday = the morning burst was distribution or noise, not a campaign; screener names can drop out of filters as later trades dilute the averages — a name that survives the whole session is itself a signal).

**H2b — Optional print check (1–2 calls).** Only when the user will act on the hunt itself rather than dive next: one `get_option_trades` pull covering the top 2–3 survivors (obey the group-tape crowding rule — group only names within ~1 order of magnitude of expected top-print premium). A candidate whose direction shows no clean single-leg ask/bid print at its tier threshold downgrades to AGGREGATE-ONLY. Skipping H2b is correct when a dive follows anyway — the dive re-runs tape properly, and hunt tags must never masquerade as tape reads.

**Output card (phone-first, max 8 rows):**
`TICKER $spot ±% · net $X · skew B% · vol ×N vs 30d · IVR · ER-days · TAG`
End with ONE line naming the 1–2 tickers that earn the dive and why — then stop. Hunt picks where the next 6–14 calls go; it never replaces them.

## Synthesis (no calls)

Quick mode output is a **report card per ticker — tight bullets, phone-readable:**

**TICKER** $spot (±day%) — **VERDICT** (NO EDGE / BULLISH / BEARISH / WAIT, + conviction)
- Net: $X (bull $Y / bear $Z) — append **⚠️** when the net direction disagrees with the largest prints
- Wk: the 7-day net-premium shape in one compact line (e.g., "wk: +267M → −266M → −123M → **+109M** — first positive since the flush") — from the snapshot rows, no extra call. Name streaks, flips, and whether today's net confirms or breaks the week's pattern
- Prints: the top 1–3 trades by premium, each as `$prem · contract · side` (e.g., "$ 3.4M · Dec-27 125c · bought @ ask"). Aggressor side is mandatory — "bought/sold @ ask/bid," never just the tag. If nothing clears the threshold, say so: "no prints >$ 150K" is the finding.
- IVR + next ER date (with days-out when inside ~45d)
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
