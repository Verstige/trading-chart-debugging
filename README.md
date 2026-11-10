# Trading Charts That Render: Fix the Blank Canvas

> Fix trading dashboards that render wrong with a clean build. The class of bugs where the DOM is correct and the CSS is live, but the chart is blank, collapsed, or covered.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg) [![Available on Agensi](https://img.shields.io/badge/Agensi-%2412-ff5a5f.svg)](https://agensi.io/skills/trading-charts-that-render-fix-the-blank-canvas)

## What it does

Diagnose "the structure is right but the visuals are wrong" rendering bugs in trading dashboards — blank chart canvases, page sections that render unstyled, content hidden behind fixed-position background layers, pop-ups that appear shifted because animation overrides a CSS transform, and charts that render but show stale or wrong data. A class-level umbrella for the family of bugs where the build is clean, the DOM is correct, and computed styles prove your CSS rules are live while the page still looks broken. Covers the root causes that hit any Next.js App Router page or component with co-located CSS: a missing CSS import on the page or component (the build ships the React markup but never bundles the stylesheet), flexbox parents collapsing to their content's height (a chart, a scrollable panel, or any flex-column leaf that needs `min-height: 0`), fixed-position background layers painting over static-positioned sections (z-index has no effect on static elements), canvas libraries capturing `clientWidth`/`clientHeight` at mount before flexbox has measured, Framer Motion's `animate={{ x: 0 }}` clobbering a CSS `transform: translateX(-50%)` centering rule, lightweight-charts v5 typing and the `series.update()` full-OHLC requirement, period-rollover detection for live ticks, and stale "Cached" badges that don't track the underlying TTL. Use when a chart shows nothing but the watermark, when data flows but no shapes draw, when a page section renders as raw browser defaults despite a sibling CSS file existing, when `elementFromPoint` returns a background layer over your content, when `position: absolute` doesn't apply because the stylesheet isn't loaded, when a flex leaf collapses to its tallest child's height, when the last candle is frozen while the price ticker moves, or when any Next.js render looks "wrong but the build is clean.

When a chart shows data flowing in the UI (price ticks, watchlist updates,
performance metrics) but the chart canvas itself is blank, the bug is in the
**layout chain around the canvas**, not in the chart library. There are four
recurring root causes. Run this checklist in order.

## The failure it prevents

Each reference file is one specific failure: the symptom, the actual cause, and the fix. The ones that come up most often:

- Case Study: A Page Rewritten From Scratch Dropped Its CSS Import
- Case Study: Trading Chart Blank Canvas
- Framer Motion vs CSS `translateX(-50%)` Centering Conflict
- Case Study: Trading Chart Live-Tick + Cache UX

## When to use it

- fix a trading chart that renders blank or wrong
- diagnose a dashboard section that shows raw browser defaults
- solve layout bugs where the build is clean and the DOM is correct

## What's inside

```
plugins/
└── trading-chart-debugging/
    ├── .claude-plugin/plugin.json
    └── skills/trading-chart-debugging/
        ├── SKILL.md
        ├── references/            (4 files)
        │   └── case-study-missing-css-import.md
        │   └── case-study-trading-chart-import.md
        │   └── framer-motion-centering-conflict.md
        │   └── trading-chart-fixes-batch.md
```

## Install

Add this repo as a Claude Code marketplace, then install the skill:

```bash
claude plugin marketplace add Verstige/trading-chart-debugging
claude plugin install trading-chart-debugging@trading-chart-debugging
```

Or copy the skill straight into your skills directory:

```bash
git clone https://github.com/Verstige/trading-chart-debugging.git
cp -R trading-chart-debugging/plugins/trading-chart-debugging/skills/trading-chart-debugging ~/.claude/skills/
```

## Usage

The skill activates on the work it describes. Ask for the outcome, and it
supplies the method:

```
> "fix a trading chart that renders blank or wrong"
```

## Reference material

Loaded on demand — the skill only reads what the task needs.

| File | Covers |
| --- | --- |
| `case-study-missing-css-import.md` | Case Study: A Page Rewritten From Scratch Dropped Its CSS Import |
| `case-study-trading-chart-import.md` | Case Study: Trading Chart Blank Canvas |
| `framer-motion-centering-conflict.md` | Framer Motion vs CSS `translateX(-50%)` Centering Conflict |
| `trading-chart-fixes-batch.md` | Case Study: Trading Chart Live-Tick + Cache UX |

## License

MIT. See [LICENSE](LICENSE).

---

**Also available as a paid skill on Agensi** — $12. Install it with one command, no clone required:

[agensi.io/skills/trading-charts-that-render-fix-the-blank-canvas](https://agensi.io/skills/trading-charts-that-render-fix-the-blank-canvas)

Part of the [Verstige](https://github.com/Verstige) skills portfolio. More at [github.com/Verstige](https://github.com/Verstige).
