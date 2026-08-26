# Trading — Opening Range Breakout (ORB)

This repository holds the operating procedure and supporting artifacts for
running a Zarattini-style **Opening Range Breakout (ORB)** equity day-trading
strategy through the **Robinhood MCP** tools inside a Claude Code session.

There is no application code here — this is a rules-driven playbook that a
live Claude Code session follows, using `mcp__Robinhood__*` tools to read
market data and (when explicitly asked to trade live) place and manage
orders. Treat every instruction in the skill as binding: it exists to keep a
mechanical, risk-capped process instead of ad-hoc discretionary trading.

## Layout

- `.claude/skills/orb-strategy/SKILL.md` — the full ORB procedure: universe
  filters, direction bias, entry/stop/target rules, position sizing, and the
  step-by-step Robinhood MCP workflow. Invoke with `/orb-strategy` or by
  asking Claude to run/scan/trade the opening range breakout.
- `.claude/skills/orb-strategy/reference/risk-config.md` — the tunable risk
  parameters (risk per trade, max concurrent positions, stop style, target
  R-multiple, time cutoffs). Edit this file to change strategy behavior
  without editing the skill itself.
- `.claude/skills/orb-strategy/reference/trade-log-template.csv` — header row
  for the per-trade log. Each trading day's log lives under `logs/trades/`.
- `logs/trades/` — one CSV per trading day (`YYYY-MM-DD.csv`), one row per
  candidate evaluated (setups that passed *and* setups that were skipped,
  with a reason). This is the audit trail.

## Non-negotiables

- **No overnight holds.** Every position opened under this strategy is flat
  by the hard exit time in the risk config, same session.
- **Risk per trade is capped** at the percentage in
  `reference/risk-config.md` (default 1% of account equity), computed from
  a live `get_accounts` equity read — never assumed or estimated.
- **Never widen a stop.** Once set, a stop only tightens (trail) or triggers.
- **One setup per symbol per day.** Once a symbol has been entered or has
  been invalidated (time cutoff with no breakout), it's done for the day.
- **Placing or modifying real orders is an outward-facing, hard-to-reverse
  action.** Confirm the specific trade (symbol, side, size, entry, stop,
  target) with the user before the first live order of the day, unless the
  user has already told this session to run autonomously for the day.
