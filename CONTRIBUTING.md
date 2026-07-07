# Contributing

Thanks for wanting to improve the skill. This project has one governing rule, inherited from the skill's own post-run audit loop:

> **Skill edits happen ONLY from logged violations or from verdicts that conflicted with real outcomes — never from formatting preference. One observed failure = one targeted patch.**

Every rule in `SKILL.md` exists because a real misread happened in a live session. Contributions follow the same bar.

## What makes a good PR

1. **Evidence first.** Describe the observed failure: what the skill output, what the data actually said, and what a correct read would have been. Session transcripts, ticker + date, and the specific numbers are ideal. "This wording could be cleaner" is not a failure.
2. **One failure, one patch.** Keep PRs surgical — a single targeted rule change or addition. Broad rewrites are near-impossible to battle-test.
3. **Bump the version stamp.** Any edit to `SKILL.md` must increment the version line (`**Skill version: vN.**`) and add a `CHANGELOG.md` entry naming the driver. The stamp is how users know which copy of the skill answered.
4. **Don't break the budgets.** New mandatory checks must state their call cost and where it fits in the mode's call budget (or explicitly extend the accuracy-override list).
5. **Phone-first output.** Any output-format change must survive a ~380px screen: short lines, no wide tables, no code blocks inside cards.

## What gets rejected

- Style/formatting preference changes with no observed failure behind them
- Patches that turn ambiguity into signal (the skill says "bearish **or hedging**" on purpose)
- Anything that makes the skill recommend position sizes — it never does
- Rules that assume a specific UW subscription tier without a graceful degradation path

## Testing a patch

1. Install your edited copy locally (`~/.claude/skills/ticker-deep-dive/`)
2. Run the affected mode against **live or most-recent-close data** on at least 2–3 names, including one where the old behavior misfired
3. Include the before/after reads in the PR description
4. Note anything the patch could not verify (e.g., next-day OI confirmation still pending) — declared blind spots are fine; silent ones are not

## Reporting a misread without a patch

Open an issue with:
- Skill version stamp from the output (e.g., "Mode: DEEP (v11)")
- Ticker, date/time, and mode
- What the skill said vs what the data/outcome showed
- The raw numbers if you have them (prints, strikes, premiums, OI)

A well-documented misread is as valuable as a patch — it becomes the next targeted fix.

## Non-goals

- Multi-provider support (this skill is purpose-built for the Unusual Whales MCP)
- Automated trade execution of any kind
- Financial advice. The skill outputs research reads with invalidation levels; what you do with them is on you.
