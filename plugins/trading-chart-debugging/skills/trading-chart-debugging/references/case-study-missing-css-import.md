# Case Study: A Page Rewritten From Scratch Dropped Its CSS Import

A production incident: a multi-step wizard page was rewritten from scratch,
the `import './setup.css'` line was silently dropped, and a fully-built CSS
file shipped that never entered the bundle. The user reported "white text,
no CSS styling."

Same bug class as `case-study-trading-chart-import.md`, but for a non-chart
page. Use this when a Next.js page renders with raw browser defaults after a
from-scratch rewrite, when the build is clean but classes don't compute, or
when a co-located CSS file exists but appears to have no effect at runtime.

## What happened

`app/setup/page.tsx` was rewritten from scratch to consolidate an 8-step
multi-page flow into a single-page 6-step wizard (~37k chars of JSX, ~2.5k
CSS added later). The replacement file did NOT include the
`import './setup.css'` line that the original page had. The existing
`app/setup/setup.css` file (40KB of styles) sat untouched on disk. Built
cleanly. Shipped.

## What the user saw

The user logged in and reported:

> "I still believe there is a TypeScript error that I need you to review because all I'm seeing is white texts without any CSS styling So can you please review the page and make sure it's working properly and Also everything that we just put together and set up is also rendering correctly in the system."

The page rendered with raw browser defaults: white text on a dark
background, no layout grid, no colors, no card borders, no progress bar
fill, no animations. Every styled class was a no-op. The DOM was correct —
the React component tree was intact, hooks ran, data flowed — but every
`className` was a bare class with no matching rule.

## Diagnostic round (one round, took 2 minutes)

1. Searched the page file for any `import` of a `.css` file → **none**.
   Confirmed.
2. Listed the directory → 40KB `setup.css` exists, untouched.
3. Ran the project's type check (`npx tsc --noEmit`) → exit 0, zero errors.
4. Ran the production build → clean.
5. Fetched the deployed page → 2 CSS files referenced in the HTML, neither
   contained the `.wizard-*` classes (verified by grepping the served CSS).
6. Searched the built CSS chunks for `.wizard` → 1 match in the
   post-fix deployment's 26KB chunk. The pre-fix chunks had no `wizard`
   classes at all.

## Root cause

Rewriting a page from scratch (or with any template-based generator)
silently drops import lines that the original had, **if you don't preserve
them in your new content**. The CSS file is unchanged on disk. The build
still emits the page chunk. But the bundler only includes a CSS file in the
emitted CSS chunks if something — *anything* — transitively imports it from
a loaded page. With no importer, the CSS is dead code.

This is structurally identical to the chart-importer-missing bug (root cause
#1 in the main SKILL.md) — same bug class, different file type.

## Fix

Added the import line to all four pages (one line each):

```diff
// app/setup/page.tsx
  import { apiFetch } from '@/lib/api-fetch'
+ import './setup.css'

// app/setup/profile/page.tsx
  import { apiFetch } from '@/lib/api-fetch'
+ import '../setup.css'

// app/setup/connect/page.tsx
  import { apiFetch } from '@/lib/api-fetch'
+ import '../setup.css'

// app/setup/notifications/page.tsx
  import { apiFetch } from '@/lib/api-fetch'
+ import '../setup.css'
```

The remaining legacy pages are now redirect shims (`router.replace('/setup')`),
so they don't render any styled markup and don't need their own CSS imports.

## Verification

| Check | Before | After |
|---|---|---|
| Type check clean | ✅ | ✅ |
| Build clean | ✅ | ✅ |
| `.wizard-*` classes appear in built CSS chunks | ❌ none | ✅ one 26KB chunk |
| Production chunk includes the expected layout rules | ❌ | ✅ |
| User-visible styling | ❌ raw browser defaults | ✅ intended design |

## Discipline going forward

When you rewrite a Next.js page or component file in a directory that
contains sibling CSS files, the FIRST thing you do before composing the JSX
body is list the sibling `.css` files and write down which ones you need to
import. If your fresh page content is going to use any className from those
CSS files, the import line must be the FIRST content line after the
`import { foo } from 'bar'` block — before any JSX, before any
component-level code.

This applies to:
- Page rewrites performed by a write-file tool or a template-based
  generator
- Page rewrites via `patch` if the patch accidentally removes a top-of-file
  import block
- Creating new sibling component files (always pair `.tsx` + `.css` import on
  creation, not as a follow-up step)
- Splitting a component out of a parent file (the new file won't have the
  parent's CSS imports)

The cost of the bug is invisible until a user opens the page. The CSS file
is intact, the build is clean, the smoke test passes — none of the checks a
session typically runs surfaces it. The type checker's exit code is the
strongest automated signal available, and it does NOT catch this. The only
hard guarantee is "if there's a sibling `.css`, your page must import it
before first paint."

## Why this will recur

A write-file tool doesn't have a "preserve these import lines from the
original" mode. There is no Next.js lint rule that flags "your page is
styled but CSS exists in the directory that wasn't imported." The natural
workflow — "rewrite a big file from scratch with cleaner structure" — is a
perfect environment for this bug. Each of the four sibling pages
(profile/connect/notifications) had this same missing import, even though
those pages weren't rewritten — they were created from scratch in the same
session, so they had never had the import line in the first place.

The only defense is the "**before you write, check what sibling CSS exists
and plan the import list**" pre-flight step. Add it to your workflow
checklist explicitly.
