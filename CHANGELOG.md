# Changelog

All notable changes to the ticker-deep-dive skill. The version is stamped in `SKILL.md` and echoed at the top of every skill output ("Mode: QUICK (v12)").

The format follows the skill's own improvement rule: **one observed failure = one targeted patch.** Each entry names the real-session failure that drove it.

## v14 — 2026-07-08

### Added
- **Premium-buy mode (naked calls/puts) — built and validated by a 4-regime blind historical backtest**, not by intuition. Replayed the evening pipeline as-of ~30 past sessions across four tapes (de-risk, chop, downtrend/IV-spike, rally/vol-crush; ~180 blind candidates, ~47 emitted), scored option-payoff proxies at realistic next-open fills, graded against what actually happened.
  - **Headline finding — direction alignment dwarfs every flow pillar.** Trend-aligned cards won by huge margins (rally window: bulls 4/4 vs bears 1/8; downtrend window: bears 67% vs bulls 29%); counter-trend emissions were the single largest loss source. So the mode leads with a hard **REGIME GATE**: uptrend → calls only, downtrend → puts only, **chop → stand down** (chop lost money in every configuration). The gate refuses far more than it emits.
  - **~90% of all losses in every regime were directional reversal** — theta-bleed and vega-crush losses were both rare. Even in the window designed to punish vega, only 1 of 12 emitted cards was crushed (and it outran the crush). Doctrine encoded: the risk on a 1–2 week naked hold is *being wrong on direction*, not decay or IV. Every card names the reversal level as its kill.
  - **Anti-chase is regime-conditional** (the backtest's sharpest correction): chasing loses in chop but WINS in a confirmed trend (chasers 56% and 45% in the two trending windows). So anti-chase is ON in chop/weak trends, OFF in strong trends — not a global veto.
  - **IVR downgraded from hard gate to disclosure.** Elevated IV rarely crushed buyers ("priced to move"); the 60–90 band was the worst cohort in 3 of 4 windows (extended-name tell) but ≥90 real-movers were fine. Cards state IVR + breakeven-in-implied-days rather than vetoing.
  - **Entry realism:** all internal grading assumes next-open fills; overnight gaps (~0.2 cone) roughly halve close-graded edge.
  - Status stamped as *framework validated on historical replay*, not proven edge — live forward test with real option P&L is the pending confirmation.

## v13 — 2026-07-07

### Added
- **Sector mode (1–3 calls).** New trigger: a THEME named instead of tickers ("strongest in semis," "scan the mags," "how's memory looking"). One `get_stock_screener` call on a curated ticker basket returns the full ranking board (perc_change, net premiums, skew, IVR, relative volume, ER dates); names are scored on 4 pillars (price leading · flow confirming · participation · clean) and output as a leaderboard with LEADER / FLOW-DIVERGENT / LAGGARD / EVENT-GATED tags plus a basket-level regime note (sector-wide tilt, sell-premium IVR regime). Ranks only — never verdicts or levels; the leader's dive still requires its own Phase-0 pull.
  - *Driver:* a Mag-7 "which is ready to run" scan cost 7 Phase-0 calls; one basket-screener call reproduced ~90% of the ranking signal. Battle-tested 2026-07-07 on semis (14 names) + memory (4 names): the leaderboard independently matched a full day of per-name dives (same leader, same laggard) AND caught a price/flow divergence a flow-alert-only hunt read had missed (−7.4% day behind the basket's biggest net call buying = dip accumulation, not strength).
- **Theme resolver table** (~20 trader themes: mags, semis, semicap, memory, networking/optics, software, cyber, AI-infra/neocloud, quantum, nuclear, space, defense, crypto-linked, EV, fintech, GLP-1, energy, financials, china tech) with a fallback rule for unknown themes (best-guess basket, name the proxy, invite correction). Baskets are dated; membership edits are maintenance, not version bumps.
- **Endpoint trap documented:** the screener's `etfs` constituent param returned EMPTY for SMH — curated `ticker` comma-lists are the reliable path; the `sectors` enum covers only the 11 GICS sectors.

## v12 — 2026-07-07

### Added
- **Move calibration (zero extra calls).** Phase 0 snapshot bumped `limit=7` → `limit=15` (same single call) so ATR-14 comes free. Every directional card now carries a MOVES line: ATR-14 (intraday range → stop hygiene) + weekly implied move = `implied_move_30` × √(7/30) (close-to-close displacement → target feasibility). Targets outside the weekly cone are labeled multi-week; stops inside ~1 ATR are labeled noise-stoppable.
  - *Driver:* levels ladders named targets the options market priced as fantasy, and "tight" stops sat inside one day's normal range. Tested live on NET/LLY/PLTR/SNOW (2026-07-07).
- **ATR ≈ 2× implied-daily documented as NORMAL** (range vs displacement; measured 2.0–2.2× on three names) — prevents misreading the gap as mispricing. Regime flag when realized close-to-close runs >1.3× implied.
- **Straddle upgrade rule** (1 call, only when the name moved >3–4% or IVR jumped): nearest-Friday ATM ±2 strikes via `get_chains_for_expiry`. Documented endpoint traps: no bid/ask fields (theo/last only), ~80K-char overflow.
  - *Driver:* √t scaling understated NET's cone ~25% on a +6% rip day — term structure inverts on event days.
- **MOMENTUM hunt tag** — EARLY cross-checks the 5-day run; >25% in ~5 sessions retags as mid-move. (*Driver:* TENB tagged "EARLY" on day ~5 of a +50% news run.)
- **CALL-SELLER hunt tag** — large negative net call premium failing bear skew = early-warning watch tag. (*Driver:* INTC flagged at −$90M call selling, matured into CLEAN-BEAR at −7% two sessions later.)
- **Early-session gate** — first ~30 min of RTH: skew/premium immature, ratio screen structurally empty; run on prior-close data or stamp EARLY-SESSION. (*Driver:* a 9:41 ET hunt returned an empty ratio screen and distorted skews.)
- **Confirm-mode package reconstruction** — OI-changes `volume_candles` reveal same-second spread legs without a tape re-pull. (*Driver:* a 7,500× 170/180 call credit spread initially read as two independent confirmations.)
- **Ledger convention + cone grading** — `huntlog.md` location/append rules codified; Friday wrap-ups grade tag-vs-outcome AND weekly-cone containment.

### Changed
- **Confluence pillar 7 (price confirming)** now judged open-to-close AND close-vs-prior, never close-only. (*Driver:* WULF's "+4.9%" close/close masked a −8% open-to-close failed-gap reversal — close-only nearly mis-scored it.)

## v11 — 2026-07-07

### Added
- **Two operating rhythms.** OVERNIGHT (primary): one evening pass on final session data → confluence scoring → deep dive the graduates → next-day focus card → **mandatory morning OI confirm** (≤2 calls) before acting. INTRADAY (occasional): morning EARLY → midday recheck → live urgency check → late-day CONFIRM.
  - *Driver:* the skill was tuned for intraday sampling, but the primary real-world use is after-hours DD to build next-day focus names. Post-close data was verified final/deterministic, making evening runs drift-free.
- Retrospective urgency check for evening runs: window the tape to the session's final 60–90 minutes — flow clustering into the bell is a next-day follow-through tell; all-day dribble is not.

## v10 — 2026-07-07

### Added
- **CONFLUENCE SCORE (0–7)** attached to every hunt shortlist name; ≥5 = dive-worthy. Pillars: stock volume vs own norm · options premium vs own norm · net directional · skew aligned · clustered/persistent flow · no distortion · price confirming.
  - *Driver:* single loud metrics (huge net premium) kept surfacing names that failed on every other axis. A quiet 6/7 beats a loud 3/7.
- **H2c urgency check** — one time-windowed tape pull (`newer_than` ≈ 45 min) to distinguish "needs the position NOW" from "patiently averaging."
- Junk-strike rules: dividend-capture deep-ITM prints, arbitrage/cross prints, options premium screaming while stock participation is weak (roll/split suspicion), intraday stat deterioration.

## v9 — 2026-07-07

### Added
- **Hunt profiles:** EARLY (pre-move discovery, bull+bear mirrors, no market-cap floor), LOUD (net-premium tails + unusual-volume ratio screen), CONFIRM (tightened floors for proven moves).
- **Promotion ladder:** an EARLY name reappearing with improving readings graduates toward CONFIRM and becomes the priority dive candidate. Persistence is the signal.
- **SPLIT/CORPORATE-ACTION DETECTOR** (mandatory before any hunt tag): fractional strikes in chain · total OI >2× in ≤2 sessions · options ratio spike with normal stock volume · implied-move inconsistency. Any two → tag SPLIT-ARTIFACT and exclude.
  - *Driver:* a 4:1 stock split inflated a name's volume ratios to 2.8×, corrupted OI history, greeks, and max pain — and the hunt mis-tagged it VOLUME-SPIKE. The deep dive caught it via four independent tells; the detector encodes them.
- **API unit traps** documented after they cost a full empty run: `min_change`/`max_change` are FRACTIONS ("0.005" = 0.5%; "0.5" = 50% returns nothing) · `stock_volume_vs_avg30_volume` is filter-only, not sortable · `net_premium` = net_call_premium − net_put_premium.

## v8 — 2026-07-07

### Added
- **Hunt mode** — market-wide screener funnel producing a coherence-scored SHORTLIST of dive candidates (never verdicts, never levels). H2 coherence gate: skew ≥60/40 · aggressor coherence · participation ≥1.5× 30-day norm · earnings gate (>14d).
  - *Driver:* ranking by net premium alone put 53/47 "mush" names at the top of every scan. The skew gate kills the #1 failure mode.
- Tags: CLEAN-BULL · CLEAN-BEAR · VOLUME-SPIKE · EVENT · MUSH · (v9 added SPLIT-ARTIFACT).

## v7 — 2026-07-06 (initial public baseline)

Core skill as battle-tested across multiple live sessions:

- Modes: quick / deep / scan / confirm, with per-mode call budgets and an accuracy override
- **Print supremacy rule** — verdicts from identifiable prints, never net aggregates (observed: −$15M net vs $5.5M of bullish structural prints; the prints were right)
- **Multileg reconstruction** — same-second sibling-leg matching; unresolved legs are excluded as AMBIGUOUS
- **Print hygiene** — canceled prints discarded; same-second cross-exchange fragments summed and read as one order (per-venue tags conflict)
- **Aggressor matrix, premium weighting, urgency tells, repetition-as-campaign, expiry coherence, hedge detection**
- Market-cap-tiered tape thresholds (mega $500K → small $50K; observed failure: a $300K threshold on an $8B name produced a false "no institutional flow")
- **ITM blind-spot rule** — OTM-filtered tape is structurally blind to deep-ITM accumulation; the strike pull is mandatory before claiming thin flow (observed: ~$950K of ITM call buying invisible to the OTM tape)
- **DATA GAP vs thin tape** — a missing feed is a plumbing finding, not a market finding; never conflate
- Earnings gate, IV requirement, verdict reconciliation, post-close data-reuse rule, ETF microstructure handling, group-tape crowding rule, environment adaptation (shell vs no-shell payload handling)
- Post-run audit loop: skill edits only from logged violations or verdicts that conflicted with real outcomes
