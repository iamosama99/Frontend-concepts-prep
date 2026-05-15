# Reflow vs. Repaint vs. Composite-only Changes

## Quick Reference

| Operation | What changes | Pipeline stages triggered | Cost |
|---|---|---|---|
| Reflow (layout) | Geometry — size, position, structure | Layout → Paint → Composite | High — can affect entire tree |
| Repaint | Visual appearance without geometry | Paint → Composite | Medium — only affected layers |
| Composite-only | `transform`, `opacity` | Composite only | Low — runs on GPU, off main thread |

## What Is This?

Once the initial page renders, any visual change the browser makes must go back through some subset of the rendering pipeline. The cost of that change is determined by how far back in the pipeline it reaches.

- **Reflow (layout):** Any change that affects geometry — where elements are, how big they are, how they affect surrounding elements. This propagates: changing one element's width can shift everything around it, requiring the browser to recalculate positions for a large portion of the page.
- **Repaint:** Any change that affects visual appearance without altering geometry — color, background, visibility, shadow, outline. The element re-draws itself (and potentially the layers it belongs to), but no positions need recalculating.
- **Composite-only:** Changes to `transform` and `opacity` that operate on already-painted layers. The GPU repositions or blends a pre-painted texture — no new drawing, no geometry recalculation.

The cost hierarchy is steep: reflow can be 10–100x more expensive than a composite-only change.

> **Check yourself:** An element's background color changes from red to blue. Does this trigger a reflow? Why or why not?

## Why Does It Exist?

The browser's rendering pipeline is designed around the insight that different kinds of visual changes have different invalidation scopes. Changing an element's color doesn't affect its neighbors' positions — there's no reason to recompute layout. The pipeline stages are semi-independent so the browser can skip earlier stages when only later-stage properties change.

This design decision is what makes 60fps animations achievable on the web — but only if you respect it. Animating `width` forces reflow on every frame. Animating `transform: translateX()` operates on a GPU-side texture and never touches layout. Both produce a moving element, but the performance profiles are completely different.

## How It Works

### Reflow (Layout Invalidation)

Reflow is triggered when the geometry of the document changes. The browser must recalculate:
- The position and size of every affected element
- The layout of their children and potentially their siblings
- How changes cascade up to ancestor elements (a child growing can push its parent to grow too)

Properties that trigger reflow:
```
Dimensions:  width, height, min/max-width/height, padding, margin, border-width
Position:    top, left, right, bottom, position, float, clear
Display:     display, overflow, flex/grid properties
Text:        font-size, font-weight, font-family, line-height, text-align
Structure:   adding/removing DOM nodes, changing element type
```

The scope of reflow depends on what changed:
- The entire document can reflow for a change to `<body>` dimensions
- A portion of the page reflows when an inner element changes
- An isolated subtree reflows when a contained element changes (if using `contain: layout`)

### Repaint (Paint Invalidation)

Repaint occurs when an element's visual appearance changes but its geometry doesn't. The browser skips layout and goes directly to the paint step for the affected elements/layers.

Properties that trigger repaint (not reflow):
```
Colors:       color, background-color, border-color, outline-color
Visibility:   visibility, opacity (partially — see below)
Appearance:   border-radius, box-shadow, text-shadow, background-image
Gradients, images (src change)
```

Repaint is cheaper than reflow because no positions shift. But it's not free — painting is CPU work that produces rasterized pixels, and complex visual effects (shadows, gradients, filters) are expensive to paint.

### Composite-only Changes

`transform` and `opacity` are special. When applied to an element that's been promoted to its own compositor layer, the compositor can reposition (transform) or fade (opacity) the pre-painted layer texture without any main-thread involvement.

```css
/* Composite-only — only runs on compositor thread */
.animated {
  transform: translateX(100px);   /* moves the layer */
  opacity: 0.5;                   /* blends the layer */
}

/* Triggers reflow — geometry change */
.animated {
  left: 100px;                    /* repositions in document flow */
  width: 50%;                     /* changes element geometry */
}
```

The GPU stores the layer as a texture. Moving it (`transform: translateX`) is just updating a matrix — the texture doesn't change. Fading it (`opacity`) is alpha blending. Both happen in the compositor, off the main thread, and can run at 60fps even when the main thread is executing JavaScript.

> **Check yourself:** `opacity: 0` and `visibility: hidden` both hide an element. Which one triggers a repaint when changed? Does either trigger a reflow?

### Layer Promotion

For transform/opacity to be composite-only, the element must be on its own compositor layer. If it shares a layer with other elements, changing its transform would require re-compositing that shared layer, potentially triggering a repaint.

The browser promotes elements to their own layers when:
- `will-change: transform` (or `opacity`) is set
- The element has a CSS animation on `transform` or `opacity`
- The element uses `position: fixed`
- The element is a `<video>` or `<canvas>`
- The element has certain 3D transforms

