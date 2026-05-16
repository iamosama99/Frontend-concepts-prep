# Animation Performance: FLIP, CSS vs. JS, requestAnimationFrame

## Quick Reference

| Property | Triggers layout? | Triggers paint? | Compositor only? |
|---|---|---|---|
| `transform` | No | No | Yes — cheap |
| `opacity` | No | No | Yes — cheap |
| `width`, `height` | Yes | Yes | No — expensive |
| `top`, `left`, `margin` | Yes | Yes | No — expensive |
| `background-color` | No | Yes | No — medium |
| `border-radius` | No | Yes | No — medium |
| `box-shadow` | No | Yes | No — medium |
| `clip-path` | No | Yes | No — medium |

---

## What Is This?

Animation performance is about ensuring visual transitions run at 60 fps (16ms per frame) or above without causing jank — visible stuttering where frames are dropped. The browser's rendering pipeline has discrete stages: JavaScript → Style → Layout → Paint → Composite. Animating properties that force the browser back through Layout or Paint on every frame is the primary cause of animation jank.

Three key topics: which CSS properties are "compositor-safe" (only touch the Composite stage), how to use the FLIP animation technique to cheaply animate layout changes, and how `requestAnimationFrame` integrates JavaScript-driven animations with the browser's rendering loop.

> **Check yourself:** If you animate `top: 100px` → `top: 0px` on an element, what browser rendering stages run on every frame? How does that compare to animating `transform: translateY(100px)` → `transform: translateY(0px)`?

---

## Why Does It Exist?

The browser targets 60 frames per second. Each frame has a 16ms budget. The rendering pipeline for each frame:

```
JS execution → Style recalculation → Layout (reflow) → Paint → Composite
```

If any stage takes too long, frames are dropped and animation stutters. Layout is the most expensive stage because it must compute the size and position of every affected element and its descendants. Paint is the second most expensive — rasterizing pixel colors to memory. Compositing is cheap: the browser just moves pre-painted layers on the GPU.

Animating properties that affect geometry (width, height, top, left, margin, padding) triggers a full layout + paint on every frame. At 60 fps that is 60 full recalculations per second. Animating only `transform` and `opacity` skips layout and paint entirely — the browser composites pre-painted layers on the GPU, which is orders of magnitude faster.

---

## Compositor-Safe Properties

### `transform` and `opacity`

These two properties are handled entirely by the browser's compositor thread — not the main thread. The compositor thread runs independently of JavaScript, meaning even a long-running JS task won't stutter a CSS `transform`/`opacity` animation.

```css
/* Jank-causing animation: triggers layout + paint on every frame */
@keyframes slide-bad {
  from { left: -100px; }
  to   { left: 0; }
}

/* Smooth animation: compositor only, never touches layout or paint */
@keyframes slide-good {
  from { transform: translateX(-100px); }
  to   { transform: translateX(0); }
}

.panel {
  animation: slide-good 0.3s ease-out;
}
```

```css
/* Fade in — compositor only */
@keyframes fade-in {
  from { opacity: 0; }
  to   { opacity: 1; }
}
```

**`will-change` hint:**

`will-change: transform` tells the browser in advance that this element will be animated. The browser promotes it to its own GPU layer before the animation starts, eliminating the layer-creation cost at animation start.

```css
.modal {
  will-change: transform, opacity;
}
```

Use `will-change` sparingly — each layer consumes GPU memory. Too many promoted layers can exhaust VRAM on mobile devices. Apply it only to elements that will actually animate, and remove it after the animation ends.

```js
// Set will-change before animation, remove after
element.style.willChange = 'transform';
element.addEventListener('animationend', () => {
  element.style.willChange = 'auto';
}, { once: true });
```

---

## The FLIP Technique

### The Problem FLIP Solves

Animating layout changes — an element moving from one position in the page to another, changing size, or reordering in a list — is expensive because the final position is determined by document flow. You can't animate `position: static` elements with `transform` until you know where they started and where they end up.

