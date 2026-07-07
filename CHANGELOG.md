# Changelog

All notable changes to the ticker-deep-dive skill. The version is stamped in `SKILL.md` and echoed at the top of every skill output ("Mode: QUICK (v11)").

The format follows the skill's own improvement rule: **one observed failure = one targeted patch.** Each entry names the real-session failure that drove it.

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
