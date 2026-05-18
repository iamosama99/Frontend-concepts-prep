# Observer APIs vs. Event Listener Patterns

## Quick Reference

| | Event Listeners | Observer APIs |
|---|---|---|
| Timing | Synchronous — fires during event dispatch | Asynchronous — fires as microtask or after paint |
| Batching | One call per event | Multiple changes batched into one callback |
| Source of truth | Developer registers on specific event | Browser determines when conditions change |
| Use case | User interactions, UI state changes | Browser-measured changes (size, visibility, DOM, performance) |
| Cleanup | `removeEventListener` | `observer.disconnect()` or `observer.unobserve()` |
| Forced layout risk | Yes (with `getBoundingClientRect`) | No — browser delivers measurements, no query needed |

---

## What Is This?

Both event listeners and observer APIs let you react to changes in the browser. They're not alternatives for the same job — they solve fundamentally different problems. Knowing which to choose requires understanding what each is optimized for.

**Event listeners** respond to *things that happen*: a user clicks, a key is pressed, a form is submitted, a network request completes. The browser dispatches a DOM event synchronously through the event target tree, and your handler runs in the middle of that dispatch.

**Observer APIs** (IntersectionObserver, ResizeObserver, MutationObserver, PerformanceObserver) react to *browser-measured state changes*: an element crosses a visibility threshold, an element's size changes, the DOM tree is mutated, or a performance metric is recorded. The browser determines when these conditions change and delivers batched notifications asynchronously.

---

## Why the Distinction Matters

Using the wrong tool for the job either doesn't work or creates performance problems:

- Using `scroll` events to detect element visibility → layout thrashing, janky scroll
- Using ResizeObserver to react to a button click → can't — clicks aren't observable by ResizeObserver
- Using MutationObserver for user input → works, but the event already exists
- Using `resize` events to observe a component's size → misses non-window-caused size changes

> **Check yourself:** A notification count badge needs to update when a new message arrives via WebSocket. Which pattern — event listener or observer — is appropriate and why?

---

## Event Listeners: Synchronous Dispatch

```js
button.addEventListener('click', (event) => {
  // This runs synchronously during the click event dispatch
  // event.preventDefault() must be called synchronously here
  event.preventDefault();
  handleSubmit();
});
```

Event listeners are synchronous for a reason: user interaction requires immediate feedback. `preventDefault()` must be called before the browser's default action happens — which is synchronous. You can't defer it to a microtask.

The DOM event model gives you:
- `capture` phase (top-down through ancestors)
- `target` phase (on the element)
- `bubble` phase (bottom-up through ancestors)
- `stopPropagation()` to short-circuit the chain
- `once: true` for one-shot listeners

---

## Observer APIs: Asynchronous Batched Notification

```js
const observer = new IntersectionObserver((entries) => {
  // Runs asynchronously after paint — never mid-scroll
  // entries is an array — multiple elements can change at once
  entries.forEach(entry => {
    if (entry.isIntersecting) lazyLoad(entry.target);
  });
});
```

Observers are asynchronous because:
1. Browser needs to measure state across the full layout/compositing pipeline
2. Batching multiple changes into one callback is vastly more efficient
3. Measurements should be delivered after the browser's own state is settled — not in the middle of layout

---

## Choosing Between Them

### Use event listeners for:

**User interactions**
```js
button.addEventListener('click', handler);
input.addEventListener('input', handleInput);
form.addEventListener('submit', handleSubmit);
```

**Browser lifecycle events**
```js
window.addEventListener('load', onPageLoad);
document.addEventListener('DOMContentLoaded', onDOMReady);
window.addEventListener('beforeunload', onBeforeUnload);
```

**Custom application events**
```js
document.addEventListener('cart:updated', syncCartUI);
element.dispatchEvent(new CustomEvent('item-added', { bubbles: true }));
```

**Media/device events**
```js
window.addEventListener('online', syncPendingQueue);
window.addEventListener('offline', showOfflineBanner);
mediaQueryList.addEventListener('change', updateLayout);
```

### Use observers for:

**Element visibility** → IntersectionObserver
```js
// When to lazy-load, trigger animation, track impression
```

**Element size changes** → ResizeObserver
```js
// Component-aware responsive layout, canvas resolution
```

**DOM tree mutations** → MutationObserver
```js
// Reacting to third-party DOM changes, light DOM from shadow root
```

**Performance metrics** → PerformanceObserver
```js
// LCP, CLS, INP, long tasks, resource timing for RUM
```

---

## The Old Anti-Patterns (And Their Observer Replacements)

### Anti-pattern 1: Scroll + getBoundingClientRect

```js
// Old way
window.addEventListener('scroll', () => {
  elements.forEach(el => {
    const rect = el.getBoundingClientRect();  // forced layout per element, per scroll event
    if (rect.top < window.innerHeight) doSomething(el);
  });
});

// Correct way
const io = new IntersectionObserver(entries => {
  entries.filter(e => e.isIntersecting).forEach(e => doSomething(e.target));
});
elements.forEach(el => io.observe(el));
```

### Anti-pattern 2: Window resize for element layout

```js
// Old way
window.addEventListener('resize', debounce(() => {
  const width = container.getBoundingClientRect().width; // forced layout
  if (width < 600) setCompact(true);
}, 100));

// Correct way
const ro = new ResizeObserver(([entry]) => {
  setCompact(entry.contentRect.width < 600);
});
ro.observe(container);
```

### Anti-pattern 3: Polling for DOM changes

```js
// Old way — polling every 100ms
const interval = setInterval(() => {
  if (document.querySelector('.injected-element')) {
    clearInterval(interval);
    handleElement();
  }
}, 100);

// Correct way
const mo = new MutationObserver(() => {
  const el = document.querySelector('.injected-element');
  if (el) {
    mo.disconnect();
    handleElement();
  }
});
mo.observe(document.body, { childList: true, subtree: true });
```

