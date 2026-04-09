# Magnetic Pull Effect — Implementation Notes

A cursor-tracking effect where elements shift toward the mouse within a radius, then spring back. Works on both small chips (62px icons) and large scaled content cards.

---

## What Finally Worked

**The core insight: stop using RAF + lerp. Use CSS `transition` + direct `transform` on `mousemove` instead.**

Every attempt with `requestAnimationFrame` loops and lerp easing failed — too many moving parts (state bugs, timing issues, scale miscalculations). The CSS transition approach is dead simple and the browser handles all smoothing.

---

## The Pattern

### 1. Add CSS transition to the target elements

```css
.tool-chip {
  transition: transform 0.2s cubic-bezier(.25,.46,.45,.94), box-shadow 0.2s ease;
}
```

For dynamically-created or non-CSS targets, set it in JS:
```js
items.forEach(el => {
  el.style.transition = 'transform 0.25s cubic-bezier(.25,.46,.45,.94)';
});
```

### 2. The JS handler (no RAF, no lerp)

```js
(function () {
  const container = document.getElementById('tools');
  if (!container) return;
  const chips = Array.from(container.querySelectorAll('.tool-chip'));
  if (!chips.length) return;

  const RADIUS = 150;    // px — activation radius around each element center
  const STRENGTH = 0.55; // multiplier on the displacement (tune for feel)
  let centers = [];

  // Step 1: cache centers on mouseenter BEFORE any transforms are active
  container.addEventListener('mouseenter', () => {
    centers = chips.map(c => {
      const r = c.getBoundingClientRect();
      return { x: r.left + r.width / 2, y: r.top + r.height / 2 };
    });
  });

  // Step 2: apply transform directly on mousemove — CSS transition smooths it
  container.addEventListener('mousemove', e => {
    if (!centers.length) return;
    const mx = e.clientX, my = e.clientY;
    chips.forEach((chip, i) => {
      const dx = mx - centers[i].x;
      const dy = my - centers[i].y;
      const d  = Math.hypot(dx, dy);
      if (d < RADIUS) {
        const f = 1 - d / RADIUS;  // linear falloff — see note below
        chip.style.transform = `translate(${(dx * f * STRENGTH).toFixed(2)}px,${(dy * f * STRENGTH).toFixed(2)}px)`;
      } else {
        chip.style.transform = '';
      }
    });
  });

  // Step 3: clear transforms on mouseleave — CSS transition springs them back
  container.addEventListener('mouseleave', () => {
    centers = [];
    chips.forEach(c => { c.style.transform = ''; });
  });
})();
```

---

## Falloff Formula

Two options. Choose based on desired feel:

### Quadratic: `f = (1 - d / RADIUS) ** 2`
- Falls off fast — effect is concentrated near the element center
- Feels precise and tight
- **Problem**: chips near container edges barely react — effect seems weak at edges

### Linear: `f = 1 - d / RADIUS`
- Gentler falloff — chips further away still get significant pull
- At d=80px, linear gives ~4× more force than quadratic
- **Recommended** — makes the whole container feel reactive, not just the center

Force at various distances (RADIUS=150, STRENGTH=0.55):

| d (px) | linear f | move (px) |
|--------|----------|-----------|
| 20     | 0.87     | 9.6       |
| 50     | 0.67     | 18.4      |
| 100    | 0.33     | 18.2      |
| 130    | 0.13     | 9.3       |

Note: movement peaks around d = RADIUS/2, not at d=0 (because at d=0, dx=dy=0 → no movement).

---

## Applying to Elements Inside a CSS `scale()` Parent

If the target elements live inside a parent with `transform: scale(N)`, CSS `transform: translate(Xpx)` on a child moves it `X * N` viewport pixels — not `X` viewport pixels.

**Fix: divide the calculated offset by the parent's scale factor.**

```js
const SCENE_SCALE = 0.499; // parent's transform scale

// In the mousemove handler:
if (d < RADIUS) {
  const f = 1 - d / RADIUS;
  const tx = (dx * f * STRENGTH) / SCENE_SCALE;  // compensate for parent scale
  const ty = (dy * f * STRENGTH) / SCENE_SCALE;
  el.style.transform = `translate(${tx.toFixed(2)}px,${ty.toFixed(2)}px)`;
}
```

`getBoundingClientRect()` already returns viewport-space coords (after all transforms), so the distance calculation is correct. Only the applied transform needs the scale compensation.

---

## Key Gotchas

### Don't cache centers after transforms have been applied
Cache `getBoundingClientRect()` on `mouseenter` — before any transforms run. If you cache mid-animation, you get shifted centers → the effect either over- or under-corrects in a feedback loop.

### Remove conflicting CSS hover transforms
If `.element:hover { transform: scale(1.02) }` exists, the CSS and JS transforms fight each other. The inline style wins, but on mouseleave the CSS hover can snap back unexpectedly. Remove any transform from CSS hover rules; use only `box-shadow` / `background` for hover feedback.

### `will-change: transform` + `backdrop-filter: blur()` = GPU layer conflict
Combining both on the same element can cause compositor bugs in Chromium. Drop `will-change: transform` when `backdrop-filter` is present.

### `overflow: hidden` on the container clips movement
If the parent container has `overflow: hidden` and the element moves beyond the container's painted bounds, the movement is invisible. Either:
- Keep movement small enough to stay within the padding of the container
- Or remove `overflow: hidden` from the parent and add it only where needed

### Linear falloff vs. quadratic for edge reactivity
If users notice "the edges feel weaker," switch from `** 2` to linear `(1 - d/R)`. Also increase RADIUS so elements at the far ends of the container are still within range.

---

## Tuning Guide

| Parameter   | Effect                                           | Start value |
|-------------|--------------------------------------------------|-------------|
| `RADIUS`    | How far away the cursor activates a chip         | 130–160px   |
| `STRENGTH`  | Max displacement as a fraction of distance       | 0.5–0.7     |
| `transition`| How fast the spring-back animates                | 0.15–0.25s  |

For **large cards** (hundreds of px) inside a scaled parent:
- Use RADIUS 250–350 viewport px
- STRENGTH 0.35–0.45
- Divide transform by SCENE_SCALE

For **small chips** (50–70px):
- RADIUS 130–160px
- STRENGTH 0.5–0.65
- No scale compensation needed (usually)
