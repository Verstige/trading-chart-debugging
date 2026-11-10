# Case Study: Trading Chart Blank Canvas (8 debug rounds)

Eight debug rounds over several hours. The chart's price data was loading
correctly from Polygon, the watchlist had live prices, the performance
metrics updated, the buy-bias gauge worked — but the candlestick canvas
itself was blank (or collapsed to 2px tall).

This case study records what failed at each round and the underlying root
cause, so the next person hitting "chart loads but no candles" can skip the
false leads.

## The component

`components/chart/TradingChart.tsx` — wraps `lightweight-charts` v5. Used
inside `TradingTerminal.tsx` inside `.chart-region` inside `.chart-panel`
inside the trading page.

## The symptom

- Watchlist shows live prices (`XAUUSD 3,347.85 +0.33%`)
- Performance bar shows real change %
- Technical buy-bias gauge works
- "Charting by TradingView" watermark visible
- No candlesticks

## The 8 rounds

### Round 1: "The Polygon API is returning Access denied"
- Tested `/api/polygon/agg?symbol=XAUUSD&tf=1h` directly
- Response: `{"error":"Access denied."}`
- Conclusion: the `POLYGON_API_KEY` in Railway is bad
- **Wrong** — this was the SECOND time a stale process served a stale
  cached response. The actual cause was an even older key. When the user
  updated the key in Railway, it worked.

### Round 2: "The Polygon key isn't configured in Railway"
- Started fresh dev server, hit `/api/polygon/agg`
- Response: `{"error":"POLYGON_API_KEY not configured on server..."}`
- Conclusion: key not yet set in Railway (this was correct)
- **Right** — but took a while to figure out which process held the
  old env. Required kill + clearing the build output directory + restart of
  `railway run`.

### Round 3: "Polygon works! 312 candles for XAUUSD 1d"
- Tested all symbols. XAUUSD, BTCUSD, NAS100, GBPJPY all return real daily
  candles (250-365 each). US30 returns 403 NOT_AUTHORIZED (expected on the
  free tier — `I:DJI` requires an upgrade).
- Chart data was confirmed working.

### Round 4: "The chart shows price but no candles"
- Clicked MARKETS → Live Chart
- Active view header shows `XAUUSD 1h 4,511.39 +13.82 (+0.41%)`
- Watchlist populated with all 5 symbols
- No candlesticks drawn on the canvas

### Round 5: "The canvas is 2px tall"
- DOM inspection:
  - `.chart-region` (parent): 874w × 1147h
  - `.chart-frame` (inner): 874w × 24h
  - `canvas`: 816w × 2h
- **Diagnosis:** inner wrap collapsed to 24px because it had
  `position: relative; height: 100%` while the parent had `min-height: 460px;
  flex: 1`. The flex parent stretched to fill the viewport. The inner wrap
  took `100%` of nothing because it wasn't a flex child itself — it took
  its tallest child's content height instead. The 24px child was a
  `.cache-badge` with `position: absolute; top: 16px; right: 16px;` — a
  24px tall pill that no one noticed was the only thing in the wrap.
- **Fix applied:** change `.chart-frame` to `position: absolute; inset: 0`.
  The cached badge stays `position: absolute` so it floats over the chart.

### Round 6: "It's still 24px tall"
- DOM inspection after the fix: still 24px, `cssPosition: 'static'`
- **Diagnosis:** the stylesheet was never being served. The browser has no
  `.chart-frame` rule in its stylesheets.
- **Root cause:** `TradingChart.tsx` imports the chart library and the
  component, but it does NOT import `TradingChart.css`. So every previous
  CSS edit (over multiple rounds!) was patching a file that the browser
  never saw.
- **Fix applied:** added `import './TradingChart.css'` to
  `TradingChart.tsx`.

### Round 7: "Now 1147px — too tall"
- After the CSS import, `cssPosition: 'absolute'` finally applied. But the
  chart was 1147px tall (the full viewport) instead of the intended
  280-460px.
- **Diagnosis:** `flex: 1` on the wrap was expanding to fill the parent flex
  container. The parent had no `max-height` cap.
- **Fix applied:** add `max-height: 280px; flex-shrink: 0` to the wrap. Also
  reduced min-heights on `.dashboard-grid` and `.chart-panel` from 600px to
  360px.

### Round 8: "Remove the volume indicator at the bottom"
- User said "remove the volume indicator at the bottom we don't need that"
- Removed `addSeries(HistogramSeries, ...)` from the chart
- Removed the `volRef` ref + the `volRef.current?.setData(...)` call
- Removed the `HistogramSeries` import
- **Final state:** clean 280px chart with just candlesticks + price/time
  scales

## The 3 root causes (matching the SKILL.md)

| Round | Root cause | Diagnostic | Fix |
|---|---|---|---|
| 5 | Flex collapse — inner wrap took child content height | `wrapH: 24, parentH: 1147` | `position: absolute; inset: 0` on wrap |
| 6 | Component doesn't import its CSS | `getComputedStyle(el).position === 'static'` despite CSS rule | Add `import './ChartComponent.css'` |
| 7 | Chart canvas sized to fill viewport | `wrapH: 1147, cssH: '1146.66px'` | `max-height: 280px; flex-shrink: 0` |

All three are recorded in the main SKILL.md with diagnostic patterns and
fixes.

## What I'd do differently next time

- **Round 1:** check the running process's configuration before declaring
  the key bad. The "Access denied" was a stale response from a previous
  process holding old configuration.
- **Round 6:** check `getComputedStyle` on the FIRST CSS edit round. If the
  value didn't match what I wrote, the CSS wasn't being applied — period.
  The 5+ rounds of CSS edits that "didn't take" were because of this single
  missing import.
- **Rounds 5-7:** start with the triage script that's in the main SKILL.md.
  It would have surfaced all three issues in one diagnostic pass.
