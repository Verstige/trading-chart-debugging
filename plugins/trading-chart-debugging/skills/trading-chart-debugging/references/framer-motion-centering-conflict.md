# Framer Motion vs CSS `translateX(-50%)` Centering Conflict

## Symptom

A pop-up, drawer, modal, tooltip, or any `<motion.div>` with
`position: fixed; left: 50%; transform: translateX(-50%)` renders **shifted
to the right** when mounted via Framer Motion. The element sits at
`x = viewport-center` (its left edge is at the viewport midpoint) instead of
being centered around that point. On mobile, content gets clipped on the
right side and the user can't see it.

## The exact failure mode

CSS alone works fine:

```css
.app-drawer {
    position: fixed;
    bottom: 110px;
    left: 50%;
    transform: translateX(-50%);   /* centers around viewport midpoint */
    width: 640px;
}
```

When you mount this via `<motion.div>`:

```tsx
<motion.div
    className="app-drawer"
    initial={{ opacity: 0, y: 24, scale: 0.96 }}
    animate={{ opacity: 1, y: 0, scale: 1 }}
    exit={{ opacity: 0, y: 16, scale: 0.97 }}
>
```

Framer Motion takes ownership of the `transform` property. It sets
`transform: translate(0px, 0px) scale(1)` inline, which **overrides the CSS
`transform: translateX(-50%)`**. The `-50%` horizontal shift is lost. The
element snaps back to `left: 50%` (its CSS position) WITHOUT the translation
that would re-center it.

Result: the left edge sits at viewport-center, not centered around
viewport-center.

## Diagnostic recipe

```js
const drawer = document.querySelector('.app-drawer');
const cs = getComputedStyle(drawer);
const rect = drawer.getBoundingClientRect();
console.log({
    position: cs.position,                       // 'fixed'
    transform: cs.transform,                     // 'matrix(...)' — Framer Motion output
    left: rect.left,                             // x coordinate of left edge
    expectedCenteredLeft: (window.innerWidth - rect.width) / 2,
    actualOffset: rect.left - (window.innerWidth - rect.width) / 2,
});
```

If `actualOffset` is `rect.width / 2` (half the drawer width), motion has
clobbered the centering. If `actualOffset` is 0 or near 0, the centering is
working.

## The fix — preserve centering inside motion variants

```tsx
<motion.div
    className="app-drawer"
    initial={{ opacity: 0, y: 24, scale: 0.96, x: '-50%' }}  // ← include x: '-50%'
    animate={{ opacity: 1, y: 0, scale: 1, x: '-50%' }}       // ← in every variant
    exit={{ opacity: 0, y: 16, scale: 0.97, x: '-50%' }}
    transition={{ type: 'spring', stiffness: 480, damping: 32 }}
>
```

Then drop the redundant CSS `transform: translateX(-50%)`. Framer Motion owns
the transform now. Leave a comment in the CSS explaining why the transform
is missing:

```css
.app-drawer {
    position: fixed;
    bottom: 110px;
    left: 50%;
    /* transform is set inline by Framer Motion to translateX(-50%)
       so the drawer is centered relative to the viewport */
    z-index: 80;
    width: min(640px, calc(100vw - 32px));
    /* ... */
}
```

## Variants of the same problem

- `transform: translate(-50%, -50%)` for vertical+horizontal centering →
  use `x: '-50%', y: '-50%'`
- `transform: translateY(20px)` for hover effects → use
  `whileHover={{ y: 20 }}` or `animate={{ y: 20 }}`
- Custom transforms on tooltips, popovers, modals, centered cards → always
  declare them in the motion variants

## General rule

**If a centered element uses Framer Motion animations on the transform
property, the centering transform MUST be part of the motion variants** — not
in CSS — or motion will override it.

## Verified

This bit in production on the trading page: the app-drawer pop-up
(`position: fixed; left: 50%; transform: translateX(-50%)`) appeared shifted
right and content was clipped on mobile because Framer Motion was overriding
the CSS centering transform. Fixed by moving the centering into the motion
variants.

Diagnostic before fix:
- `viewportWidth: 1280, drawerLeft: 640, drawerRight: 1280` (right-edge of
  viewport, content clipped)
- `centeredOffset: 320` (drawer is half its width past center)

Diagnostic after fix:
- `viewportWidth: 1280, drawerLeft: 320, drawerRight: 960, drawerWidth: 640`
- `centeredOffset: 0` (perfectly centered)
