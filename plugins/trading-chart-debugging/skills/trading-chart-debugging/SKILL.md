---
name: trading-chart-debugging
description: "Diagnose \"the structure is right but the visuals are wrong\" rendering bugs in trading dashboards \u2014 blank chart canvases, page sections that render unstyled, content hidden behind fixed-position background layers, pop-ups that appear shifted because animation overrides a CSS transform, and charts that render but show stale or wrong data. A class-level umbrella for the family of bugs where the build is clean, the DOM is correct, and computed styles prove your CSS rules are live while the page still looks broken. Covers the root causes that hit any Next.js App Router page or component with co-located CSS: a missing CSS import on the page or component (the build ships the React markup but never bundles the stylesheet), flexbox parents collapsing to their content's height (a chart, a scrollable panel, or any flex-column leaf that needs `min-height: 0`), fixed-position background layers painting over static-positioned sections (z-index has no effect on static elements), canvas libraries capturing `clientWidth`/`clientHeight` at mount before flexbox has measured, Framer Motion's `animate={{ x: 0 }}` clobbering a CSS `transform: translateX(-50%)` centering rule, lightweight-charts v5 typing and the `series.update()` full-OHLC requirement, period-rollover detection for live ticks, and stale \"Cached\" badges that don't track the underlying TTL. Use when a chart shows nothing but the watermark, when data flows but no shapes draw, when a page section renders as raw browser defaults despite a sibling CSS file existing, when `elementFromPoint` returns a background layer over your content, when `position: absolute` doesn't apply because the stylesheet isn't loaded, when a flex leaf collapses to its tallest child's height, when the last candle is frozen while the price ticker moves, or when any Next.js render looks \"wrong but the build is clean."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# Trading Chart Debugging

When a chart shows data flowing in the UI (price ticks, watchlist updates,
performance metrics) but the chart canvas itself is blank, the bug is in the
**layout chain around the canvas**, not in the chart library. There are four
recurring root causes. Run this checklist in order.

This skill is deliberately broader than charts. The same "structure right,
visuals wrong" failure modes hit ordinary pages, panels, and modals — a
missing stylesheet import or a collapsed flex leaf breaks a marketing
section exactly the way it breaks a candlestick chart. Treat the root causes
as a family.

## The 4 root causes (in order of frequency)

### 0. Live price ticks but chart's last candle is frozen