---

## Combining Both Patterns

Many real-world cases use both:

```js
// IntersectionObserver detects visibility (browser measurement)
const io = new IntersectionObserver(([entry]) => {
  if (entry.isIntersecting) {
    videoEl.play(); // DOM manipulation — no event involved
  } else {
    videoEl.pause();
  }
});
io.observe(videoEl);

// Event listener handles user interaction
videoEl.addEventListener('click', () => {
  videoEl.paused ? videoEl.play() : videoEl.pause();
  io.disconnect(); // user took over — stop auto-play logic
});
```

---

## Performance Trade-off Summary

| Scenario | Wrong tool | Cost | Right tool |
|---|---|---|---|
| Lazy load on scroll | `scroll` + `getBoundingClientRect` | Layout thrashing every frame | `IntersectionObserver` |
| Component responsive layout | `window.resize` | Misses non-window size changes | `ResizeObserver` |
| Watch third-party DOM | Polling with `setInterval` | CPU burn, latency | `MutationObserver` |
| Measure LCP | `window.load` + `getEntriesByType` | Misses progressive LCP updates | `PerformanceObserver` |
| Handle button click | None | Event listener is the only option | `addEventListener` |
| React to WebSocket message | None | Event or callback from WS | `addEventListener` on WS |

---

## Gotchas

**Observers aren't real-time.** If your code needs to react *synchronously* to a change (e.g., `preventDefault` during an event), an observer cannot do that. Observers are always asynchronous.

**Event listeners don't tell you the current state.** `addEventListener('resize', fn)` doesn't fire immediately — you must read the current state separately. Observers deliver the current state on `observe()` (initial callback).

**Both need cleanup.** Forgetting `removeEventListener` leaks closures; forgetting `observer.disconnect()` leaks observation targets and keeps callbacks alive.

**Debouncing event listeners doesn't make them observers.** A debounced `scroll` listener still runs on the main thread, still requires `getBoundingClientRect()`, still blocks paint. Debouncing reduces frequency; IntersectionObserver eliminates the problem.

---

## Interview Questions

**Q (High): When should you choose IntersectionObserver over a scroll event listener, and why is one better than the other for element visibility?**

Answer: Always prefer IntersectionObserver for element visibility detection. A `scroll` event listener has two fundamental problems. First, it fires on every pixel of scroll movement — potentially 60 times per second — regardless of whether any element has changed visibility. With IntersectionObserver, the callback fires only when an element actually crosses a threshold. Second, inside a scroll handler you must call `getBoundingClientRect()` to determine element position, which forces a synchronous layout calculation. On a page with many observed elements, this is called dozens of times per scroll frame, creating layout thrashing that causes visible scroll jank. IntersectionObserver computes visibility as part of the browser's compositing pipeline — no forced layout, delivered asynchronously after paint, batched across all observed elements.

**Q (High): A senior engineer says "we should use MutationObserver instead of polling to detect when a third-party script injects an element." Explain why MutationObserver is better and describe the implementation.**

Answer: Polling with `setInterval` burns CPU every N milliseconds regardless of whether anything has changed — it's a busy-wait. It also introduces latency: if the script interval is 100ms and the element is injected 1ms after a check, detection takes up to 99ms. MutationObserver delivers a notification within one microtask of the DOM change — essentially zero latency. Implementation: create a `MutationObserver` with `{ childList: true, subtree: true }` on a high-level ancestor (often `document.body`). In the callback, check each `record.addedNodes` for the target element or use `document.querySelector()`. Call `observer.disconnect()` after detection to stop watching. This is event-driven: zero CPU cost while waiting, near-zero latency on detection, and correct cleanup.

**Q (Medium): What is the key difference between when event listener callbacks fire and when observer callbacks fire?**

Answer: Event listener callbacks fire synchronously — during the event dispatch, before control returns to the code that triggered the event or before the browser's default action. This synchronous guarantee is what makes `preventDefault()` work. Observer callbacks fire asynchronously: MutationObserver callbacks are microtasks (fire after the current task, before paint); IntersectionObserver and ResizeObserver callbacks fire after the browser's layout and paint phase — they can't be called mid-layout because they require the layout to be complete to measure. This asynchronous delivery is what makes observers safe from layout thrashing: they deliver measurements *after* the browser's work is done, not in the middle of it.

**Q (Low): If you debounce a `resize` event listener, does it become equivalent to ResizeObserver? Why or why not?**

Answer: No. Debouncing reduces the *frequency* of the handler calls, but doesn't fix the fundamental problems. A debounced `resize` handler still only fires when the window resizes — it doesn't observe element-level size changes. If an element's size changes because a sibling was removed from a flex container, a debounced `resize` listener fires nothing. Inside the debounced handler, you still need `getBoundingClientRect()` or `offsetWidth` to read the element's size, which still forces synchronous layout. Debouncing just reduces how often that layout force happens, not eliminates it. ResizeObserver eliminates the polling, delivers the new size in the callback entry (no forced layout needed), and responds to any cause of size change — not just window resize.

---

## Self-Assessment

- [ ] List three use cases that belong to event listeners and three that belong to observer APIs
- [ ] Explain why `scroll` + `getBoundingClientRect` causes layout thrashing and how IntersectionObserver avoids it
- [ ] Describe when observer callbacks fire relative to the browser rendering pipeline
- [ ] Explain why debouncing a resize handler doesn't make it equivalent to ResizeObserver
- [ ] Write a pattern that uses both an observer and an event listener together for auto-play video

---
*Next: Phase 13 — Security — XSS, CSRF, CSP, CORS, authentication patterns, SRI, clickjacking, and Trusted Types.*
