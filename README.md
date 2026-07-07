# Ticker Deep Dive

**Options-flow due diligence skill for Claude, powered by the [Unusual Whales](https://unusualwhales.com) MCP.**

Turns "deep dive NVDA" into a structured, decision-grade read: who is paying up, for what strikes and expiries, with how much urgency — cross-checked against dealer positioning, dark pool levels, and catalysts. Battle-tested against live market sessions; every rule in the skill exists because a real misread was observed and patched.

> ⚠️ **Not financial advice.** This skill produces research summaries of publicly observable options-market data. It never recommends position sizes and treats "no edge" as a first-class conclusion. Trade at your own risk.

## What it does

| Mode | Trigger | Calls | What you get |
|------|---------|-------|--------------|
| **Quick** | "quick pull TSLA", "check the flow on AMD" | ≤6 | Flow + tape report card: net premium, top prints with aggressor side, week shape, IV rank, verdict + invalidation level |
| **Deep** | "deep dive LLY", "full DD" | ≤14 | Quick card **plus** dealer gamma map (walls, flip, max pain), dark pool levels, week-sweep persistence check, catalysts, "what kills this thesis" |
| **Scan** | 4+ tickers named | 2–3/name | Breadth-first cards across a watchlist |
| **Confirm** | "confirm TSLA", any next-day follow-up | ≤2 | Did yesterday's flagged volume become real open interest? OPENED / CHURN / PARTIAL per contract |
| **Hunt** | "find me tickers", "what's moving" | ≤7 | Market-wide screener funnel → coherence-scored shortlist of dive candidates (EARLY / LOUD / CONFIRM profiles) |

### The discipline baked in

- **Print supremacy** — verdicts trace to specific large prints, never to net-premium aggregates alone
- **Multileg reconstruction** — spread legs are never read standalone; packages are rebuilt from same-second sibling legs
- **Aggressor matrix** — calls@ask ≠ calls@bid; puts sold at the bid are bullish; mid prints are discounted
- **Earnings gate** — flow spanning an ER date is capped at moderate conviction ("directional or event hedge")
- **Volume ≠ OI** — every volume-over-OI read is labeled unconfirmed until the next morning's OI update
- **Split/corporate-action detector** — refuses to read contaminated post-split data as signal
- **Confluence scoring (0–7)** — hunt candidates are scored on how many independent metrics agree; a quiet 6/7 outranks a loud 3/7

## Requirements

1. **Claude** with skill support — [Claude Code](https://claude.com/claude-code) (CLI, desktop, or web) or claude.ai (Pro/Max/Team)
2. **The Unusual Whales MCP server** connected to your Claude environment (setup below) with an active [Unusual Whales API subscription](https://unusualwhales.com/public-api)

**The skill does not work without the Unusual Whales MCP.** Every phase reads UW endpoints (flow per strike/expiry, tape prints, greek exposure, max pain, dark pool, OI changes, screeners). With no UW connection the skill has no data and will tell you so.

## Step 1 — Set up the Unusual Whales MCP

1. **Get an API key:** subscribe to the UW API, then copy your key from the [API dashboard](https://unusualwhales.com/settings/api-dashboard).
2. **Connect it to Claude:**

   **Claude Code (CLI):**
   ```bash
   claude mcp add --transport http unusualwhales https://api.unusualwhales.com/api/mcp --header "Authorization: Bearer YOUR_KEY"
   ```

   **claude.ai (web/mobile/desktop):** Settings → Connectors → *Add custom connector* → URL `https://api.unusualwhales.com/api/mcp`, authenticated with your key.

   ⚠️ Common mistake: `unusualwhales.com/public-api/mcp` is the **docs page**, not the server. The server endpoint is `https://api.unusualwhales.com/api/mcp`.

3. **Verify:** ask Claude "what Unusual Whales tools do you have?" — you should see ~90+ tools (flow, greeks, dark pool, screeners, etc.).

Full setup docs: [UW MCP Setup Guide](https://unusualwhales.com/public-api/mcp) · [official MCP repo](https://github.com/unusual-whales/unusual-whales-official-mcp) · [@unusualwhales/mcp on npm](https://www.npmjs.com/package/@unusualwhales/mcp) (local/stdio alternative via `UW_API_KEY`).

The skill degrades gracefully: endpoints your plan doesn't cover are declared as named blind spots in the output rather than silently skipped.

## Step 2 — Install the skill

### Claude Code (CLI / desktop)

Clone straight into your skills directory — the repo root **is** the skill:

```bash
git clone https://github.com/Tle101/ticker-deep-dive.git ~/.claude/skills/ticker-deep-dive
```

Updating later:

```bash
cd ~/.claude/skills/ticker-deep-dive && git pull
```

Project-scoped install (this project only) instead:

```bash
git clone https://github.com/Tle101/ticker-deep-dive.git .claude/skills/ticker-deep-dive
```

### claude.ai / mobile (upload as a `.skill` file)

Package the skill folder as a zip whose root folder matches the skill name:

```bash
git clone https://github.com/Tle101/ticker-deep-dive.git
cd ticker-deep-dive
mkdir -p /tmp/pkg/ticker-deep-dive
cp SKILL.md /tmp/pkg/ticker-deep-dive/
cd /tmp/pkg && zip -r tickerdeepdive.skill ticker-deep-dive/
```

Upload `tickerdeepdive.skill` in your claude.ai skill settings (or attach it in a conversation and ask Claude to install it).

### Verify the install

Ask Claude: `quick pull SPY`. Every output opens with a mode + version stamp, e.g. **"Mode: QUICK (v11)"** — if you see the stamp, the skill is live and you know exactly which version answered.

## Usage examples

```
deep dive LLY
quick pull TSLA AMD
scan rklb lly glw shop vst
confirm TEVA            # next-day: did yesterday's flagged flow open as OI?
run hunt                # market-wide scan for dive candidates
find me tickers with clean directional flow
```

Typical rhythm for swing traders (the skill's primary design target):

1. **Evening** — `run hunt` on final session data → confluence-scored shortlist → deep dive the 1–2 graduates → next-day focus card
2. **Next morning** — `confirm <ticker>` (≤2 calls): did the flagged volume become open interest overnight? Opened = act; churn = strike it
3. **Intraday (occasional)** — quick pulls and midday hunt rechecks

## Versioning

The skill version is stamped in `SKILL.md` (currently **v11**) and echoed at the top of every output. The version bumps on **every** edit to `SKILL.md` — this is how you tell which copy answered when the same skill lives in Claude Code, claude.ai, and mobile at once.

See [CHANGELOG.md](CHANGELOG.md) for the full evolution and the observed failures that drove each patch.

## Contributing

Patches are welcome — but this skill evolves by **evidence, not preference**. Every rule in it traces to an observed failure in a live session. Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR.

## License

[MIT](LICENSE)