FLIP is a technique that makes layout-driven animations cheap by recording positions before and after, then animating only `transform` and `opacity`.

**FLIP = First, Last, Invert, Play**

### Step by Step

```
First  — Record the element's initial position/size (getBoundingClientRect)
Last   — Apply the layout change (DOM update, CSS class change)
Invert — Apply a transform that visually reverses the change (back to First position)
Play   — Animate the transform to identity (0, 0) — smooth compositor animation
```

```js
function flipAnimate(element, applyChange) {
  // First — record starting position
  const first = element.getBoundingClientRect();

  // Last — apply the layout change
  applyChange();

  // Force layout recalculation to get final position
  const last = element.getBoundingClientRect();

  // Invert — calculate the delta and apply it as a transform
  const deltaX = first.left - last.left;
  const deltaY = first.top - last.top;
  const deltaW = first.width / last.width;
  const deltaH = first.height / last.height;

  // Instantly snap element back to First position using transform
  element.style.transform = `
    translate(${deltaX}px, ${deltaY}px)
    scale(${deltaW}, ${deltaH})
  `;
  element.style.transformOrigin = 'top left';

  // Play — animate transform to identity (the actual Last position)
  requestAnimationFrame(() => {
    element.style.transition = 'transform 0.3s ease-out';
    element.style.transform = '';
  });
}
```

**Practical example — list reorder:**

```js
const list = document.querySelector('.items');
const items = [...list.querySelectorAll('.item')];

// First — record all positions
const firsts = new Map(items.map(el => [el, el.getBoundingClientRect()]));

// Last — reorder in DOM
const sorted = [...items].sort((a, b) => a.dataset.name.localeCompare(b.dataset.name));
sorted.forEach(el => list.appendChild(el));

// Invert + Play for each item
items.forEach(el => {
  const first = firsts.get(el);
  const last = el.getBoundingClientRect();

  const deltaX = first.left - last.left;
  const deltaY = first.top - last.top;

  if (deltaX === 0 && deltaY === 0) return; // didn't move

  el.style.transform = `translate(${deltaX}px, ${deltaY}px)`;

  requestAnimationFrame(() => {
    el.style.transition = 'transform 0.4s cubic-bezier(0.25, 0.46, 0.45, 0.94)';
    el.style.transform = '';

    el.addEventListener('transitionend', () => {
      el.style.transition = '';
    }, { once: true });
  });
});
```

**FLIP in React with Framer Motion:**

Framer Motion's `layout` prop implements FLIP automatically. Add `layout` to any element and Framer Motion handles the FLIP calculation internally when the element's layout changes.

```jsx
import { motion } from 'framer-motion';

function SortableList({ items }) {
  return (
    <ul>
      {items.map(item => (
        // layout prop enables automatic FLIP animation
        <motion.li key={item.id} layout transition={{ duration: 0.3 }}>
          {item.name}
        </motion.li>
      ))}
    </ul>
  );
}
```

---

## requestAnimationFrame

### What It Does

`requestAnimationFrame(callback)` schedules the callback to run before the browser's next repaint — synchronized with the screen's refresh rate (typically 60 Hz, 90 Hz, or 120 Hz). It is the correct API for JavaScript-driven animations because:

1. It synchronizes with the display refresh cycle — no tearing or partial frames.
2. The browser pauses it when the tab is hidden — saving CPU/battery.
3. It gives you the current timestamp with sub-millisecond precision.

```js
function animate(timestamp) {
  // timestamp is DOMHighResTimeStamp — microsecond precision
  const elapsed = timestamp - startTime;
  const progress = Math.min(elapsed / duration, 1); // 0 to 1

  // Easing function
  const eased = easeOutCubic(progress);

  element.style.transform = `translateX(${eased * targetX}px)`;

  if (progress < 1) {
    requestAnimationFrame(animate); // schedule next frame
  }
}

const startTime = performance.now();
const duration = 400;
const targetX = 300;
requestAnimationFrame(animate);
```

