# ResizeObserver

## Quick Reference

| Concept | Detail |
|---|---|
| Purpose | Observe changes to an element's size (content box or border box) |
| Replaces | `window.resize` event for element-level layout reactions |
| Callback timing | After paint, asynchronous |
| Entry properties | `contentRect`, `contentBoxSize`, `borderBoxSize`, `devicePixelContentBoxSize` |
| Loop detection | Browser auto-skips notifications that would cause infinite resize loops |
| Cleanup | `observer.unobserve(el)` or `observer.disconnect()` |

---

## What Is This?

`ResizeObserver` notifies you when the size of an element changes — its content area, border box, or both. Unlike `window.resize`, which fires when the *window* changes size, `ResizeObserver` watches *individual elements* regardless of what caused the size change: window resize, CSS animation, DOM mutation, flex/grid reflow, font loading, or any other cause.

This is the correct API for building components whose internal layout depends on their own size — components that don't know in advance how wide they'll be placed.

---

## Why It Exists

`window.resize` is the wrong abstraction for component-level responsive behavior:

1. It only fires when the *window* changes, not when an element's size changes due to a layout shift elsewhere in the document
2. It doesn't tell you *which* elements changed or by how much
3. It requires developers to query `getBoundingClientRect()` or `offsetWidth` to get element sizes — which forces synchronous layout

ResizeObserver gives you element-level observation with no forced layout, delivered asynchronously after paint.

> **Check yourself:** If a component's width changes because its parent flexbox redistributes space — but the window size didn't change — which API catches it?

---

## Basic Usage

```js
const observer = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    entry.target.style.setProperty('--component-width', `${width}px`);

    if (width < 600) {
      entry.target.classList.add('compact');
    } else {
      entry.target.classList.remove('compact');
    }
  }
});

observer.observe(document.querySelector('.my-component'));
```

---

## Entry Properties

Each `ResizeObserverEntry` has several size representations:

```js
const observer = new ResizeObserver(([entry]) => {
  // Legacy, widely supported:
  const { width, height } = entry.contentRect;  // content box, excludes padding/border

  // Modern, preferred (array — for multi-column fragmentation):
  const [{ inlineSize, blockSize }] = entry.contentBoxSize;  // logical dimensions
  const [{ inlineSize: bInline, blockSize: bBlock }] = entry.borderBoxSize;  // includes padding+border
  const [{ inlineSize: dInline, blockSize: dBlock }] = entry.devicePixelContentBoxSize; // in physical pixels
});
```

**`contentRect`** — the content area (excluding padding and border), as a `DOMRectReadOnly`. Works in all browsers that support ResizeObserver.

**`contentBoxSize` / `borderBoxSize`** — logical size arrays using CSS logical properties (`inlineSize` = width in horizontal writing mode, `blockSize` = height). These are arrays to support multi-column fragmented elements.

**`devicePixelContentBoxSize`** — size in physical device pixels (not CSS pixels). Critical for canvas rendering to avoid blurriness on high-DPI screens.

---

## Box Model: Which Size to Observe

The `observe()` method accepts an options object specifying which box to observe:

```js
observer.observe(element, { box: 'content-box' });  // default — content area only
observer.observe(element, { box: 'border-box' });    // includes padding + border
observer.observe(element, { box: 'device-pixel-content-box' }); // physical pixels
```

Choosing `border-box` is important when the element uses `box-sizing: border-box` (the modern default) — the border-box size is what the element's CSS `width` describes.

---

## Primary Use Case: Container Queries (Before CSS Container Queries)

Before native CSS Container Queries, ResizeObserver was the only way to do element-based responsive layout. The pattern is equivalent to what `@container` now does in CSS:

```js
const breakpoints = { sm: 480, md: 768, lg: 1024 };

const observer = new ResizeObserver(([entry]) => {
  const width = entry.contentRect.width;
  const el = entry.target;

  el.dataset.size = width < breakpoints.sm ? 'sm'
    : width < breakpoints.md ? 'md'
    : width < breakpoints.lg ? 'lg'
    : 'xl';
});

observer.observe(document.querySelector('.card'));
```

```css
/* CSS reacts to data-size attribute */
.card[data-size="sm"] { flex-direction: column; }
.card[data-size="md"] { grid-template-columns: 1fr 1fr; }
```

Now that CSS Container Queries are widely supported, this JS pattern is no longer needed for pure layout changes — but ResizeObserver remains the right tool when *JavaScript logic* needs to react to element size changes.

---

## Canvas Rendering on High-DPI Screens

The most precise ResizeObserver use case:

```js
const canvas = document.querySelector('canvas');
const ctx = canvas.getContext('2d');

const observer = new ResizeObserver(([entry]) => {
  // Use device pixel size for sharp rendering on high-DPI screens
  const [{ inlineSize: width, blockSize: height }] = entry.devicePixelContentBoxSize;
  canvas.width = width;
  canvas.height = height;
  // Re-render at the new resolution
  render(ctx);
});

observer.observe(canvas, { box: 'device-pixel-content-box' });
```

This is the correct pattern. Using `canvas.offsetWidth * devicePixelRatio` is the common alternative but less accurate due to float rounding. `devicePixelContentBoxSize` gives exact integer pixel dimensions.

