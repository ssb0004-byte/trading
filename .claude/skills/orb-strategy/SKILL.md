---
name: orb-strategy
description: Run the Opening Range Breakout (ORB) day-trading strategy via Robinhood MCP tools — scan the universe, compute the 09:30-09:35 ET opening range, determine direction bias, watch for a confirmed breakout close, size the position against account risk caps, and manage entry/stop/target/time-stop through the close. Use when the user asks to scan for ORB setups, run/trade the opening range breakout, check ORB candidates, or manage open ORB positions.
---

# Opening Range Breakout (ORB) Strategy

A mechanical, risk-capped ORB playbook. Follow it literally — do not
improvise entries, stops, or sizing outside these rules. All times are
US/Eastern. All account/risk numbers come from live Robinhood MCP reads,
never from assumption or from a prior turn's stale value.

Before running any phase below, read
`.claude/skills/orb-strategy/reference/risk-config.md` for the active
parameters (risk %, stop style, cutoffs, universe filter thresholds). Log
every candidate — passed or skipped — to
`logs/trades/YYYY-MM-DD.csv` using the header in
`reference/trade-log-template.csv` (create the day's file from that template
if it doesn't exist yet).

## Core definitions

- **Opening Range (OR):** high/low of the first 5-minute candle after the
  regular-session open, 09:30:00–09:34:59 ET.
- **ORB High / ORB Low:** high/low of that candle, wicks included.
- **ORB Range Height:** ORB High − ORB Low.
- Ignore pre-market and after-hours data entirely when computing the OR.

## Phase 0 — Preconditions

1. Confirm today is a regular trading session (not a weekend/market
   holiday). If unsure, check via `mcp__Robinhood__get_equity_tradability`
   or `mcp__Robinhood__get_equity_quotes` on a liquid symbol (e.g. SPY) and
   look at market/session state.
2. Pull live account state with `mcp__Robinhood__get_accounts` and
   `mcp__Robinhood__get_portfolio` — this is the equity figure risk % is
   computed against for the whole day. Re-pull before sizing each new
   position (equity moves as positions open/close).
3. If this is the first live order of the session and the user has not
   already authorized autonomous trading for the day, confirm with them
   before placing it: symbol, side, entry, stop, target, share count, and
   dollar risk.

## Phase 1 — Universe scan (~09:25–09:30 ET)

Build the day's candidate list before the open finishes forming:

1. Get a starting universe from the user's watchlist(s)
   (`mcp__Robinhood__get_watchlists`, `mcp__Robinhood__get_watchlist_items`)
   and/or a volume/movers scan (`mcp__Robinhood__create_scan` /
   `mcp__Robinhood__run_scan` / `mcp__Robinhood__get_scanner_filter_specs`
   for available filter fields, `mcp__Robinhood__get_scans` for existing
   scans). If the user gave an explicit symbol list, use that instead of
   scanning.
2. For each candidate, apply the universe filters using
   `mcp__Robinhood__get_equity_quotes` (last price),
   `mcp__Robinhood__get_equity_historicals` (14-day daily bars → avg
   volume, ATR-14), and `mcp__Robinhood__get_equity_fundamentals` if useful
   for context:
   - Last price ≥ `min_price`.
   - 14-day average daily volume ≥ `min_avg_volume_14d`.
   - 14-day ATR ≥ `min_atr_14d` (compute ATR-14 from the daily historicals —
     True Range = max(high−low, |high−prev close|, |low−prev close|),
     averaged over 14 sessions).
   - Prefer liquid large-caps/ETFs/"stocks in play" (news/catalyst
     preferred, not mandatory).
   - Exclude penny stocks and low-float names that fail the above.
3. Keep this as the day's watchlist. RVOL of the first candle (filter #4)
   can't be evaluated until 09:35 ET — defer that check to Phase 2.

## Phase 2 — Opening range + direction bias (after 09:35 ET)

Wait until the first 5-minute candle has **fully closed** (09:35:00 ET or
later) before doing anything below — no acting on a partially-formed candle.

For each surviving candidate, pull the 5-minute bar covering 09:30–09:35 ET
via `mcp__Robinhood__get_equity_historicals` (5-minute interval, regular
hours only):

1. `or_high` = candle high, `or_low` = candle low, `or_range` =
   `or_high - or_low`.
2. RVOL check: compare this candle's volume to the symbol's typical opening
   5-min volume (e.g. recent average opening-candle volume from the daily
   historicals or a short recent lookback). Require ≥ `min_rvol_first_candle`
   (100%). If Robinhood MCP has no direct RVOL field, compute the proxy
   from recent same-time-of-day volume and note the method used in the log.
3. Doji check: if `open ≈ close` (within a small tolerance, e.g. <10% of
   `or_range` or <0.1% of price) **and** `or_range` is tiny relative to
   recent ranges, mark it a pure doji → **no trade on this symbol today**,
   log and drop it.
4. Direction bias:
   - Candle closes green (`close > open`) → long bias only, watch for
     breakout **above** `or_high`.
   - Candle closes red (`close < open`) → short bias only, watch for
     breakdown **below** `or_low`.
5. `max_or_range_pct_of_price` check: if `or_range / last_price` exceeds the
   configured threshold, skip the symbol (or, if the setup is otherwise
   excellent, halve the position size later and note why in the log).
6. Rank the surviving candidates by RVOL and keep the top
   `watchlist_top_n` for active monitoring. Log every candidate here,
   including those just dropped, with `status=skipped` and a reason.

## Phase 3 — Breakout confirmation (each subsequent 5-min close, until `entry_cutoff_et`)

For each candidate still active, on every new 5-minute candle close:

1. Pull the latest closed 5-min candle
   (`mcp__Robinhood__get_equity_historicals`).
2. **Long candidates:** trigger requires this candle's **close** (body,
   not a wick) above `or_high`. **Short candidates:** close below `or_low`.
   A wick-only pierce that closes back inside the range is not a trigger —
   keep watching the next candle.
