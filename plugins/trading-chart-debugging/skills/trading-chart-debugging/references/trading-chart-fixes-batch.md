# Case Study: Trading Chart Live-Tick + Cache UX (a fixes batch)

The earlier chart case study fixed "chart blank because the canvas was
0×0." This second case study comes from a later release, after the chart was
working but had a different class of bugs — the chart worked, but the data
was wrong or stale, and the cache UI was lying.

## What was different this time

By this point the chart had been live for weeks. The bugs weren't "chart
doesn't render" — they were:

- "The last candle is frozen at the wrong price."
- "The cached pill says 'Cached · 30s' but the cache TTL is actually 24h."
- "On a 1m timeframe, the X-axis time never advances."
- "When I switch symbols, the chart sometimes shows the wrong symbol's
  data because the previous symbol's positions are still in state."

These are **data correctness** bugs, not rendering bugs. The canvas was
fine; what was inside it was wrong.

## The fixes (5 of them, all data correctness)

### Fix 1: Hardcoded "Cached · 30s" badge (root cause #3h)

**Symptom:** The chart had a `.cache-badge` pill that always said
"Cached · 30s" regardless of cache age or source. The 30s was a vestigial
constant from an early prototype. The real cache TTL was 24h and there were
4 distinct source states (`live`, `cache`, `stale`, `stale-fallback`) that
the badge should reflect.

**Fix:**

- Server returns `source` and `cached_at` in the response body.
- Client tracks `cacheBadge: { source, ageMs }` state.
- A memoized label derives the visible string from source + age:
  - `live` → hidden (no badge for fresh fetches)
  - `cache` → "cache · 5m ago" (or 1h, etc.)
  - `stale` → "stale · 2h ago"
  - `stale-fallback` → "Stale (different timeframe)"

**Lesson:** when a TTL changes, the user-facing string MUST update with it.
Hardcoded "30s" left in the UI is a "we changed the implementation but not
the copy" bug. The fix is mechanical but easy to miss.

### Fix 2: Last candle frozen at the last snapshot close (root cause #0)

**Symptom:** Watchlist / hero price updated in real time but the most recent
candle on the chart stayed at the value from the initial market-data fetch
(e.g. candle close = 4511 but watchlist says 4062). The user saw "the price
says 4511 but the watchlist says 4062 — they should match."

**Diagnosis:** Candle data from the market-data provider is a snapshot of
historical closes. Between refreshes, the price drifts in the watchlist but
the candle's last entry stays frozen. Candles are anchored to time buckets,
not ticks.