```css
/* Promote before the animation starts to avoid a paint on first frame */
.card {
  will-change: transform;
}
/* Now card has its own layer, and transform changes are composite-only */
.card:hover {
  transform: translateY(-4px);
}
```

## Layout Thrashing

The most common performance mistake involving reflow is layout thrashing: reading and writing layout-triggering properties in alternating sequence within the same JavaScript task.

When JavaScript writes a layout-affecting property, the browser marks the layout as "dirty" — it needs recalculation. Normally, the browser batches layout recalculations and runs them together before the next paint. But if JavaScript *reads* a layout property (like `offsetWidth`) after marking layout dirty, the browser is forced to recalculate layout *immediately* and synchronously — before the JavaScript execution continues — to return a valid current value.

```javascript
// Layout thrashing — forces synchronous reflow on every iteration
for (const el of elements) {
  el.style.width = el.parentNode.offsetWidth + 'px';  // read (forces reflow) + write
}

// Fixed — batch reads, then batch writes
const widths = elements.map(el => el.parentNode.offsetWidth);  // all reads
elements.forEach((el, i) => { el.style.width = widths[i] + 'px'; });  // all writes
```

Properties that force a synchronous layout when read:
```
offsetTop, offsetLeft, offsetWidth, offsetHeight, offsetParent
scrollTop, scrollLeft, scrollWidth, scrollHeight
clientTop, clientLeft, clientWidth, clientHeight
getComputedStyle(), getBoundingClientRect()
```

The `fastdom` library and React's batched state updates are solutions to this pattern — they queue reads and writes to avoid interleaving.

## The `will-change` Property

`will-change` is a hint to the browser that an element is about to change in a specific way. The browser can use this to set up GPU layers before the change occurs, avoiding the cost of layer promotion mid-animation.

```css
/* Good — tell the browser upfront, applied before animation starts */
.interactive-card {
  will-change: transform;
}

/* Bad — applying will-change to everything */
* {
  will-change: transform;  /* creates a layer for every element — destroys GPU memory */
}
```

`will-change` has a cost: each layer consumes GPU memory. Applying it broadly — to every element on a page — can exhaust GPU memory on mobile devices, causing worse performance than if you hadn't used it at all.

Best practice: apply `will-change` only to elements that will animate, and remove it after the animation completes if the animation is not continuous.

## CSS Properties by Cost Category

**Composite only (run on GPU, no main thread):**
- `transform` (translate, scale, rotate, skew)
- `opacity`

**Repaint only (no layout recalculation):**
- `color`, `background-color`, `border-color`
- `box-shadow`, `text-shadow`
- `border-radius` (when no size changes)
- `visibility`
- `outline`

**Reflow + repaint (most expensive):**
- `width`, `height`, `padding`, `margin`
- `top`, `left`, `right`, `bottom` (for positioned elements)
- `font-size`, `font-weight`, `line-height`
- `display`, `overflow`
- `float`, `position`