**Easing functions:**

```js
// Common easing functions — progress is 0 to 1, returns 0 to 1
const easings = {
  linear:      t => t,
  easeOutCubic: t => 1 - Math.pow(1 - t, 3),
  easeInOutCubic: t => t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2,
  easeOutBounce: t => {
    const n1 = 7.5625, d1 = 2.75;
    if (t < 1 / d1) return n1 * t * t;
    if (t < 2 / d1) return n1 * (t -= 1.5 / d1) * t + 0.75;
    if (t < 2.5 / d1) return n1 * (t -= 2.25 / d1) * t + 0.9375;
    return n1 * (t -= 2.625 / d1) * t + 0.984375;
  },
};
```

---

### rAF vs. `setInterval` / `setTimeout`

**Never use `setInterval` or `setTimeout` for animations:**

```js
// Bad — runs at a fixed interval regardless of display refresh rate
// Will cause tearing at 90/120Hz, drops frames when the browser is busy
setInterval(() => {
  element.style.left = parseInt(element.style.left) + 2 + 'px';
}, 16);

// Good — synchronized with the display, paused when tab is hidden
function step() {
  // ... animation logic
  requestAnimationFrame(step);
}
requestAnimationFrame(step);
```

`setInterval(fn, 16)` fires at ~62 Hz on a 60 Hz display, slightly out of sync. On a 90 Hz or 120 Hz display, frames are skipped. When the tab is hidden, `setInterval` continues running, wasting CPU. `requestAnimationFrame` automatically adapts to the screen's refresh rate and pauses when not visible.

---

### CSS Animations vs. JavaScript Animations

| Dimension | CSS animation | JS + rAF |
|---|---|---|
| Threading | Compositor thread (transform/opacity only) | Main thread |
| JS blocked? | No — runs independently | Yes — paused by long JS tasks |
| Dynamic values | Limited (needs static keyframes) | Full JS expressiveness |
| Interruptible | With JavaScript override | Natively |
| Physics/spring | Not built in | Full control |
| Dev ergonomics | Simple for fixed transitions | More code |

**When to use CSS animation:**

- Fixed transitions (open/close, enter/exit) with known start and end states
- Compositor-safe properties (transform, opacity)
- When you want zero JS cost and the animation must run even under JS load

**When to use JS + rAF:**

- Physics-based animations (spring, inertia, particle systems)
- Animations driven by real-time user input (drag, scroll progress)
- Animations that must be interruptible mid-flight with dynamic targets
- Sequenced animations across multiple elements with complex timing

**The Web Animations API (WAAPI)** is a middle ground — JavaScript API that runs animations on the compositor thread like CSS:

```js
// Web Animations API — runs on compositor thread, controllable from JS
const animation = element.animate(
  [
    { transform: 'translateX(-100px)', opacity: 0 },
    { transform: 'translateX(0)',      opacity: 1 },
  ],
  {
    duration: 400,
    easing: 'cubic-bezier(0.25, 0.46, 0.45, 0.94)',
    fill: 'forwards',
  }
);

// Interruptible — cancel and reverse mid-flight
animation.reverse();
animation.cancel();

// Await completion
await animation.finished;
```

WAAPI gives you compositor-thread performance (for transform/opacity) with full JavaScript control — the best of both worlds for most interactive animations.

---

## Layout Thrashing

Layout thrashing is a pattern where JavaScript alternately reads and writes layout properties in a loop, forcing the browser to recompute layout on every read. Each `getBoundingClientRect()`, `offsetWidth`, `scrollTop` after a DOM write invalidates the layout cache and forces a synchronous reflow.

