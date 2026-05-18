# IntersectionObserver

## Quick Reference

| Concept | Detail |
|---|---|
| Purpose | Detect when an element enters/exits a viewport or ancestor container |
| Replaces | `scroll` event + `getBoundingClientRect()` polling |
| Performance | Async, off main thread, batched — no forced synchronous layout |
| `threshold` | 0–1 or array: what fraction of the element must be visible to trigger |
| `rootMargin` | CSS-margin-like expansion of the root boundary (for pre-loading) |
| `root` | Ancestor element or `null` for the viewport |

---

## What Is This?

`IntersectionObserver` is an async API that notifies you when a target element intersects with a root element (default: the viewport). Rather than listening to `scroll` events and computing element positions on every scroll tick, you declare *what* you want to watch and *when* (at what threshold of visibility), and the browser calls your callback asynchronously when the condition changes.

The key insight: the browser can compute intersections as part of its compositing step — off the main thread — and batch-deliver the results to your callback. There's no forced synchronous layout, no polling, no jank.

---

## Why It Replaced Scroll + getBoundingClientRect

The old pattern:

```js
// Old approach — runs on every scroll event, forces layout recalc
window.addEventListener('scroll', () => {
  const rect = element.getBoundingClientRect(); // synchronous layout — expensive
  if (rect.top < window.innerHeight) {
    loadContent(); // lazy load trigger
  }
});
```

Problems:
1. `scroll` fires on every pixel of scroll — dozens of times per frame
2. `getBoundingClientRect()` forces a synchronous layout calculation
3. Together they cause jank on scroll, especially on low-end devices
4. Requires manual cleanup and threshold logic

IntersectionObserver solves all of these by design.

> **Check yourself:** Why does IntersectionObserver not cause layout thrashing even though it's measuring element positions?

---

## Basic Usage

```js
const observer = new IntersectionObserver((entries, observer) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      // Element entered the viewport
      loadImage(entry.target);
      observer.unobserve(entry.target); // stop observing after first intersection
    }
  });
});

// Start observing elements
document.querySelectorAll('img[data-src]').forEach(img => {
  observer.observe(img);
});
```

### The callback

The callback receives an array of `IntersectionObserverEntry` objects — one per observed element that changed intersection state. Multiple elements can trigger the same callback call if they intersect simultaneously.

Key entry properties:

```js
entry.isIntersecting   // boolean — true if any part is visible
entry.intersectionRatio  // 0 to 1 — fraction of the element that is visible
entry.target           // the observed element
entry.boundingClientRect  // element's bounding box
entry.intersectionRect  // the visible portion of the element
entry.rootBounds       // the root's bounding box
entry.time             // DOMHighResTimeStamp — when the intersection change occurred
```

---

## Configuration Options

```js
const observer = new IntersectionObserver(callback, {
  root: document.querySelector('.scroll-container'), // null = viewport
  rootMargin: '200px 0px',  // expand root by 200px on top/bottom — pre-load trigger
  threshold: [0, 0.25, 0.5, 0.75, 1], // fire at each 25% visibility change
});
```

### `threshold`

Controls when the callback fires based on visibility fraction:

```js
// Fire only when fully invisible (0) or fully visible (1)
{ threshold: [0, 1] }

// Fire at every 10% increment
{ threshold: Array.from({ length: 11 }, (_, i) => i / 10) }

// Default: fire once at any intersection (threshold 0)
{ threshold: 0 }
```

### `rootMargin`

Expands (or shrinks) the root rectangle before computing intersection. Same format as CSS `margin` (top right bottom left):

```js
{ rootMargin: '200px 0px 200px 0px' }
// Root boundary is expanded by 200px on top and bottom
// An element 200px below the viewport is considered "intersecting"
// Perfect for pre-loading images before they scroll into view
```

Negative margins shrink the root — useful for "is element fully in the viewport" detection:

```js
{ rootMargin: '-100px' }
// Element must be 100px inside the viewport on all sides to trigger
```

---

## Common Use Cases

### 1. Lazy loading images

```js
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach(({ isIntersecting, target }) => {
    if (!isIntersecting) return;
    target.src = target.dataset.src;
    target.removeAttribute('data-src');
    imageObserver.unobserve(target);
  });
}, { rootMargin: '200px' }); // pre-load 200px before viewport

document.querySelectorAll('img[data-src]').forEach(img => {
  imageObserver.observe(img);
});
```

### 2. Infinite scroll

```js
const sentinel = document.querySelector('.scroll-sentinel'); // empty div at list bottom

const infiniteScroll = new IntersectionObserver(([entry]) => {
  if (entry.isIntersecting) {
    fetchNextPage().then(renderItems);
  }
}, { threshold: 0.1 });

infiniteScroll.observe(sentinel);
```

### 3. Analytics — visibility tracking

```js
// Track when an ad or content block is actually seen
const adObserver = new IntersectionObserver((entries) => {
  entries.forEach(({ isIntersecting, intersectionRatio, target, time }) => {
    if (isIntersecting && intersectionRatio >= 0.5) {
      // 50% of the ad is visible — record impression
      analytics.track('ad_impression', {
        adId: target.dataset.adId,
        timestamp: time,
      });
      adObserver.unobserve(target); // only count once
    }
  });
}, { threshold: 0.5 });
```

