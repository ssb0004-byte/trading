# ORB Risk Configuration

Tunable parameters for the Opening Range Breakout skill. The skill reads
these values at the start of every run — edit this file to change behavior;
don't hardcode overrides in conversation unless the user explicitly asks for
a one-off deviation for a single session.

| Parameter | Default | Notes |
|---|---|---|
| `risk_per_trade_pct` | 1.0% | Hard cap, % of live account equity, per trade. Use ≤1%; if the user says 0.5%, use that instead. |
| `max_concurrent_positions` | 3 | Max simultaneous open ORB positions. |
| `stop_style` | `full_range` | One of: `full_range` (opposite side of OR), `midpoint` (OR midpoint), `atr10pct` (10% of 14-day ATR beyond entry), `breakout_extreme` (just beyond the breakout candle's extreme). Pick one and stay consistent within a day. |
| `target_r_multiple` | 1.5–2.0R | Primary profit target range. Default to 1.5R for the first scale-out if `scale_out` is true, 2R for the remainder. |
| `scale_out` | true | If true: sell 50% at 1R, trail/hold remainder to `target_r_multiple` or trail stop. |
| `entry_cutoff_et` | 11:00 | No new entries after this time; setup is invalidated if no clean breakout by 11:00–11:30 ET. |
| `time_stop_et` | 15:50 | Flatten any still-open position no later than this time. |
| `hard_exit_et` | 15:55 | Absolute latest flatten time — never hold past this into the close. |
| `min_price` | $5.00 | Universe filter: last price floor. |
| `min_avg_volume_14d` | 1,000,000 sh | Universe filter: 14-day average daily volume floor. |
| `min_atr_14d` | $0.50 | Universe filter: 14-day ATR floor. |
| `min_rvol_first_candle` | 100% | Universe filter: relative volume of the first 5-min candle vs. its typical volume. |
| `max_or_range_pct_of_price` | 1.5% | If ORB range height exceeds this fraction of price, skip the symbol or cut size (analyst judgment: cut to half size rather than an automatic skip if the setup is otherwise clean). |
| `watchlist_top_n` | 20 | Prefer the top 20 highest-RVOL names each day when the candidate list is larger. |

## Changing these values

If the user asks to change a parameter for "today only," apply it in-memory
for the session and say so explicitly in the trade log notes column — don't
edit this file. If they ask to change the default going forward, edit this
file and mention the diff.