---

## React Pattern

```jsx
import { useEffect, useRef, useState } from 'react';

function useElementSize(ref) {
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    if (!ref.current) return;
    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      setSize({ width, height });
    });
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref]);

  return size;
}

function ResponsiveCard() {
  const ref = useRef(null);
  const { width } = useElementSize(ref);

  return (
    <div ref={ref} className={width < 400 ? 'card--compact' : 'card--full'}>
      {/* ... */}
    </div>
  );
}
```

---

## Infinite Loop Protection

If a ResizeObserver callback causes a resize (e.g., changing an element's width inside the callback changes another observed element's width), the browser detects the loop and skips delivering further notifications in that frame. It fires a `ResizeObserver loop completed with undelivered notifications` console error but doesn't throw — it just silently prevents the infinite loop.

This means ResizeObserver is safe in theory but the error signal is subtle. When you see this console warning, the fix is to break the feedback loop — usually by using `requestAnimationFrame` to defer the DOM change:

```js
const observer = new ResizeObserver(([entry]) => {
  requestAnimationFrame(() => {
    // Make DOM changes in the next frame — breaks the synchronous feedback loop
    entry.target.style.width = computeNewWidth(entry.contentRect.width) + 'px';
  });
});
```

---

## Gotchas

**Callback fires on initial observe.** Like IntersectionObserver, ResizeObserver fires the callback once immediately after `observe()` with the current size.

**Multiple elements in one callback.** If you observe multiple elements and they all resize in the same frame, the callback receives all entries in one call. Always iterate `entries`.

**No `once` equivalent.** If you want a one-time size reading, unobserve inside the callback.

**`contentRect` vs. `getBoundingClientRect`** — `contentRect` excludes transforms (CSS `scale`, `rotate`). `getBoundingClientRect()` includes transforms. For layout purposes, `contentRect` is correct. For visual position, `getBoundingClientRect()` is correct.

---

## Interview Questions

**Q (High): Why use ResizeObserver instead of `window.addEventListener('resize', ...)`?**

Answer: `window.resize` only fires when the browser window is resized. An element's size can change for many other reasons: its parent's flex or grid layout redistributes space, a sibling is removed from the DOM, a CSS class is toggled, a font loads and changes text wrapping, or an accordion expands. None of these trigger `window.resize`. ResizeObserver watches a specific element and fires whenever that element's content box or border box dimensions change, regardless of cause. Additionally, `window.resize` doesn't tell you which elements changed or by how much — you still have to call `getBoundingClientRect()` or read `offsetWidth`, which forces synchronous layout. ResizeObserver delivers the new size in the callback entry with no layout forcing.

**Q (High): What is `devicePixelContentBoxSize` and why does canvas rendering need it?**

Answer: `devicePixelContentBoxSize` gives the element's content box size in physical device pixels rather than CSS logical pixels. On a device with `devicePixelRatio = 2` (like most modern phones and Retina displays), a CSS pixel equals two physical pixels. A canvas that is 300 CSS pixels wide needs to have its `canvas.width` set to 600 physical pixels to render at full sharpness. Without this, canvas content is upscaled and appears blurry. The traditional approach is `canvas.width = canvas.offsetWidth * devicePixelRatio`, but this involves float multiplication and rounding. `devicePixelContentBoxSize` gives the exact integer physical pixel dimensions directly, without rounding errors. Observe with `{ box: 'device-pixel-content-box' }` to receive this data.

**Q (Medium): What triggers the "ResizeObserver loop completed with undelivered notifications" error?**

Answer: This happens when the ResizeObserver callback causes a size change in an element that is itself being observed (directly or indirectly). The callback fires → DOM change → observed element resizes → callback would fire again → loop. The browser detects this as an infinite loop and breaks it by not delivering the notifications that would continue the loop, issuing a console warning instead. The callback is not thrown — it's a silent skip. The fix is to break the feedback loop by deferring the DOM change out of the current callback execution. Wrapping the mutation in `requestAnimationFrame` schedules it in the next frame, where it's treated as a new observation cycle rather than part of the current loop.

**Q (Low): What is the difference between `contentRect` and `borderBoxSize` in a ResizeObserver entry?**

Answer: `contentRect` is the content area size — it excludes padding and border. If an element has `padding: 10px` and `border: 2px`, and is 200px wide total (border box), `contentRect.width` is 176px (200 - 20px padding - 4px border). `borderBoxSize` gives the full box including padding and border — its `inlineSize` is 200px in this case. For most layout logic you want `borderBoxSize`, because that's what CSS `width` describes when `box-sizing: border-box` is set (which is the standard today). For positioning content inside the element's content area, use `contentRect`. The difference only matters for elements with non-zero padding or border.

---

## Self-Assessment

- [ ] Explain why `window.resize` is insufficient for component-level responsive layout
- [ ] Implement a useElementSize hook using ResizeObserver in React
- [ ] Explain when to use `devicePixelContentBoxSize` and why
- [ ] Describe what causes the resize loop warning and how to fix it
- [ ] Choose between `content-box` and `border-box` observation for a given use case

---
*Next: MutationObserver — detecting DOM changes, use cases, and performance gotchas.*