### 4. Scroll-triggered animations

```js
const animationObserver = new IntersectionObserver((entries) => {
  entries.forEach(({ isIntersecting, target }) => {
    target.classList.toggle('is-visible', isIntersecting);
  });
}, { threshold: 0.1 });

document.querySelectorAll('.animate-on-scroll').forEach(el => {
  animationObserver.observe(el);
});
```

---

## Cleanup

```js
// Unobserve a specific element
observer.unobserve(element);

// Stop observing all elements and disconnect
observer.disconnect();
```

In React:

```js
useEffect(() => {
  const observer = new IntersectionObserver(callback, options);
  observer.observe(ref.current);
  return () => observer.disconnect(); // cleanup on unmount
}, []);
```

---

## Gotchas

**Callback fires immediately on `observe()`** — with the current intersection state. This is intentional: you always get an initial state report, even if the element isn't scrolled to yet. Handle the case where `isIntersecting` is `false` on the first call.

**The callback is asynchronous** — it fires after the browser finishes painting, not synchronously during scroll. Don't use it where you need synchronous position data.

**One observer per configuration** — if you have two sets of elements with different `rootMargin` or `threshold` values, you need two observer instances. Sharing one observer with different options isn't possible.

**`rootMargin` doesn't work with `root: null` in cross-origin iframes** — a security restriction. Margins work fine with explicit root elements.

**Threshold precision** — the browser may coalesce nearby thresholds. Setting `threshold: 0.499` and `0.5` may produce one callback per event, not two.

---

## Interview Questions

**Q (High): Why is IntersectionObserver more performant than scroll event + getBoundingClientRect?**

Answer: Two separate reasons compound. First, `scroll` events fire on every pixel of scroll movement — potentially 60+ times per second. Each handler call burns main thread time. `IntersectionObserver` callbacks fire only when intersection state *changes* — element crosses a threshold — which is far less frequent. Second, `getBoundingClientRect()` forces a synchronous layout: the browser must finish all pending style calculations and layout to return an accurate value. Called on scroll, this causes layout thrashing — a layout forced mid-frame, which invalidates the next frame's layout calculations, causing another forced layout. `IntersectionObserver` computes intersections as part of the browser's own compositing step — it can do this on a separate thread or at least without forcing a synchronous main-thread layout. The browser batches the results and delivers them asynchronously after painting.

**Q (High): What does the `rootMargin` option do, and what is its most common use case?**

Answer: `rootMargin` expands (or shrinks) the root element's bounding rectangle before the intersection calculation. It uses the same four-value syntax as CSS `margin`: top, right, bottom, left. A positive value expands the root boundary — an element that is 200px below the actual viewport boundary is considered "intersecting" if `rootMargin: '0px 0px 200px 0px'`. The most common use case is pre-loading: setting `rootMargin: '200px'` on an image lazy-loader means images start loading when they're 200 pixels away from the viewport, not when they first become visible. This eliminates the flash of empty images that appears when a user scrolls faster than images load. Negative margins work the opposite way — they shrink the intersection zone, useful for detecting when an element is fully visible with some margin of safety.

**Q (Medium): Why does the IntersectionObserver callback fire immediately when you call `observe()`, and how does that affect your handler?**

Answer: The browser fires the callback synchronously (well, as a microtask) after `observe()` to deliver the initial intersection state. This is useful: you don't need a separate "check if it's already in the viewport" path. But it means your handler must correctly handle `isIntersecting === false` on the first call — not just the subsequent scroll-triggered calls. A common bug is writing `if (entry.isIntersecting) doWork()` and not realizing the handler also fires when the element is scrolled out of view (threshold crossed in the other direction). If you're using IntersectionObserver for a one-time "element first enters viewport" trigger, always call `observer.unobserve(entry.target)` inside the `isIntersecting` branch to avoid the exit callback.

**Q (Low): Can you observe an element with two different threshold values using one IntersectionObserver instance?**

Answer: No. An `IntersectionObserver` instance has a single configuration — one `root`, one `rootMargin`, one `threshold` array — applied uniformly to all observed elements. To observe elements with different threshold configurations, create separate observer instances. In practice, this is common: a lazy-load observer with `{ rootMargin: '200px', threshold: 0 }` and an analytics impression observer with `{ threshold: 0.5 }` are two different instances, each observing their respective elements. The overhead is minimal — observers are cheap, and each callback batch is delivered efficiently regardless of how many observers exist.

---

## Self-Assessment

- [ ] Explain why IntersectionObserver doesn't cause layout thrashing
- [ ] Implement a lazy-loading image observer that pre-loads 300px before the viewport
- [ ] Explain what happens when you call `observe()` — when does the callback first fire?
- [ ] Describe what `threshold: [0, 0.5, 1]` means and when the callback fires for each
- [ ] Write the cleanup pattern for IntersectionObserver inside a React useEffect

---
*Next: ResizeObserver — element-level resize detection that replaces window resize listeners for component-aware layout.*