A useful reference: [csstriggers.com](https://csstriggers.com) catalogues every CSS property by which pipeline stages it triggers.

## Gotchas

**`transform: translate` and `position: left` look similar but perform completely differently.** Both move an element visually. `transform` is composite-only and GPU-driven. `left`/`top` (for positioned elements) trigger layout recalculation. For any animation, prefer `transform`.

**`display: none` → `display: block` is one of the most expensive operations.** It requires adding the element back to the render tree (reflow) and painting it from scratch. For elements that toggle visibility frequently, prefer `visibility: hidden` (stays in render tree) or `opacity: 0; pointer-events: none` (stays in render tree, painted on layer) over display toggling.

**Scrolling reads force reflow.** `element.scrollTop = 0` is a write (cheap — batched). `element.scrollTop` as a read after a write forces synchronous reflow. In scroll event handlers that both read and write positions, this is a common source of jank.

**`will-change` creates layers prematurely.** Using `will-change` on an element that isn't actually animating wastes GPU memory with no benefit. Apply it conditionally — add it on `mouseenter`, remove it on `mouseleave`:

```javascript
el.addEventListener('mouseenter', () => { el.style.willChange = 'transform'; });
el.addEventListener('mouseleave', () => { el.style.willChange = 'auto'; });
```

**Pseudo-element changes trigger the same pipeline as their host elements.** Changing `:hover` pseudo-element styles causes reflow/repaint on the element itself. A `:hover` that changes `width` triggers the same full reflow as changing `width` directly.

## Interview Questions

**Q (High): What is the difference between reflow, repaint, and composite-only changes? Give an example of a CSS property in each category.**

Answer: Reflow (layout) happens when element geometry changes — sizes, positions, anything that could affect surrounding elements. The browser must recalculate layout for the affected subtree. Examples: `width`, `height`, `padding`, `margin`, `font-size`, `top`, `left`. Repaint happens when appearance changes without geometry changing — the element redraws itself but no positions shift. Examples: `color`, `background-color`, `box-shadow`, `visibility`. Composite-only happens when `transform` or `opacity` on a separately composited layer changes — the GPU repositions or blends a pre-painted texture without any main-thread work, no layout or paint. Example: `transform: translateX(100px)`, `opacity: 0.5`. Cost order: reflow > repaint > composite-only. For any animation, prefer properties in the composite-only category — they run at 60fps even when the main thread is busy.

The trap: Calling `opacity` a repaint trigger unconditionally. Opacity on a composited layer is composite-only. Opacity on a non-composited element may trigger repaint. The distinction is whether the element has its own layer.

---

**Q (High): What is layout thrashing and how do you prevent it?**

Answer: Layout thrashing is the performance anti-pattern of interleaving layout writes (changes that mark layout dirty) with layout reads (queries that require the browser to complete layout synchronously). Normally the browser batches layout work and runs it before the next paint. But reading a layout property (offsetWidth, getBoundingClientRect, getComputedStyle, scrollTop) after writing a layout-affecting property forces the browser to recalculate layout immediately to return a valid value — this is a forced synchronous reflow. In a loop that reads and writes on every iteration, this causes a reflow per iteration instead of one batched reflow. Prevention: batch all reads together, then all writes together within the same JavaScript task. Frameworks like React batch state updates to prevent this automatically for component renders. For imperative DOM manipulation, read what you need first, then apply all writes.

The trap: Not knowing which property reads force layout. A common mistake is "I'm just reading, not writing, so it's fine" — the read forces the write to finalize, which is what's expensive.

---

**Q (High): Why should you prefer `transform` over `top`/`left` for animations?**

Answer: Both `transform: translateX(100px)` and `left: 100px` produce a visually moving element. But they take completely different paths through the rendering pipeline. `left` is a layout property — changing it triggers a full reflow, paint, and composite cycle on every animation frame. At 60fps, that's 60 reflows per second, each potentially affecting nearby elements and stressing the CPU. `transform` on a composited layer is handled entirely by the GPU compositor: the painted layer texture is just repositioned by updating a matrix. No layout recalculation, no repaint, no main thread involvement. The GPU can run this at 60fps independently, even when the main thread is processing JavaScript. Additionally, the browser can use hardware acceleration (GPU rendering) for `transform` animations, which is simply not possible for layout-based animations.

The trap: "Both work and look the same to the user." True for visual output, false for performance. On a mid-range mobile device, the difference between a layout-animated element and a transform-animated element is the difference between jank and smooth 60fps.

---

**Q (Medium): What does `will-change` do, when should you use it, and what's the risk of overusing it?**

Answer: `will-change` is a browser hint that an element will undergo a specific type of change — typically `transform` or `opacity`. The browser can use this to promote the element to its own compositor layer before any change occurs, avoiding the cost of layer creation mid-animation. This eliminates the "flash" or "paint" that can happen on the first frame of an animation if the layer promotion wasn't ready. When to use: for elements that animate `transform` or `opacity` in response to user interaction, and specifically when you observe a first-frame hiccup without it. Risk of overuse: each compositor layer consumes GPU memory — the layer stores a texture (rasterized pixels). Applying `will-change: transform` to every element (or inside `*`) creates hundreds of layers, exhausting GPU memory on mobile devices, causing the compositor to fall back to software rendering, and making performance worse than without it. Use it selectively and remove it after animations complete.

The trap: Using `will-change` as a general performance booster applied broadly. It's a scalpel, not a paint roller.

---

**Q (Low): How does CSS `contain: layout` affect reflow scope, and when would you use it?**

Answer: `contain: layout` is a CSS property that tells the browser the element's layout is independent of its surroundings — changes inside the element don't affect the layout of elements outside it, and vice versa. Without `contain: layout`, a change deep in a component's DOM subtree can cascade outward, requiring the browser to recalculate layout for the entire page or a large subtree. With `contain: layout`, the browser knows it can limit the layout recalculation to inside the containment boundary. This is valuable for widgets that have frequent internal layout changes — a virtualized list, a live-updating dashboard component — where you don't want internal changes to thrash the layout of the surrounding page. It also allows the browser to skip painting off-screen contained elements. The newer `content-visibility: auto` combines containment with lazy rendering for off-screen content.

The trap: Not knowing `contain` exists, or confusing it with `overflow: hidden` (which doesn't establish layout containment in the same way).

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can list one CSS property example from each category: reflow, repaint, composite-only
- [ ] Can explain why `transform` outperforms `left`/`top` for animations
- [ ] Can describe layout thrashing: what causes it and how to fix it with read-write batching
- [ ] Can explain what `will-change` does and why overusing it hurts performance
- [ ] Can name four DOM read properties that force synchronous layout (`offsetWidth`, `getBoundingClientRect`, etc.)
- [ ] Can explain why `display: none → block` is more expensive than `visibility: hidden → visible`

---
*Next: GPU Layers & Compositing — how the browser creates and manages GPU-backed layers, the mechanics of layer promotion, and the memory/performance trade-offs.*
