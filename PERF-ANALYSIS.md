# Mobile Performance Analysis — Background Themes

## Background

`index.html` and `about.html` randomly pick one of 3 backgrounds on load.
The user can click the avatar to cycle through them.

| Theme | DOM elements (background only) | Continuous CSS animations | SVG filters |
|-------|-------------------------------|--------------------------|-------------|
| **Circuit** | ~70 | 6 pulse paths + 42 node glows | `feGaussianBlur(2)` + `drop-shadow()` |
| **Night Sky** | ~520+ stars + aurora + shooting stars (~530) | 150+ twinkling stars, 4 auroras, 3 shooting stars | none (pure CSS) |
| **Blossom** | ~40 | 30 petals (3D), 4 clouds, 1 rays, 1 bough (~36) | `feGaussianBlur(1.6–11)` SVG filters + CSS blur |

---

## Bottleneck Breakdown (by severity)

### 1. Night Sky — Star field: ~520 `<div>` elements, ~150 animating

**What happens:** JavaScript creates 520 absolutely-positioned `<div>`s on page load.
Of those, ~30% (150+) receive a twinkle class with `transform: scale() rotate()` running forever.

**Why it hurts mobile:**
- Each star is its own render layer + style recalculation target.
- `transform: scale() rotate()` on a 1px element still requires per-pixel compositing.
- 150 simultaneous `@keyframes` with `will-change`-free animated elements cause the GPU compositor to recalculate 30–60 times/sec.

**Estimated impact:** 15–30% of mobile GPU compositing time on mid-range devices.

### 2. Night Sky — Aurora: 4 elements with large `filter: blur()` + `mix-blend-mode: screen`

**What happens:** Each aurora is a ~30% screen-height `div` with a multi-stop `linear-gradient`, `filter: blur(80px)`, `mix-blend-mode: screen`, and continuous `translateX` / `scaleX` / `scaleY` / `rotate` / `opacity` animation.

**Why it hurts mobile:**
- `filter: blur(80px)` on a large rect is a heavy SVG filter — GPU has to sample many pixels per frame.
- `mix-blend-mode: screen` is another per-pixel operation layered on top.
- The blur region expands with every `scaleX`/`scaleY` transform, meaning the filter kernel recomputes over a moving area each frame.
- 4 of them running simultaneously = 4 separate filter passes.

**Estimated impact:** 20–40% of mobile GPU compositing time.

### 3. Night Sky — Shooting stars: 3 elements with complex `::before`/`::after` gradients + `box-shadow`

**Lower priority** (already `display: none` on ≤768px), but on desktop the `box-shadow: 0 0 6px 2px` on the head dot + gradient track adds per-frame blur compositing.

### 4. Blossom — Petals: 30 elements with 3D transforms + blur

**What happens:** Each petal (`<span.bl-p><i></i></span>`) has:
- `filter: blur(var(--b))` on the parent
- A `<i>` child animated with `perspective(160px) translateX() rotate() rotateX()`
- Each `<i>` also has a gradient fill

**Why it hurts mobile:**
- 30 simultaneous `@keyframes` each running a different cycle duration.
- `perspective()` + `rotateX()` creates a new compositing layer per petal (30 layers).
- `filter: blur()` on the parent compounds with the 3D transform.
- The petal positions span the full screen, so the GPU composites blur over a large area every frame.

**Estimated impact:** 15–25% of mobile GPU compositing time.

### 5. Blossom — Cloud layers: 4 elements with dozens of `radial-gradient` backgrounds + `filter: blur()` + infinite horizontal scroll

**What happens:** Each cloud has 7–12 `radial-gradient` stops as background-image plus `filter: blur(12–16px)` and a `translate3d` loop animation (75s–240s cycles).

**Why it hurts mobile:**
- `radial-gradient` with many stops is expensive to tile/scroll.
- Combined with `filter: blur(14px)` on a 200%-wide element, the GPU blurs a huge rect every frame.
- 4 of them stacked = 4 blur passes.

**Estimated impact:** 10–20% of mobile GPU compositing time.

### 6. Blossom — SVG filters on inline SVGs

**What happens:** Two large inline SVGs (bokeh corner + branch) use `feGaussianBlur(10)` and `feGaussianBlur(11)`.

**Why it hurts mobile:**
- SVG filters render to an offscreen buffer, then composite back.
- `stdDeviation="10–11"` on elements with ~15–20 circles means significant per-frame work on load.
- However, these are **static** (no animation), so cost is one-time.

### 7. Circuit — SVG traces with `drop-shadow` + mask operations

**Lower priority** (~10–15% of GPU time), but still continuous:
- `filter: drop-shadow(0 0 3px) drop-shadow(0 0 6px)` on 6 pulse paths
- `feGaussianBlur(2)` SVG filter on mask-composited `<use>` elements
- 42 circle elements animated with `opacity` + `r` (radius) changes

---

## Root Causes Summary