**Fix (from SKILL.md root cause #0):** Pass the watchlist's `livePrice` into
the chart. The chart stores the last candle's full OHLC in a `useRef`, then
a `useEffect` on `livePrice` calls `candle.update()` to substitute close +
clamp high/low. Drop the `onLatestPrice` callback from the chart entirely —
the watchlist is the single source of truth for the displayed price.

### Fix 3: Bar close not detected — X-axis stuck (root cause #3f)

**Symptom:** On a `1m` (or `5m`, `1h`, `1d`) timeframe, the chart's X-axis
time label never advances. The "last candle" updates its close with each
tick, but a new candle never appears at the next period boundary.

**Diagnosis:** The standard live-tick pattern (update in-progress candle)
never detects "the period has ended — append a new bar." The chart shows
the current bar but the time axis thinks time has stopped.

**Fix:** Detect period rollover:

```ts
const periodSeconds = TF_SECONDS[timeframe];  // 60 for 1m
const nowSec = Math.floor(Date.now() / 1000);
const currentBarStart = nowSec - (nowSec % periodSeconds);

if (prev.time < currentBarStart) {
  // Previous bar has CLOSED — start a new in-progress candle.
  // Open = previous close so the bar is visually continuous.
  const fresh = {
    time: currentBarStart as UTCTimestamp,
    open: prev.close,
    high: Math.max(prev.close, livePrice),
    low: Math.min(prev.close, livePrice),
    close: livePrice,
  };
  lastCandleRef.current = fresh;
  candleRef.current.update(fresh);
} else {
  // Same bar — just update close + clamp high/low (standard tick).
  // ...
}
```

The `open = prev.close` trick is the key — without it, every new candle
opens with the watchlist's mid price, which causes a visible gap at the
period boundary. Anchoring `open` to `prev.close` makes the chart look
continuous.

Also: skip no-op ticks (`if (lastTickPriceRef.current === livePrice) return`).
A watchlist that polls every 10s would otherwise call `candle.update()`
with identical data 6 times per minute — wasted redraws.

Also: reset `lastTickPriceRef.current = null` when symbol or timeframe
changes — otherwise the first tick on a new symbol might be skipped because
it equals the previous symbol's last tick.

### Fix 4: Chart 0×0 on first mount (root cause #3g)

**Symptom:** On initial page load, the chart canvas was blank. After 1-2
seconds of activity (window resize, layout shift), the chart suddenly
rendered correctly.

**Diagnosis:** A single `requestAnimationFrame` re-fit wasn't enough.
Flexbox measurements can settle AFTER the first paint — especially with
SSR/hydration, dynamic imports, and conditional rendering.

**Fix (from SKILL.md root cause #3g):** Three-stage initial fit (rAF + 50ms
+ 250ms timeouts). The `ResizeObserver` handles every subsequent resize
including window resize and mobile orientation.

```ts
const applyFit = () => {
    if (containerRef.current && chartRef.current) {
        chartRef.current.applyOptions({
            width: containerRef.current.clientWidth,
            height: containerRef.current.clientHeight,
        });
        chartRef.current.timeScale().fitContent();
    }
};
const rafId = requestAnimationFrame(applyFit);
const t1 = setTimeout(applyFit, 50);
const t2 = setTimeout(applyFit, 250);
```

Always `fitContent()` in each fit — calling `applyOptions` resizes the
canvas, but if the data was `setData`'d before the resize, the chart's time
scale might still think the visible range is "everything" with weird
padding. `fitContent()` tightens to the actual data extent.

### Fix 5: The rate-limit cascade — wrong cache key

**Symptom:** The chart still showed a broker rate-limit error even after
deploying the new rate-limit cache. The user had to wait 5+ minutes and try
several (symbol, timeframe) combos to get the chart to load.

**Diagnosis:** The cache key was `${accountId}_${tiId}_${resolution}`. On a
cache HIT, the code checked `cached.to === to && cached.from === from`. But
`to = Math.floor(Date.now() / 1000)` — it changes every second. So
`cached.to !== to` ALWAYS — the cache NEVER hit.

**Fix:** Cache by stable params only (`${accountId}_${tiId}_${resolution}`),
store a generous lookback window once (e.g. 500 bars), and slice locally
for smaller requests:

```ts
const all = fullCached.results as any[];
const startIdx = all.findIndex((b) => b.time >= from);
const sliced = startIdx >= 0 ? all.slice(startIdx) : all;
liveResults = sliced;
chartSource = 'cache';
```

The cache key is invariant across requests; the window is a slice operation,
not a key collision.

This is a common failure mode, and it typically costs several iterations to
diagnose because the cache looks correct, the writes succeed, and the misses
are silent. See the `broker-rate-limit-cache` skill in this category for the
full cache pattern and the anti-pattern list — reading it before writing the
cache would have saved the debugging time.

## The fixes landed as one batch

These five bugs shipped in a single commit because they were all "the chart
works but is wrong" — a fix-batch was appropriate. The chart went from
"looks right but data is stale" to "data ticks live, bar rollover works,
cache badge is honest, resize doesn't break."

## Why this case study matters

The earlier chart case study was about "the chart doesn't render at all."
This one is about "the chart renders, but the data is wrong, the UI lies,
the cache is broken, and the user can't tell." Different bug class,
different diagnostic mindset.

When debugging a "chart that looks fine but the data feels off" problem,
the checklist is:

1. **Is the data flowing into the chart fresh?** Compare the chart's last
   candle close to the live price the user sees elsewhere. If they diverge,
   you're hitting Fix 2 (last candle frozen).
2. **Does the badge tell the truth about cache age?** If it says "Cached ·
   30s" but your TTL is 24h, the UI is lying. Fix 1.
3. **Does the bar rollover?** On a 1m timeframe, after 1 minute, does a new
   candle appear at the right edge? If not, Fix 3.
4. **Does the chart mount at the right size?** If it appears empty for 1-2
   seconds then snaps in, Fix 4 (three-stage fit).
5. **Is the cache actually hitting?** If the broker keeps returning a
   rate-limit error despite your cache, Fix 5 — likely a cache key that
   includes a time-window component.

The earlier fixes (CSS import, `position: absolute; inset: 0`,
max-height) were about rendering. These fixes were about data flow.
Different problems, different fixes, both about the same component.