```js
// Layout thrashing — forces reflow on every element
const elements = document.querySelectorAll('.item');
elements.forEach(el => {
  const width = el.offsetWidth;          // read — forces reflow
  el.style.width = width * 1.5 + 'px';  // write — invalidates layout
  // Next iteration reads offsetWidth after a write → forced reflow again
});

// Fixed — batch reads, then batch writes (FastDOM pattern)
const widths = [...elements].map(el => el.offsetWidth);  // all reads
elements.forEach((el, i) => {
  el.style.width = widths[i] * 1.5 + 'px';               // all writes
});
```

The FastDOM library formalizes this: reads are batched in a microtask, writes in the next rAF frame, ensuring the browser only reflows once per frame.

---

## Gotchas

**`will-change` GPU memory cost.** Each element with `will-change: transform` is promoted to a GPU layer. A page with 50 animated list items all using `will-change` can exhaust GPU memory on mobile, causing the browser to fallback to software rendering — slower than no will-change at all. Apply `will-change` right before animation starts, remove it when done.

**FLIP with scale requires adjusting children.** Scaling a parent element with `scale()` in the FLIP invert step also scales its children, distorting text. To correct, apply the inverse scale to children: if parent scales by `deltaW` and `deltaH`, children need `scale(1/deltaW, 1/deltaH)`. Framer Motion handles this automatically.

**`requestAnimationFrame` callbacks run before the next paint, not after.** The timestamp passed to the callback is the *scheduled* start of the next frame, not when the callback actually runs. If your callback takes 10ms of computation, the animation update happens 10ms later — your easing calculation should still use the `timestamp` parameter, not `performance.now()`.

**CSS `transition` on dynamically added elements.** If you add a class that triggers a transition on an element at the same time the element enters the DOM, the browser sees the element at its final state — no transition. Force a reflow between adding the element and adding the transition class:

```js
element.classList.add('fade-element');
element.offsetHeight; // force reflow — reads layout, flushing pending writes
element.classList.add('fade-in');
```

---

## Interview Questions

**Q (High): Why does animating `transform` avoid layout and paint, and what makes it "compositor-only"?**
Answer: The browser's rendering pipeline has stages: JavaScript → Style → Layout → Paint → Composite. Layout computes element positions and sizes; Paint rasterizes pixels to memory bitmaps. The Composite stage assembles painted layers and positions them on screen. `transform` and `opacity` can be applied by the Compositor — they change how pre-painted bitmaps are moved or blended on the GPU, without needing to recompute layout or re-rasterize pixels. The browser promotes animated elements to their own GPU layers (via `will-change: transform` or when an animation starts), and the compositor thread moves these layers independently of the main thread. This means a heavy JavaScript task can block the main thread without stuttering a CSS transform animation, because the compositor runs on a separate thread.
The trap: Thinking that just using `transform` in CSS automatically avoids all performance issues — you also need to ensure the element is actually promoted to its own layer.

**Q (High): Explain the FLIP animation technique step by step.**
Answer: FLIP makes layout-driven animations cheap by converting them into compositor-safe transform animations. First: Record the element's initial bounding box with `getBoundingClientRect()`. Last: Apply the layout change (add class, modify DOM, reorder elements) and record the final bounding box. Invert: Calculate the delta between First and Last positions, and immediately apply a `transform` that moves the element back to its First position — visually it looks unchanged. Play: Animate the `transform` to identity (`translate(0, 0)`) using a CSS transition or WAAPI. The element appears to animate from First to Last, but the browser only runs compositor-stage work for the animation — the expensive layout recalculation happened once, synchronously, during the "Last" step.
The trap: Missing that the layout change still happens once synchronously — FLIP doesn't avoid layout; it avoids running layout on every animation frame.