3. Prefer the 2nd or 3rd 5-min candle after the open over the very next one
   (09:35–09:40 candle) to avoid immediate fakeouts, unless the breakout on
   that very next candle is unusually clean (strong body, volume ≥ average
   of recent opening candles) — use judgment, note it in the log either way.
4. Optional stronger confirmation (apply consistently, not selectively):
   candle body occupies most of the candle's range (not a long-wick close),
   and candle volume ≥ the average volume of the prior opening candles for
   that symbol.
5. No entry after `entry_cutoff_et` (default 11:00 ET) — if no clean
   breakout has occurred by then, invalidate the setup, log `status=expired`,
   and stop watching that symbol for the day.
6. **One setup per symbol per day, max.** Once a symbol triggers an entry
   (Phase 4) or expires, it's done — don't re-enter it later even on a
   second breakout.

## Phase 4 — Entry, stop, target, sizing

On a confirmed breakout candle close:

1. **Entry price:** the breakout candle's close, or the next candle's open
   if you choose to confirm the breakout holds — pick one approach and use
   it consistently for the day; note which in the log.
2. **Stop price**, per `stop_style` in the risk config:
   - `full_range`: long stop = `or_low`; short stop = `or_high`.
   - `midpoint`: stop at `(or_high + or_low) / 2`.
   - `atr10pct`: stop = entry ∓ (10% of 14-day ATR).
   - `breakout_extreme`: stop just beyond the breakout candle's own
     high/low (long: just below its low; short: just above its high).
3. **Risk per share** = `|entry_price - stop_price|`. Reject the setup if
   this is zero or negative (bad data — skip and log).