**Symptom:** Watchlist / price header updates in real time, but the most
recent candle on the chart stays at the value from the initial Polygon fetch
(e.g. last candle's close = 4511, but the watchlist is now showing 4062).
The user sees "the price says 4511 but the watchlist says 4062 — they should
match."

**Test:**
```js
const hero = document.querySelector('.chart-price-value');
const watch = document.querySelector('.watch-row.active .watch-price');
console.log('hero:', hero?.textContent, 'watch:', watch?.textContent);
// If they differ by more than 0.01, the chart is showing stale data.
```

**Why:** Candle data from Polygon is a snapshot of historical closes (e.g.
"at 4 PM the price was 4511"). Between Polygon refreshes, the price drifts
in the watchlist but the candle's last entry stays frozen. Candles are
anchored to time buckets, not ticks.

**Fix — store the last candle + re-apply on every live-tick:** The parent
component passes a `livePrice` prop (the watchlist's currently-drifting
value) to the chart. The chart stores the last candle's full OHLC in a
`useRef`, then a `useEffect` on `livePrice` calls `candle.update({...})` to
substitute close + clamp high/low.

```tsx
const lastCandleRef = useRef<{time, open, high, low, close} | null>(null);

// On initial load, remember the last candle:
lastCandleRef.current = data[data.length - 1];

// On every live-tick, re-apply the last candle with the new close:
useEffect(() => {
    if (livePrice == null || !candleRef.current || lastCandleRef.current == null) return;
    const prev = lastCandleRef.current;
    candleRef.current.update({
        time: prev.time as Time,
        open: prev.open,
        close: livePrice,
        high: Math.max(prev.high, livePrice),
        low:  Math.min(prev.low,  livePrice),
    });
    lastCandleRef.current = { ...prev, close: livePrice, high: ..., low: ... };
}, [livePrice]);
```

Standard pattern in real trading platforms. The candle's body grows/shrinks
in real time as the watchlist drifts. Also standardizes the chart's hero
price to match the watchlist (drop the chart's `onLatestPrice` callback —
let the watchlist be the single source of truth for the displayed price).

### 1. Component doesn't import its CSS file

**Symptom:** `getComputedStyle(chartWrap).position` returns the default
(e.g. `static`) when you've written `position: absolute` in the CSS file.
`getComputedStyle(canvas).height` is 2px even though the parent is 1000px
tall. Your CSS edits "never apply." Same for non-chart pages: rewriting a
page from scratch and forgetting the `import './page-name.css'` line
produces a fully-built, fully-deployed CSS file on disk that never enters
the bundle — the page renders with raw browser defaults (white text on a
dark background for a dark-themed app, no layout, no colors, no animations).
The DOM structure is correct, but every styled class is a no-op.

**Not just charts.** This applies to **every Next.js App Router page or
component** with co-located CSS: charts (per the original case), onboarding
wizard pages, marketing pages, dashboard sections, anything imported via
`import './X.css'`. Next.js only bundles CSS that is imported (directly or
transitively) by a page or component file. A `.css` file sitting next to a
`.tsx` file is dead code unless something imports it.

**Test:**
```js
// Run in browser console, scoped to a route
const sheets = Array.from(document.styleSheets).map(s => s.href).filter(Boolean);
console.log('Sheets loaded:', sheets);
// If your-page-styles.css isn't in there, missing import is the bug.

// Or by class — confirm a styled rule actually computed:
const el = document.querySelector('.wizard-root');   // or '.chart-frame'
console.log(getComputedStyle(el).position);          // should be 'absolute' if your CSS works; static = missing import

// Long-form: every class on the page should have its declarations applied.
// If `style="..."` inline attrs look right but no className produces the
// expected colors / spacing / size, you're looking at dead CSS.
```

**Fix:** The component (or page) imports the CSS but forgot
`import './Component.css'`. Add it. **And always add it back when
rewriting a page from scratch** — that path strips import lines if the
template you're copying from is missing them.

```tsx
// ChartComponent.tsx
import './ChartComponent.css';  // ← this line is missing

// OR for a page wrapper at app/page.tsx
import './setup.css';    // ← this line is missing
```

**Production scenario where this bites hardest:** when the build output
directory is cleared, or a cache-config line forces a fresh emit of the page
chunk, the bundler re-walks the import graph from scratch. If the CSS
import line was never there (or was lost in a patch round-trip), the chunk
ships without the CSS — and the user reports "all I'm seeing is white text
without any CSS styling." The CSS file appears intact on disk; version
control shows it hasn't changed; the type checker reports zero errors. The
build passes cleanly. The bug is purely a missing-import line in the page
that no automated check catches.

**Code-review discipline (non-negotiable):** every time you create or
rewrite a Next.js page file with co-located CSS (`page.tsx`, `layout.tsx`, or
a component file with `*.module.css` / `*.css` next to it), **before writing
the file, check whether a sibling `.css` exists. If yes, your first TSX
line after the imports block is `import './that-file.css'`**. Add it to your
mental pre-flight checklist alongside 'check the TypeScript path aliases'
and 'check the parent layout exists'.

**How this shows up in production:** a multi-step wizard page was rewritten
from scratch to consolidate an 8-step multi-page flow into a single-page
6-step wizard (~37k chars of JSX). The 40KB co-located stylesheet sat
untouched on disk. The build was clean, type-check was clean, the smoke test
passed 14/14. The production chunk shipped with all the React markup but no
CSS classes were styled. A user opened the route and saw the wizard rendered
as raw white text on dark — exactly the "white text, no styling" report.
Diagnosed in one round: `import './setup.css'` was missing. Added it back to
all four pages (main + the three sub-pages), bumped the service-worker cache
version, redeployed. A 26KB production CSS chunk appeared where there had
been none. Fix verified.

Other reference docs for adjacent rendering bugs:

- `references/case-study-trading-chart-import.md` — the original
  chart-importer-missing case (8 debug rounds). Same bug class as this, but
  specific to the trading-chart component. Useful as the predecessor case
  study.
- `references/framer-motion-centering-conflict.md` — same symptom family (a
  positioning rule from your CSS doesn't take effect), but the fix is
  `x: '-50%'` in the motion variant, not a missing import.

### 2. Inner wrap collapses to child height (flex collapse)

**Symptom:** Chart container is 1147px tall (full viewport) but the *inner*
wrap is 24px tall, exactly matching its tallest child (a "Cached" badge or a
loading spinner). The canvas draws into a 0-height or 24-height region.

**Also applies to non-chart flex column layouts:** the same collapse hits
social feeds, community pages, and any scrollable panel inside a flex
column — not just charts. All have `height: 100%` on the leaf node but no
`flex: 1` on intermediate wrappers. The symptom is identical: the scrollable
region shows nothing (0px) even though children are rendering.

**The flex height chain rule:** Every container in a flex column must
declare `flex: 1` AND `min-height: 0` to avoid collapsing. The chain must
be unbroken from viewport down to the leaf scroll container:

```
Viewport (body/html or a flex parent)
  └── flex container (display:flex; flex-direction:column; flex:1; min-height:0)
        └── motion.div / div (flex:1; min-height:0; height:100%)  ← motion.div in tab views
              └── CommunityPage (height:100%; min-height:0; flex:1)
                    └── .panel-root (height:100%; min-height:0; flex:1)
                          └── .feed-shell (flex:1)  ← leaf fills remaining space
                                └── .feed-panel (overflow-y:auto; height:100%)  ← scroll here
```

Without `min-height: 0` on a flex item, the default `min-height: auto`
makes the flex item default to its content size — collapsing the feed to its
content height (often 0 or a single line).

**Fix:** Add `min-height: 0` to every flex item in the chain. In Framer
Motion tab views, the `motion.div` wrapping a lazy-loaded component must
carry `style={{ display:'flex', flex:1, minHeight:0, height:'100%' }}` to
complete the chain. If the lazy-loaded component is behind a React
`Suspense` boundary, the boundary's fallback element must not be the thing
that establishes the chain's height — put the flex properties on the
suspense wrapper.

**Diagnosis:** `document.querySelector('.feed-panel')?.offsetHeight` — if
it's 0 but children are in the DOM, the chain is broken somewhere above.
Find the first parent with `display: flex` and `flex-direction: column` — it
needs `min-height: 0` if it has no explicit height.

**Test:**
```js
const el = document.querySelector('.feed-panel');
let parent = el?.parentElement;
while (parent) {
  const cs = getComputedStyle(parent);
  if (cs.display === 'flex' && cs.flexDirection === 'column') {
    console.log('flex chain at:', parent.className, 'minHeight:', cs.minHeight, 'flex:', cs.flex);
  }
  parent = parent.parentElement;
}
// If any flex item shows minHeight: auto (the default), that's the bug.
```

**Fix for a community/feed layout (a real production case):**
```css
/* .feed-shell — the flex column leaf */
.feed-shell { display: flex; flex-direction: column; flex: 1; min-height: 0; }

/* .panel-root — parent of feed-shell */
.panel-root { height: 100%; min-height: 0; flex: 1; }

/* motion.div wrapping CommunityPage in tab pane */
<motion.div style={{ display: 'flex', flex: 1, minHeight: 0, height: '100%' }}>
    <CommunityPage />
</motion.div>
```

**Test:**
```js
const parent = document.querySelector('.chart-region');  // whatever the immediate parent is
const wrap = parent?.querySelector('.chart-frame');
console.log('parent:', parent?.offsetHeight, 'wrap:', wrap?.offsetHeight);
// If parent is 1147 and wrap is 24, the wrap is collapsed.
```

**Why:** The wrap had `position: relative; height: 100%` and the parent had
`min-height: 460px` and `flex: 1`. The flex parent stretched to fill the
viewport (1147px), but the wrap inside it took `height: 100%` of nothing —
it collapsed to its tallest child's content height (a 24px cached badge).
The badge was absolutely positioned, so the parent had 1147px of nothing
inside it.

**Fix:** Switch the wrap from `position: relative; height: 100%` to
`position: absolute; inset: 0`. This makes the wrap fill its parent
regardless of its own children.

```css
.chart-frame {
    position: absolute;  /* was: relative */
    inset: 0;             /* was: height: 100% */
    width: 100%;
    background: rgba(255, 255, 255, 0.008);
    border-radius: 16px;
    overflow: hidden;
}
```

Now the chart fills the parent no matter what content (badges, spinners) is
inside it. The badge stays `position: absolute` so it floats over the chart
instead of collapsing it.

### 2b. Sections/content covered by fixed-position background layers (DOM correct, paint wrong)

**Symptom:** DOM is perfect — `elementFromPoint(x, y)` returns the expected
card/heading at the visible scroll position, all `getComputedStyle` values
look right (display: block, opacity: 1, color: white, position: relative,
width matches viewport). But the rendered screenshot/page shows empty space
where the content should be. The fixed-position background layers
(`.bg-stars`, `.bg-grid`, an animated star field) paint OVER the scrolling
content.

**Why:** CSS stacking context rules. `z-index` only works on **positioned**
elements (position != static). Fixed-position background layers
(`position: fixed; z-index: 0`) form their own stacking context and paint on
top of static-positioned content. Even if the static content has `z-index: 1`
declared, the rule has no effect because z-index doesn't apply to static
elements. So the section content is in normal flow, and the fixed layers
paint on top of it.

This bit hard on a marketing landing page: DOM verified all 9 sections
rendered at correct positions with full opacity and white text, but every
screenshot after the hero showed empty space. The bug was a
`.section { position: relative; z-index: 1 }` rule that didn't match the
actual section class names (`.section-features`, `.section-audience`, etc.)
— so all sections were `position: static` with a no-op z-index, and the
`position: fixed` star/grid layers covered them.

**Test — confirm the diagnosis:**

```js
// After navigating to a page with fixed bg layers + content sections
const sections = Array.from(document.querySelectorAll('section, .content-block'));
const broken = sections.filter(s => {
  const cs = getComputedStyle(s);
  return cs.position === 'static' && cs.zIndex !== 'auto';
});
console.log('Sections with z-index but no positioning:', broken.length);
// If > 0, the rule isn't taking effect.

// Then probe what's covering the viewport
const stack = document.elementsFromPoint(195, 400);
console.log('Element stack at viewport center:', stack.map(e => e.tagName + '.' + (e.className?.toString().split(' ')[0] || '?')));
// If you see your bg layer (.bg-stars, .bg-grid) on TOP of section content,
// the fixed layers are covering the static content.
```

**Fix — `position: relative` (or any non-static value) on every content
section.** Use a single combined selector so it's impossible to forget new
sections:

```css
/* WRONG — z-index has no effect on static-positioned elements */
.section { z-index: 1; }
.section-features { padding: 120px 52px; }  /* position: static, default */

/* RIGHT — a single combined selector covering every visible section */
.section-hero, .section-features, .section-audience, .section-pricing,
.section-faq, .section-cta { position: relative; z-index: 1; }
.section-features { padding: 120px 52px; }  /* now positioned + z-index works */
```

**Discipline — non-negotiable:** whenever you add `<section class="X">` to a
landing page with fixed background layers, immediately add `.X` to that
combined selector in the same commit. The bug recurs silently — DOM looks
correct, computed styles look correct, but the fixed layers paint over the
static sections and the section becomes invisible on the live page.

**Diagnostic order:** always verify in CDP/Playwright that
`elementFromPoint(viewportCenterX, viewportCenterY)` returns your content
(not the background layer) when scrolled to a content section. If it returns
the background layer, that's the bug. If it returns content but the
screenshot is empty, that's a paint timing issue, not stacking.

This is a class of bug that ONLY manifests with fixed background layers
(decorative starfield, gradient mesh, animated canvas backdrop).
Pure-static-content pages never hit it. Test a page with
`body { background: red }` to verify whether the bug reproduces without the
fixed layers — if it goes away, stacking is the issue.

### 3. Chart captures `clientWidth`/`clientHeight` at mount before flexbox has measured

**Symptom:** Chart is rendered with the correct data (you can see the
watermark, the price scale, the time scale) but the candles/lines/bars are
missing. `getComputedStyle(canvas).width` and `height` are non-zero but the
data area is blank. `chart.timeScale().getVisibleLogicalRange()` returns
null or an empty range.

**Test:**
```js
const canvas = document.querySelector('.chart-frame canvas');
console.log('canvas w x h:', canvas?.width, 'x', canvas?.height);
console.log('clientW x clientH:', canvas?.clientWidth, 'x', canvas?.clientHeight);
// If width > 0, height > 0 but chart is empty → sizing mismatch
```

**Why:** Canvas libraries (`lightweight-charts`, `chart.js`, `recharts` via
canvas, `d3` with a canvas renderer) call `clientWidth`/`clientHeight`
synchronously when `createChart` runs. If the parent flexbox hadn't measured
yet (very common on initial mount), the values were 0, the chart was created
at 0x0, and all subsequent `setData` calls paint into a 0-area region.

**Fix:** Force a re-fit on the next animation frame, and again after
`setData`:

```tsx
useEffect(() => {
    const chart = createChart(containerRef.current, {
        width: containerRef.current.clientWidth,
        height: containerRef.current.clientHeight,
        // ...
    });
    // ... addSeries, setData, etc.

    // (a) Re-fit on next frame so flexbox has measured
    const rafId = requestAnimationFrame(() => {
        chart.applyOptions({
            width: containerRef.current.clientWidth,
            height: containerRef.current.clientHeight,
        });
    });

    // (b) Re-fit after every data load (in your fetch handler)
    setData(newData);
    chart.timeScale().fitContent();
    requestAnimationFrame(() => {
        chart.applyOptions({
            width: containerRef.current.clientWidth,
            height: containerRef.current.clientHeight,
        });
        chart.timeScale().fitContent();
    });

    return () => {
        cancelAnimationFrame(rafId);
        chart.remove();
    };
}, []);
```

**Variant — don't create the chart until the container is visible.** A chart
mounted inside a collapsed tab or an off-screen panel measures 0x0 no
matter how many re-fits you schedule, because the element genuinely has no
size. Defer chart creation until the container intersects the viewport
using `IntersectionObserver`:

```tsx
useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    let chart: IChartApi | null = null;
    const observer = new IntersectionObserver((entries) => {
        if (!entries[0].isIntersecting || chart) return;
        // First intersection: the element has real layout dimensions now.
        chart = createChart(container, {
            width: container.clientWidth,
            height: container.clientHeight,
            // ...
        });
        // ... addSeries, setData
        chart.timeScale().fitContent();
        observer.disconnect();
    }, { rootMargin: '200px' });

    observer.observe(container);
    return () => { observer.disconnect(); chart?.remove(); };
}, []);
```

This removes the whole class of "chart created before layout" bugs for
charts that live in tabs, accordions, or below-the-fold panels, and it
avoids paying for charts the user never scrolls to.

For lightweight-charts v5, use `addSeries(CandlestickSeries, options)` and
`series.setData(data)`. For recharts, use
`<ResponsiveContainer width="100%" height="100%">` — but wrap it in a flex
item with `min-height` (not `flex: 1` alone) so the parent doesn't grow to
fill the viewport.

### 3b. `flex: 1` on the chart wrapper overrides the min-height cap

**Symptom:** You set `.chart-region { min-height: 280px }` but the chart
renders at 1147px (the full viewport). `getComputedStyle(wrap).minHeight ===
'280px'` but `wrap.offsetHeight === 1147`. The min-height is being
respected, but something else is making the wrap grow.

**Why:** `flex: 1` on the chart's parent flex container makes it stretch to
fill all available vertical space. `min-height` is a *minimum* — it doesn't
cap growth. The parent has no `max-height` so it expands to fill the flex
parent's height.

**Fix:** Add `max-height` matching your min-height (or any value) and
`flex-shrink: 0` to keep the chart at a fixed size regardless of the flex
parent:

```css
.chart-region {
    flex: 1;
    min-height: 280px;
    max-height: 280px;   /* ← the cap */
    flex-shrink: 0;     /* ← don't let the flex parent shrink/grow it */
}
```

This pattern recurs in dashboards where the chart is inside a
`.dashboard-grid` → `.chart-panel` → `.chart-region` flex chain. Cap the
leaf node, not the intermediate flex items.

### 3c. Framer Motion `<motion.div>` overrides your CSS `translateX(-50%)` centering

**Symptom:** A pop-up, drawer, or tooltip with
`position: fixed; left: 50%; transform: translateX(-50%)` looks correctly
centered in the static CSS but appears shifted to the right when mounted via
`<motion.div animate={{ x: 0, y: 0, scale: 1 }}>`. The element sits at
`x = viewport-center` (left edge at the viewport midpoint) instead of being
centered around that point.

**Test:**
```js
const drawer = document.querySelector('.app-drawer');
const cs = getComputedStyle(drawer);
const rect = drawer.getBoundingClientRect();
console.log('pos:', cs.position, 'transform:', cs.transform, 'left:', rect.left);
// If transform doesn't include 'matrix' with -50% translation, motion is overriding it.
console.log('expected centered left:', (window.innerWidth - rect.width) / 2);
console.log('actual left:', rect.left, 'offset:', rect.left - (window.innerWidth - rect.width) / 2);
```

**Why:** Framer Motion owns the `transform` property. When it animates
`animate={{ x: 0, y: 0, scale: 1 }}`, it sets `transform: translate(0, 0)
scale(1)` inline, which clobbers your CSS `transform: translateX(-50%)`. The
element loses its horizontal centering.

**Fix:** Include `x: '-50%'` (or your desired centering) in every motion
variant — initial, animate, exit. Framer Motion then composes the centering
with your other animations:

```tsx
<motion.div
    className="app-drawer"
    initial={{ opacity: 0, y: 24, scale: 0.96, x: '-50%' }}
    animate={{ opacity: 1, y: 0, scale: 1, x: '-50%' }}
    exit={{ opacity: 0, y: 16, scale: 0.97, x: '-50%' }}
    transition={{ type: 'spring', stiffness: 480, damping: 32 }}
>
```

Then drop the redundant CSS `transform: translateX(-50%)` — motion owns the
transform property now. Leave a CSS comment explaining the inline transform.

This hit in production on the trading page: the drawer pop-up rendered
shifted right and content was clipped on mobile because Framer Motion was
clobbering the centering transform.

### 3d. lightweight-charts v5 typing — `time: number` doesn't satisfy `CandlestickData<Time>`

**Symptom:** `Argument of type 'ChartPoint[]' is not assignable to parameter
of type '(CandlestickData<Time> | WhitespaceData<Time>)[]'` even though your
data has `{ time, open, high, low, close, volume }` with all the right
fields. Build fails with `Type 'number' is not assignable to type 'Time'`.

**Why:** lightweight-charts v5 defines `Time` as `UTCTimestamp` (a branded
number type, not just `number`). Your external data type uses raw `number`
for `time`. TypeScript correctly flags the mismatch.

**Fix — cast on the setData call:**

```tsx
candleRef.current?.setData(
    data.map(({ time, open, high, low, close }) => ({
        time: time as Time,    // ← cast required for v5 typing
        open, high, low, close,
    }))
);
```

Or, if you export the candle-data type from your chart component, define it
with the `Time` cast up front:

```tsx
export interface ChartPoint {
    time: number;
    open: number;
    high: number;
    low: number;
    close: number;
    volume: number;
}
// Internal cast type:
interface CandleData {
    time: Time;
    open: number;
    high: number;
    low: number;
    close: number;
}
```

### 3e. `series.update()` requires the FULL OHLC, not just close

**Symptom:** You want to live-tick the last candle's close as the watchlist
price drifts. You call `series.update({ time, close: livePrice })` and either
nothing happens or the candle goes blank.

**Why:** lightweight-charts `series.update()` requires ALL of
open/high/low/close. There's no partial-update API. If you pass only time +
close, the library treats the missing fields as undefined and breaks the
rendering.

**Fix:** Store the full OHLC of the last candle in a `useRef`, then re-apply
with the new close + clamped high/low on every live-tick:

```tsx
const lastCandleRef = useRef<{time, open, high, low, close} | null>(null);

// On initial data load:
lastCandleRef.current = data[data.length - 1];

// On every live-tick:
useEffect(() => {
    if (livePrice == null || !candleRef.current || lastCandleRef.current == null) return;
    const prev = lastCandleRef.current;
    const updated = {
        time: prev.time as Time,
        open: prev.open,
        close: livePrice,
        high: Math.max(prev.high, livePrice),
        low: Math.min(prev.low, livePrice),
    };
    lastCandleRef.current = { ...prev, close: livePrice, high: updated.high, low: updated.low };
    candleRef.current.update(updated);
}, [livePrice]);
```

**Pair with single-source-of-truth for the displayed price:** drop the
`onLatestPrice` callback from the chart (it was racing the watchlist's drift
with a stale Polygon close). Let the watchlist be the only thing driving
the top-right hero price.

### 3f. Live tick doesn't roll candles on bar close — X-axis stuck

**Symptom:** On a `1m` (or `5m`, `1h`, `1d`) timeframe, the chart's X-axis
time label never advances. The "last candle" keeps updating its
close/high/low with each live-tick, but a new candle never appears at the
next period boundary. After 5 minutes on `1m`, only one candle is shown
instead of five.

**Why:** The standard live-tick pattern (root cause #3e) just updates the
in-progress last candle. It never detects "the period has ended — append a
new bar." The chart correctly shows OHLC for the current bar but the time
axis thinks time has stopped.

**Fix — detect period rollover, append a fresh candle:**

```ts
useEffect(() => {
    if (livePrice == null || !candleRef.current || lastCandleRef.current == null) return;
    if (lastTickPriceRef.current === livePrice) return;  // skip no-op ticks
    lastTickPriceRef.current = livePrice;

    const prev = lastCandleRef.current;
    const periodSeconds = TF_SECONDS[timeframe];  // e.g. 60 for 1m, 300 for 5m
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
        const updated = {
            time: prev.time as UTCTimestamp,
            open: prev.open,
            close: livePrice,
            high: Math.max(prev.high, livePrice),
            low: Math.min(prev.low, livePrice),
        };
        lastCandleRef.current = { ...prev, close: livePrice, high: updated.high, low: updated.low };
        candleRef.current.update(updated);
    }
}, [livePrice, timeframe]);
```

**The math:** `currentBarStart = floor(now / periodSeconds) * periodSeconds`.
For a `1m` timeframe at 14:32:47, `currentBarStart = 14:32:00`. If
`prev.time < 14:32:00`, the previous bar closed at the 14:31:00 boundary
and we're in a new bar.

**The open=prev.close trick:** without it, every new candle opens with the
watchlist's mid price, which causes a visible gap at the period boundary.
Anchoring `open` to `prev.close` makes the chart look continuous — the user
sees one smooth line instead of jumps.

**Don't forget:** skip no-op ticks
(`if (lastTickPriceRef.current === livePrice) return`). A watchlist that
polls every 10s would otherwise call `candle.update()` with identical data 6
times per minute — wasted redraws.

**Reset `lastTickPriceRef.current = null` when symbol or timeframe
changes** — otherwise the first tick on a new symbol might be skipped
because it equals the previous symbol's last tick.

### 3g. Chart 0×0 on first mount — single rAF isn't enough

**Symptom:** On initial page load, the chart canvas is blank. After 1-2
seconds of activity (window resize, layout shift), the chart suddenly
renders correctly. The chart was always wired up correctly — its container
just hadn't been measured when `createChart` ran.

**Why:** A single `requestAnimationFrame` re-fit fires once on the next
paint frame. But flexbox measurements can settle AFTER the first paint —
especially with SSR/hydration, dynamic imports, and conditional rendering.
The single rAF hits before the container has its final width/height.

**Fix — two-stage initial fit (rAF + setTimeout 50ms + setTimeout 250ms):**

```ts
useEffect(() => {
    if (!containerRef.current) return;
    const container = containerRef.current;
    const chart = createChart(container, {
        width: container.clientWidth,
        height: container.clientHeight,
        // ...
    });
    // ... addSeries, store refs

    const ro = new ResizeObserver(() => {
        if (containerRef.current && chartRef.current) {
            chartRef.current.applyOptions({
                width: containerRef.current.clientWidth,
                height: containerRef.current.clientHeight,
            });
            chartRef.current.timeScale().fitContent();
        }
    });
    ro.observe(container);

    // Two-stage initial fit: catches both fast (rAF) and late (setTimeout) flexbox measurements.
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

    return () => {
        cancelAnimationFrame(rafId);
        clearTimeout(t1);
        clearTimeout(t2);
        ro.disconnect();
        chart.remove();
        // ...
    };
}, []);
```

**Why three timeouts:** the rAF catches the immediate-next-paint case. The
50ms catches single-frame measurement settling. The 250ms catches slow
measurement (lazy-loaded images, a network round-trip on a font, etc.). The
`ResizeObserver` handles every subsequent resize including window resize and
mobile orientation.

**Always `fitContent()` in each fit:** calling `applyOptions` resizes the
canvas, but if the data was `setData`'d before the resize, the chart's time
scale might still think the visible range is "everything" with weird
padding. Calling `fitContent()` after every `applyOptions` tightens the
visible range to the actual data extent.

### 3h. Hardcoded "Cached · 30s" badge — UI lies about cache freshness

**Symptom:** The chart shows a "Cached · 30s" indicator in the corner even
when the cache is actually 24 hours old. Or the badge never appears even
though data IS cached. Or the badge says "Cached" on a fresh fetch (no
cache hit at all).

**Why:** The badge was added when the cache was 30s. The TTL changed to 24h
but the label was never updated. OR the badge logic checks `body.cached ===
true` but the route returns `cached` only for the cache miss fallback path,
not the happy-path "I fetched fresh and stored it."

**Fix — track source + age, not just "was cached":**

```ts
// Server side: include both flags + timestamp in the response
return NextResponse.json({
    results: liveResults,
    cached: chartSource !== 'live',
    source: chartSource,         // 'cache' | 'live' | 'stale' | 'stale-fallback'
    cached_at: chartsCache[cacheKey]?.cached_at ?? new Date().toISOString(),
    // ...
})

// Client side: derive a meaningful label from source + age
const cacheLabel = useMemo(() => {
    if (!cacheBadge.source) return null;
    if (cacheBadge.source === 'live') return null;  // hide for fresh fetches
    if (cacheBadge.source === 'stale-fallback') return 'Stale (different timeframe)';
    if (cacheBadge.ageMs != null) {
        const ageMin = Math.floor(cacheBadge.ageMs / 60_000);
        if (ageMin < 1) return `${cacheBadge.source} · <1m ago`;
        if (ageMin < 60) return `${cacheBadge.source} · ${ageMin}m ago`;
        return `${cacheBadge.source} · ${Math.floor(ageMin / 60)}h ago`;
    }
    return cacheBadge.source;
}, [cacheBadge]);
```

**The `source` enum (a broker-agnostic pattern):**

| Value | Meaning | UI |
|---|---|---|
| `'live'` | Fresh fetch, just stored to cache | Hidden (don't show badge for fresh) |
| `'cache'` | Served from cache, fresh (< TTL) | "cache · 5m ago" |
| `'stale'` | Served from cache, past TTL but within stale window | "stale · 2h ago" |
| `'stale-fallback'` | The broker returned 429, so a different cache key was served | "Stale (different timeframe)" + warning field |

**Don't show the badge for live fetches.** Showing "cached · 0m ago" when the
user just refreshed is noise. The badge should communicate "this data isn't
fresh" — fresh data doesn't need a badge.

## Triage script (run this first)

```js
(() => {
    const candidates = [
        '.chart-frame', '.chart-container', '.recharts-wrapper',
        '.chartjs-render-monitor', 'svg', 'canvas'
    ];
    const result = [];
    for (const sel of candidates) {
        const el = document.querySelector(sel);
        if (!el) continue;
        const cs = getComputedStyle(el);
        result.push({
            sel,
            tag: el.tagName,
            w: el.offsetWidth,
            h: el.offsetHeight,
            pos: cs.position,
            flex: cs.flex,
            maxH: cs.maxHeight,
            minH: cs.minHeight,
            display: cs.display,
        });
    }
    return JSON.stringify(result, null, 2);
})();
```

Read it top-down. For each row:
- **`w < 1100` AND `h < 250`** → canvas is too small. Look for: missing CSS
  import, parent with `flex: 1` and no `max-height`, or chart created before
  flexbox measured.
- **`pos: static` but you wrote `position: absolute`** → CSS not loaded
  (root cause #1).
- **`pos: absolute; inset: 0; h: 24`** → flex collapse (root cause #2).
- **`pos: absolute; h: 1000; canvas inside has w/h: 0`** → mount-time
  sizing (root cause #3).
- **DOM correct, screenshot empty, `elementFromPoint` returns bg layer over
  content** → fixed-position bg layers covering static-positioned content
  (root cause #2b).

## Related patterns

- **Charts that resize but never show data:** the chart was correctly sized
  but the data fetch returned an error. Check
  `chartContainer.querySelector('canvas')?.getContext('2d')` to see if any
  pixels were drawn. Also check the network tab for the upstream data API.
- **Chart flickers on re-render:** `setData` is being called inside a
  `useEffect` without a cleanup, so old chart instances pile up. Add
  `chart.remove()` to the cleanup.
- **Chart works on desktop but blank on mobile:** check that the chart has
  `min-width: 0` on its flex parent. Otherwise flexbox makes the chart's
  content determine its width, which can be 0 if the data hasn't loaded.
- **All page sections "blank" but DOM has everything:** check whether you
  have fixed-position background layers (`.bg-stars`, `.bg-grid`, animated
  star fields, decorative gradients) and whether your section content is
  `position: static`. The fixed layers will paint on top of static content
  even if the section has `z-index: 1` declared (z-index has no effect on
  static elements). Apply `position: relative` to all content sections.
- **Same bug for non-chart sections in a landing page** — the bug class is
  identical to the chart variant; the difference is only whether the
  covered element is a chart or a regular section.

## Reference

For a worked example of all three root causes hitting in sequence on the
same component, see `references/case-study-trading-chart-import.md`
(sequence of 8 debug rounds that ended with the chart working).

For a follow-up case study covering a different bug class — "the chart
renders but the data is wrong": live-tick data staleness, hardcoded cache
badge, bar rollover on timeframe change, mount-time sizing, and the
rate-limit cascade from a wrong cache key — see
`references/trading-chart-fixes-batch.md`. Same component, different bug
class.

For the Framer Motion + CSS transform conflict (a drawer or popup that
appears shifted right because motion clobbers the centering transform), see
`references/framer-motion-centering-conflict.md`.

For the case study where a multi-step wizard page was rewritten from
scratch and silently dropped the `import './setup.css'` line, producing a
40KB orphaned stylesheet and a "white text, no styling" report from the
user — see `references/case-study-missing-css-import.md`. Same bug class as
root cause #1 above, but for a non-chart page.