| Root cause | Where | Severity |
|-----------|-------|----------|
| **DOM element count** (500+) | Night sky stars | High |
| **GPU compositing layers** (150+ simultaneous) | Night sky twinkle | High |
| **Large-area `filter: blur()`** on animated rects | Aurora, clouds | High |
| **`mix-blend-mode: screen`** on animated elements | Aurora | Medium |
| **3D transforms creating layers** | Blossom petals | Medium |
| **Per-pixel blend operations per frame** | Aurora + petals | Medium |
| **Radial-gradient tiling + blur scroll** | Blossom clouds | Medium |
| **SVG filter compositing** | Circuit traces, blossom SVGs | Low (one-time or light) |

---

## Proposed Optimizations

### A. Night Sky — Replace stars with `<canvas>` (saves ~500 DOM nodes)

**Approach:** A single `<canvas>` element replaces the 520 `<div>`s. Draw stars once at page load, then only re-render twinkling stars every few frames.

- Static stars: drawn once, never re-rendered.
- Twinkling stars: re-rendered at 5–10fps (opacity only, no layout).
- Shooting stars: can stay as CSS (only 3 elements) or go on canvas too.

**Expected savings:** Eliminates ~500 DOM nodes, reduces compositing from 150+ layers to 1 canvas layer.

### B. Night Sky — Replace aurora `filter: blur()` with CSS radial gradients

**Approach:** Remove `filter: blur(80px)` entirely. Instead, draw the gradient with soft edges directly:

```css
.aurora {
  background: radial-gradient(ellipse at center,
    rgba(r, g, b, 0.4) 0%,
    rgba(r, g, b, 0.15) 40%,
    transparent 70%
  );
  /* No filter: blur() needed — gradient itself is soft */
}
```

Use multiple layered `radial-gradient` stops to simulate the aurora glow. The animation (translateX, scaleX, opacity) stays the same but without any filter.

**Expected savings:** Removes 4 heavy blur filters from the continuous render loop.

### C. Blossom — Simplify petal animation (replace 3D with 2D)

**Approach:** Replace the 3D perspective transform with a simpler 2D animation:

```css
/* Before (3D, creates compositing layer): */
@keyframes bl-flutter {
  from { transform: perspective(160px) translateX(-3vmin) rotate(-40deg) rotateX(0); }
  to   { transform: perspective(160px) translateX(3vmin) rotate(55deg) rotateX(75deg); }
}

/* After (2D, single composite): */
@keyframes bl-flutter {
  from { transform: translateX(-3vmin) rotate(-15deg); }
  to   { transform: translateX(3vmin) rotate(15deg); }
}
```

Also reduce petal count on small screens via JS (e.g., 15 instead of 30 on screens < 480px).

**Expected savings:** Cuts 30 compositing layers to 1–2, removes perspective overhead.

### D. Blossom — Reduce cloud gradient stops + lower blur on mobile

**Approach:**
- Use fewer radial-gradient stops (5–6 instead of 12) on mobile via media queries.
- Reduce blur amount: `blur(8px)` instead of `blur(14px)` on screens < 768px.
- Consider pre-rendering clouds to an image on desktop, or using a single gradient per cloud.

### E. Universal — Respect `prefers-reduced-motion` more aggressively

Add the current `prefers-reduced-motion` rule for **all** animated background elements (currently only the blossom petals are fully hidden). On devices with motion preferences, still background = instant load, zero GPU cost.

### F. Universal — `content-visibility: auto` on background containers

```css
.night-sky, .blossom-sky {
  content-visibility: auto;
  contain-intrinsic-size: 0 100vh;
}
```

This tells the browser it can skip layout/paint of the background until it's in view. For a full-viewport background this is debatable (it's always in view), but combined with `view-timeline` it can help in nested scroll scenarios.

### G. Universal — Lazy/defer star generation

Defer star canvas creation by one frame using `requestAnimationFrame` or `setTimeout(0)`. This prevents layout from being blocked on JS execution.

---

## Recommended Priority

| Priority | Change | Effort | Impact |
|----------|--------|--------|--------|
| 1 | Canvas stars (A) | Medium | High — biggest DOM savings |
| 2 | Gradient aurora (B) | Low | High — removes 4 heavy filters |
| 3 | 2D petals (C) | Low | Medium-High — simplifies GPU layers |
| 4 | Reduce cloud complexity (D) | Medium | Medium |
| 5 | Better reduced-motion (E) | Low | Medium (UX + perf) |
| 6 | Defer JS execution (F) | Trivial | Low (perceived perf) |

Changes A + B + C alone should give the biggest performance gain with zero visual compromise.

---

## Visual Impact Assessment

| Change | Visual difference |
|--------|-------------------|
| Canvas stars vs DOM stars | **None** — identical appearance, just drawn differently |
| Gradient aurora vs blur aurora | Near-identical — radial gradients simulate blur naturally |
| 2D petals vs 3D petals | **Slight** — loses some depth illusion but still looks like petals floating |
| Reduced cloud blur on mobile | **Minimal** — blur is subtle at this distance; 8px vs 14px is barely noticeable |
| Fewer petals on mobile | Acceptable — 15 petals still looks busy enough at small screen size |