4. **Position size:** pull current equity
   (`mcp__Robinhood__get_accounts` / `get_portfolio`), then
   `shares = floor((equity * risk_per_trade_pct/100) / risk_per_share)`.
   Cap by available buying power — never size past what the account can
   actually cover
   (`mcp__Robinhood__get_accounts` buying power field). If the ATR/OR-range
   check in Phase 2 flagged an oversized range, halve this before rounding.
5. **Target price:**
   - Primary: `entry ± target_r_multiple * risk_per_share` (1.5R–2R).
   - Cross-check against the alternative (50–100% of OR range height
     projected from the breakout level) — if they diverge a lot, prefer
     the tighter of the two unless the user says otherwise.
   - If `scale_out` is true: first target at 1R for 50% of shares, remainder
     targeted at `target_r_multiple` or trailed.
6. Re-derive `risk_pct_equity = (shares * risk_per_share) / equity` and
   confirm it's ≤ the configured cap before placing anything. If a bigger
   share count would breach the cap, round down, never up.
7. Enforce `max_concurrent_positions` — don't open a new ORB position if
   the account already holds that many open ORB positions
   (`mcp__Robinhood__get_equity_positions`).
8. Confirm the trade with the user if required per Phase 0 step 3, then
   place the order with `mcp__Robinhood__place_equity_order`
   (use `mcp__Robinhood__review_equity_order` first if available/useful to
   sanity-check estimated cost/fees before submitting). Record the resulting
   order id.
9. Log the trade row immediately: OR levels, bias, entry time/price, stop,
   target, share count, dollar and % risk, R:R, and pass/fail reasoning for
   every candidate considered (not just the ones taken).

## Phase 5 — Managing the open position

Robinhood MCP may not support bracket orders on all accounts — manage exits
by monitoring and placing opposing orders yourself:

1. Poll price/position state periodically
   (`mcp__Robinhood__get_equity_quotes`, `mcp__Robinhood__get_equity_positions`,
   `mcp__Robinhood__get_equity_orders`).
2. **Stop:** if price trades through the stop, exit immediately at
   market/marketable-limit via `mcp__Robinhood__place_equity_order`
   (opposite side, full remaining size). **Never move the stop further
   away** — only tighten/trail once in profit (e.g. under/above the most
   recent 5-min swing, or VWAP) if the user wants a trailing approach.
3. **Target(s):** on reaching 1R (if scaling out), sell 50% and move the
   stop on the remainder to breakeven or trail it; on reaching the full
   target, close the remainder.
4. **Time stop:** if momentum has clearly died before `time_stop_et`,
   consider flattening early — use judgment, log the reasoning.
5. **Hard time-based flatten:** no later than `time_stop_et`
   (default 15:50 ET), and absolutely no later than `hard_exit_et`
   (15:55 ET) — flatten any still-open ORB position with a market order.
   No overnight holds, ever, under this strategy.
6. Update the trade log row with exit time/price, realized P&L, and
   realized R-multiple as soon as the position is closed.

## Phase 6 — End of day

1. Confirm no ORB positions remain open past `hard_exit_et`
   (`mcp__Robinhood__get_equity_positions`).
2. Finalize `logs/trades/YYYY-MM-DD.csv` — every candidate scanned should
   have a row, whether traded, skipped, or expired, with a reason.
3. Summarize the day for the user: setups seen, setups taken, win/loss,
   total realized P&L, and total risk deployed vs. the cap.

## Hard rules (never violate)

- No trade on a pure-doji first candle.
- No entry on a wick-only pierce — require a body close outside the range.
- No entry after the configured cutoff without a clean breakout.
- No more than one entry per symbol per day.
- No stop ever moved further from price after entry.
- No position sized past the configured risk-per-trade cap or past
  available buying power.
- No more than `max_concurrent_positions` open ORB positions at once.
- No overnight holds — flatten everything by the hard exit time.
- If required data (price, historicals, account equity) is missing or
  ambiguous for a candidate, skip it and log why — don't guess.