**Q (High): What is the difference between `requestAnimationFrame` and `setInterval` for animations, and why does the difference matter?**
Answer: `setInterval` fires at a fixed wall-clock interval regardless of the display refresh rate or browser state. On a 60 Hz screen, `setInterval(fn, 16)` fires at ~62 Hz — slightly misaligned, causing micro-stuttering. On a 120 Hz display, frames are skipped. When the tab is hidden, `setInterval` continues running, wasting CPU and battery. `requestAnimationFrame` fires synchronized with the display's next repaint — once per frame at the screen's actual refresh rate (60, 90, or 120 Hz). It pauses automatically when the tab is hidden, saving resources. The callback receives a high-precision timestamp, enabling correct delta-time-based animation that adapts to frame rate. The practical difference: rAF animations are visually smooth and resource-efficient; `setInterval` animations stutter at non-60Hz frame rates and waste CPU when hidden.
The trap: Thinking `setInterval(fn, 16)` is equivalent to rAF because 16ms ≈ 60Hz. The synchronization with the display repaint cycle is the critical difference.

**Q (Medium): What is layout thrashing and how do you fix it?**
Answer: Layout thrashing occurs when JavaScript interleaves DOM writes and reads in a loop, forcing the browser to synchronously recompute layout on each read. After any DOM write that affects layout, the browser marks its layout cache as invalid. The next read of a layout property (`offsetWidth`, `getBoundingClientRect`, `scrollTop`) forces a synchronous layout recalculation — this is called a "forced reflow." In a loop that reads after every write, the browser reflows on every iteration, making an O(n) layout calculation O(n²). Fix: batch all reads first (before any writes), then batch all writes. This pattern is called "read-write separation." The FastDOM library enforces this by scheduling reads in microtasks and writes in rAF frames, ensuring only one layout calculation per frame.
The trap: Not knowing what triggers a forced reflow, or thinking just "don't use JavaScript animations" is the answer.

**Q (Medium): When would you use the Web Animations API (WAAPI) instead of CSS transitions or JS + rAF?**
Answer: WAAPI is ideal when you need compositor-thread performance (for transform/opacity animations that run independently of the main thread) but also need JavaScript control — the ability to pause, reverse, seek, cancel, or await completion of an animation. CSS transitions and animations run on the compositor thread but are not programmatically controllable mid-flight (you can't call `.reverse()` or `await animation.finished`). Pure JS + rAF gives full control but runs on the main thread and can be interrupted by long JS tasks. WAAPI combines both: it runs transform/opacity on the compositor thread and exposes a Promise-based API for control. Use WAAPI for interactive animations that respond to user gestures, multi-step choreographed sequences that need await-based orchestration, or when you need to start a CSS-level animation from JavaScript with programmatic control.
The trap: Not knowing WAAPI runs on the compositor thread for transform/opacity — it's not the same as rAF.

**Q (Low): Why do CSS transitions sometimes not fire on elements that are added to the DOM?**
Answer: CSS transitions animate between two states. When an element is added to the DOM with a class that includes the final transition state, the browser sees only one state — the "from" state doesn't exist. The transition has nothing to animate from. The browser skips the transition and renders the final state immediately. Fix: force a reflow between adding the element and adding the transition class, which causes the browser to record the initial state before applying the transition. `element.offsetHeight` is the classic flush trick — reading a layout property forces a synchronous layout, which locks in the current visual state as the transition's starting point. After that, adding the class that changes the animated property triggers the transition normally.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Name the two compositor-safe CSS properties and explain what "compositor-only" means in the browser pipeline
- [ ] Describe the four steps of FLIP and write a working implementation for a list reorder
- [ ] Explain the difference between `requestAnimationFrame` and `setInterval` for animations, including what happens on 120Hz displays
- [ ] Write a rAF-based animation loop with delta-time-based easing
- [ ] Define layout thrashing and show the read-write separation fix
- [ ] Compare CSS animations, JS + rAF, and WAAPI — which runs on which thread and when you'd choose each

---
*Next: Phase 7 — Concurrency, Parallelism & Scheduling. Starts with Web Workers, which move CPU-intensive work off the main thread — the complement to keeping animations smooth by running compositor-only.*
